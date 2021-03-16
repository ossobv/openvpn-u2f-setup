openvpn-u2f-setup
=================

Configuration and howto to use a *U2F device (YubiKey)* as time based second
authentication factor for *OpenVPN* logins.

Components
----------

* OpenVPN server:

  - ``openvpn`` daemon, with an already sane configuration and proper
    certificates;

  - ``u2f-server`` command line tool to verify the challenge signature;

  - an ``auth-user-pass-verify`` script that receives the *U2F* key handle
    *as username* and the challenge and response *as password*:
    `<openvpn-u2f-verify>`_

* OpenVPN client:

  - ``openvpn`` daemon, with an already sane configuration and proper
    certificates;

  - a *YubiKey* or other *U2F/FIDO2* device;

  - ``u2f-host`` command line tool to sign a challenge based on the
    current timestamp;

  - a `SystemD ask-password agent
    <https://systemd.io/PASSWORD_AGENTS/>`_ (external link) to pick up
    ``auth-user-pass`` requests from *OpenVPN*:
    `<openvpn-u2f-ask-password>`_

    *(Do not forget to add this option to the client config. The OpenVPN
    client will not complain, but simply fail to set up the VPN. Even
    clients with explicitly disabled U2F need to provide something.)*

The ``u2f-host(1)`` and ``u2f-server(1)`` CLI applications are in charge
of the *U2F* heavy lifting. The *key handles*, challenges and
responses are sent to the *OpenVPN* server using the *username* and
*password* additional authentication.

**Personal certificates are your first line of defense. U2F is the second.**


Caveats
-------

**The challenge is not random.** But because it uses the unixtime which
should be the same everywhere, we use that to guard against replay attacks
(outside a short time window). This caveat is due to the fact that
*OpenVPN* does not support passing a challenge to the client. If it did,
we could pass the required *key handle* and supply a truly random
challenge.

*Note that using the timestamp is no less secure than the ubiqitous
time-based one-time passwords (TOTP) as provided by the Google
Authenticator and Authy apps.* And on all other fronts, using a hardware
*U2F* device with a public/private key is *more* secure because *there
is no shared secret* and *the private key cannot be extracted from the
hardware device.*


How does this compare to other solutions?
-----------------------------------------

* You can use the `old style Yubico OTP
  <https://developers.yubico.com/yubico-pam/YubiKey_and_OpenVPN_via_PAM.html>`_
  (external link) far more easily. It's drawback however is that unless
  you set up a *Yubico OTP* server locally, your service has to have
  access to the internet, relying on *their* service. That is not good
  enough if you rely on this access to *fix* internet connectivity
  problems.

* *SparkLabs* has some kind of `modified OpenVPN server + plugins
  <https://www.sparklabs.com/support/kb/article/yubikey-u2f-two-factor-authentication-with-openvpn-and-viscosity/>`_
  (external link) which requires (a) a patched *OpenVPN*, (b) an equally
  complicated setup and (c) it has no documentation on `how an OpenVPN
  client should present these U2F credentials
  <https://github.com/thesparklabs/openvpn-two-factor-extensions/blob/73166ce305260bf0baa4381f98330bb82c36447c/yubikey-u2f-pam-plugin/auth-pam-u2f.py#L66-L96>`_
  (source code) at all. (Their *Viscosity* product is proprietary and
  works only on Windows and Mac.)


How does this work?
-------------------

At connect time ``openvpn-u2f-setup`` calls ``u2f-host`` to sign the
current timestamp using the ``-a authenticate`` action.

This goes in::

    {"keyHandle": "<86-byte-b64encoded-handle>",
     "version": "U2F_V2",
     "challenge": "<43-byte-b64encoded-challenge>",
     "appId": "<APPID>"}

The client knows which handle it's supposed to use (from
``keyhandle.dat``). The challenge is constructed from the current
timestamp.

The *U2F device* signs the challenge, and responds with something like this::

    {"signatureData": "<102-byte-encoded-signed-respons>",
     "clientData": "<N-byte-64encoded-json-which-was-signed>",
     "keyHandle": "<86-byte-b64encoded-handle>"}

The ``openvpn-u2f-ask-password`` will pass these to the *OpenVPN* client
(which in turn passes them on to the server):

* ``username = KEYHANDLE`` (the 86 byte handle)

* ``password = TIMESTAMP '/' SIGNATURE`` (10 digit timestamp, 1 slash,
  and about 102 bytes of signature)

These both fit in the 256 bytes maximum of *OpenVPN*
``OPTION_PARM_SIZE``, so no changes to *OpenVPN* are needed.

On the server side, the same challenge is reconstructed from the known
*key handle* and *timestamp* and the signature is validated against the
(known) public key (``userkey.dat``).


OpenVPN server setup
--------------------

server.conf:

::

    # [OSSO B.V. openvpn-u2f-setup]
    # (Use via-file because we'd have to set --script-security 3 for via-env.)
    auth-user-pass-verify /etc/openvpn/openvpn-u2f-setup/openvpn-u2f-verify via-filez
    reneg-sec 28800  # 8 hours so we don't need to renegotiate U2F too often
    # Or, if you insist on long lived sessions with only a single interaction:
    #auth-gen-token  # an alternative to longer reneg-sec

Further, you'll need to create a ``keyhandle.dat`` and ``userkey.dat``
and place them in ``/etc/openvpn/u2f/<CN>/`` where ``<CN>`` is the
certificate *commonName*. See CREATING HANDLES below.

And ``/etc/openvpn/openvpn-u2f-setup/openvpn-u2f-verify`` needs to work. It
will mainly require you to have ``u2f-server(1)`` installed.


OpenVPN client setup
--------------------

client.conf:

::

    # [OSSO B.V. openvpn-u2f-setup]
    # When OpenVPN is tied to SystemD, this will trigger ask-password support.
    # You'll need to have the openvpn-u2f-ask-password daemon available to
    # notify you that U2F authentication is needed and interact with it.
    auth-user-pass
    auth-nocache        # caching the one time password is pointless
    auth-retry interact # force user action before reconnection attempt
    reneg-sec 0         # let the server decide when to renegotiate keys

For clients where *U2F* is explicitly disabled, you will still need dummy
credentials::

    # [OSSO B.V. openvpn-u2f-setup]
    # Dummy auth user/pass for VPN clients with U2F explicitly disabled.
    # (Any file with 2 or more lines will do.)
    auth-user-pass /etc/protocols
    auth-retry nointeract


CREATING HANDLES
----------------

On your laptop/desktop, ``u2f-host(1)`` needs to be installed. It will
handle the communication with the *U2F device (YubiKey)* through the
``openvpn-u2f-ask-password`` helper.

When configuring the *U2F* support, you will need to run a registration
step, preferably directly on the *OpenVPN* server:

.. code-block:: console

    # CN=yourCommonName && mkdir -p /etc/openvpn/u2f/$CN
    # ORIGIN=pam://openvpn-server && APPID=openvpn
    # umask 0077

.. code-block:: console

    # u2f-server -a register -o $ORIGIN -i $APPID \
        -k /etc/openvpn/u2f/$CN/keyhandle.dat \
        -p /etc/openvpn/u2f/$CN/userkey.dat
    { "challenge": "nO72...", "version": "U2F_V2", "appId": "openvpn" }

Feed this challenge (the entire JSON blob) to ``u2f-host``:

.. code-block:: console

    $ ORIGIN=pam://openvpn-server

.. code-block:: console

    $ u2f-host -a register -o $ORIGIN <<EOF
    { "challenge": "nO72...", "version": "U2F_V2", "appId": "openvpn" }
    EOF

Now touch the *U2F* device. The ``u2f-host`` will output something like this:

.. code-block:: data

    { "registrationData": "BQS...", "clientData": "eyAiY..." }

Feed the reponse (the entire JSON blob) back to the ``u2f-server``, and end
*stdin* with a ^D (control-D).

It will say ``Registration successful`` and you should now have two files:

.. code-block:: console

    # ls -l /etc/openvpn/u2f/$CN
    total 8
    -rw------- 1 root root 86 jan 29 17:47 keyhandle.dat
    -rw------- 1 root root 65 jan 29 17:47 userkey.dat

(For the curious: the `details of the registrationData layout
<https://fidoalliance.org/specs/u2f-specs-1.0-bt-nfc-id-amendment/fido-u2f-raw-message-formats.html#registration-response-message-success>`_
(external link) or `example registrationData extraction
<https://github.com/Yubico/python-u2flib-server/blob/b2053563d4cdd530f254a863e59af11235bfde8f/u2flib_server/model.py#L156-L164>`_
(source code).)

.. code-block:: console

    # cat /etc/openvpn/u2f/$CN/keyhandle.dat
    b6Ac2BI...

You'll need this *key handle* on the client side as well. See below.

**NOTE: If your openvpn server runs as the openvpn user, make sure the
key files on the server are readable by the auth-user-pass-verify
script:**

.. code-block:: console

    # chown -R openvpn: /etc/openvpn/u2f/$CN


Configuring the ask-password helper on the client
-------------------------------------------------

* Install `<openvpn-u2f-ask-password>`_ (or simply this repository) in
  ``/etc/openvpn/openvpn-u2f-setup/``.

* Copy your personal ``keyhandle.dat`` from the server to
  ``/etc/openvpn/client/VPN_NAME/keyhandle.dat`` when ``VPN_NAME.conf``
  holds your VPN config. If you run the
  `<openvpn-u2f-ask-password.service>`_ as a user you can use ``~/.u2f_keys``.

* Ensure that your have all dependencies (``python3-pyinotify`` and
  optionally ``python3-gi`` for *GNOME* notification integration).

* Configure so it auto-starts, using *SystemD* (see
  `<openvpn-u2f-ask-password.service>`_).


Running
-------

If everything is properly configured, a restart of your VPN connection
should trigger a blinking light on your *U2F device (YubiKey)*. Touch it
to log in.

Or don't touch it, and confirm that you cannot log in.

While testing, you can start ``openvpn-u2f-ask-password`` from the
command line to get a better feel of what's going on.


F.A.Q.
======

* How do I know when to touch the *U2F device*?

  If you're using *GNOME* on *Ubuntu/Focal*, it should look somewhat
  like this:

  .. image:: ./openvpn-u2f-ask-password.gif
    :alt: GUI notification on right side

  Right now, you probably need to have ``notify-osd`` installed.

* When doing a ``openvpn`` restart, I get a ``Enter Auth Username:`` shown.

  This is an unfortunate effect of having multiple ``systemd-ask-password``
  agents. You can ignore the console version when
  ``openvpn-u2f-ask-password`` works as intended. The console agent will
  automatically close/abort once our agent has provided the credentials.

* ``openvpn-u2f-ask-password`` reports: ``error (-6): authenticator error``

  This generally means one of two things:

  - the *wrong U2F device* is inserted, or

  - the *key handle* was generated for a different APPID
    (did you change it after performing the registration step?)

* ``u2f-host`` claims my *YubiKey* is not an *U2F* device:

  .. code-block:: console

    $ u2f-host -a register -o pam://openvpn-server
    { "challenge": "VIrN...", "version": "U2F_V2", "appId": "openvpn" }
    ^D
    error: u2fh_devs_discover (-5): cannot find U2F device

  This is might be because it is not enabled. See ``ykman(1)`` (from the
  *yubikey-manager* package):

  .. code-block:: console

    $ ykman mode
    Current connection mode is: OTP+CCID
    Supported USB interfaces are: OTP, FIDO, CCID

  .. code-block:: console

    $ ykman mode OTP+FIDO+CCID
    Set mode of YubiKey to OTP+FIDO+CCID? [y/N]: y
    Mode set! You must remove and re-insert your YubiKey for this change
    to take effect.


BUGS/TODO
=========

* See if we want to use the server-side openvpn management interface of
  ``--management`` through which we can send a ``client-deny`` command
  which takes a ``client_reason`` through which we could push a
  challenge... see: ./doc/management-notes.txt in the openvpn tree.

* Fix better sane /u2f/ keyhandle.dat paths:

  - for less confusion (difference between client and server)

  - for multiple key handles for a single $CN

* Improve openvpn-u2f-ask-password usage for non-root users (@Urth?).

* We may want to add some wrapper scripts to make life
  managing/registering keys and handles easier. (We can manage quite a
  bit by decoding the ``registrationData`` ourself.)

* When everything is well-tested and works, we'll need to swap the
  default of ignoring users without ``keyhandle.dat`` to prohibit. We'll
  probably still want some way to allow certain certificates to connect
  without *U2F* though. (For systems where there is no human interaction.)

* Document why you'd want to be root. And what you need to not be root.
  (umask? Or fix key-read permissions to the openvpn-user?)

* For systems where systemd-ask-password does not work well, we could
  see if we can abuse auth-user-pass to read from a pseudofile (pipe?
  special filesystem?). Benefit: more granular control. Drawback: more
  complicated.

* If we wanted, we could use a side-channel (https?) to request a
  challenge during authentication. If set up properly, this could be
  more secure (because of the random challenge), but it does complicate
  the setup. (*SparkLabs Viscosity* appears to use custom *OpenVPN*
  `client reject reason
  <https://github.com/thesparklabs/openvpn-two-factor-extensions/blob/73166ce305260bf0baa4381f98330bb82c36447c/yubikey-u2f-pam-plugin/auth-pam-u2f.c#L487-L497>`_
  for this purpose.)

* Note that ``auth-token`` is not something we could use to pass a
  challenge to the connecting client. This is a token that is used for
  *renegotiation*. See ``auth-gen-token`` in the manual.
