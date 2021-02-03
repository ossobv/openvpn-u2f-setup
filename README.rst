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

  - a `SystemD ask-password agent (external link)
    <https://systemd.io/PASSWORD_AGENTS/>`_ to pick up
    ``auth-user-pass`` requests from *OpenVPN*:
    `<openvpn-u2f-ask-password>`_

    *(Do not forget to add this option to the client config. The OpenVPN
    client will not complain, but simply fail to set up the VPN. Even
    clients with explicitly disabled U2F need to provide something.)*

The ``u2f-host(1)`` and ``u2f-server(1)`` CLI applications are in charge
of the *U2F* heavy lifting. The *key handles*, challenges and
responses are sent to the *OpenVPN* server using the *username* and
*password* additional authentication.

**Personal certificates are your first line of defence. U2F is the second.**


Caveats
-------

**The challenge is not random.** But because it uses the unixtime which
should be the same everywhere, we use that to guard against replay attacks
(outside a short time window).

This caveat is due to the fact that *OpenVPN* does not support passing a
challenge to the client. If it did, we could pass the required keyhandle
and supply a truly random challenge.

*Note that using the timestamp is no less secure than the ubiqitous
time-based one-time passwords (TOTP) as provided by the Google
Authenticator and Authy apps.*

And on all other fronts, using a hardware *U2F* device with a
public/private key is *more* secure because *there is no shared secret*
and *the private key cannot be extracted from the hardware device.*


OpenVPN server setup
--------------------

server.conf:

::

    # [OSSO B.V. openvpn-u2f-setup]
    # (Use via-file because we'd have to set --script-security 3 for via-env.)
    auth-user-pass-verify /etc/openvpn/openvpn-u2f-setup/openvpn-u2f-verify via-file
    reneg-sec 28800  # 8 hours so we don't need to renegotiate U2F too often

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
    auth-retry interact
    reneg-sec 0  # let the server decide when to renegotiate keys

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

Feed this challenge to ``u2f-host``:

.. code-block:: console

    $ ORIGIN=pam://openvpn-server

.. code-block:: console

    $ u2f-host -a register -o $ORIGIN <<EOF
    { "challenge": "nO72...", "version": "U2F_V2", "appId": "openvpn" }
    EOF

Now touch the *U2F* device. The ``u2f-host`` will output something like this:

.. code-block:: data

    { "registrationData": "BQS...", "clientData": "eyAiY..." }

Feed the ``registrationData`` back to the ``u2f-server``, and end
*stdin* with a ^D (control-D).

It will say ``Registration successful`` and you should now have two files:

.. code-block:: console

    # ls -l /etc/openvpn/u2f/$CN
    total 8
    -rw------- 1 root root 86 jan 29 17:47 keyhandle.dat
    -rw------- 1 root root 65 jan 29 17:47 userkey.dat

.. code-block:: console

    # cat /etc/openvpn/u2f/$CN/keyhandle.dat
    b6Ac2BI...

You'll need this keyhandle on the client side as well. See below.


Configuring the ask-password helper
-----------------------------------

* Install `<openvpn-u2f-ask-password>`_ (or simply this repository) in
  ``/etc/openvpn/openvpn-u2f-setup/``.

* Copy your personal ``keyhandle.dat`` from the server to
  ``/etc/openvpn/client/VPN_NAME/keyhandle.dat`` when ``VPN_NAME.conf``
  holds your VPN config.

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
command line (as root) to get a better feel of what's going on.


F.A.Q.
======

* How do I know when to touch the *U2F device*?

  If you're using *GNOME* on *Ubuntu/Focal*, it should look somewhat
  like this:

  .. image:: ./openvpn-u2f-ask-password.gif
    :alt: GUI notification on right side

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

* When everything is well-tested and works, we'll need to swap the
  default of ignoring users without keyhandle.dat to prohibit. We'll
  probably still want some way to allow certain certificates to connect
  without *U2F* though. (For systems where there is no human interaction.)

* Document why you'd want to be root. And what you need to not be root.

* Check whether we can use ``auth-token`` and ``auth-gen-token`` stuff
  with a client-connect script; this might fix the passing of challenges
  and key handles...
