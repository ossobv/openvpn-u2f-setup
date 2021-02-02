openvpn-u2f-setup
=================

Configuration and howto to use a *U2F device (YubiKey)* as time based second
authentication factor for *OpenVPN* logins.

Components
----------

* OpenVPN client:

  - openvpn daemon, with sane certificates already set up;

  - a *YubiKey* or other U2F/FIDO2 device;

  - ``u2f-host`` CLI application to sign a challenge based on the
    current timestamp;

  - a `SystemD ask-password agent [ext]
    <https://systemd.io/PASSWORD_AGENTS/>`_ to pick up
    ``auth-user-pass`` requests from *OpenVPN*: ``openvpn-u2f-ask-password``

* OpenVPN server:

  - openvpn daemon, with sane certificates already set up;

  - ``u2f-server`` CLI application;

  - an ``auth-pass-verify`` script that receives the *U2F* key handle
    *as username* and the challenge and response *as password*:
    `openvpn-verify <./openvpn-verify>`_

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

    # Use via-file because we'd have to set --script-security 3 for via-env:
    auth-user-pass-verify /etc/openvpn/openvpn-u2f-setup/openvpn-verify via-file

Further, you'll need to create a ``keyhandle.dat`` and ``userkey.dat``
and place them in ``/etc/openvpn/u2f/<CN>/`` where ``<CN>`` is the
certificate *commonName*. See CREATING HANDLES below.

And ``/etc/openvpn/openvpn-u2f-setup/openvpn-verify`` needs to work. It
will mainly require you to have ``u2f-server(1)`` installed.


OpenVPN client setup
--------------------

client.conf:

::

    # When OpenVPN is tied to SystemD, this will trigger ask-password support.
    # You'll need to have the openvpn-u2f-ask-password daemon available to
    # notify you that U2F authentication is needed and interact with it.
    auth-user-pass


CREATING HANDLES
----------------

On your laptop/desktop, ``u2f-host(1)`` needs to be installed. It will
handle the communication with the *U2F device (Yubikey)* through the
``openvpn-u2f-ask-password`` helper.

When configuring the *U2F* support, you will need to run a registration
step, preferably directly on the *OpenVPN* server:

.. code-block:: console

    # CN=yourCommonName && mkdir -p /etc/openvpn/u2f/$CN
    # ORIGIN=pam://myorigin && APPID=myappid
    # umask 0077

.. code-block:: console

    # u2f-server -a register -o $ORIGIN -i $APPID \
        -k /etc/openvpn/u2f/$CN/keyhandle.dat \
        -p /etc/openvpn/u2f/$CN/userkey.dat
    { "challenge": "nO72...", "version": "U2F_V2", "appId": "myappid" }

Feed this challenge to ``u2f-host``:

.. code-block:: console

    $ ORIGIN=pam://myorigin

.. code-block:: console

    $ u2f-host -a register -o $ORIGIN <<EOF
    { "challenge": "nO72...", "version": "U2F_V2", "appId": "myappid" }
    EOF

Now touch the *U2F* device. The ``u2f-host`` will output something like this:

.. code-block:: data

    { "registrationData": "BQS...", "clientData": "eyAiY..." }

Feed the ``registrationData`` back to the ``u2f-server``, and end
*stdin* with a ^D (control-D).

It will say ``Registration successful`` and you should now have two files:

.. code-block:: console

    # ls /etc/openvpn/u2f/$CN
    -rw------- 1 root root 86 jan 29 17:47 keyhandle.dat
    -rw------- 1 root root 65 jan 29 17:47 userkey.dat

.. code-block:: console

    # cat /etc/openvpn/u2f/$CN/keyhandle.dat
    b6Ac2BI...

You'll need this keyhandle on the client side as well. See below.


Configuring the ask-password helper
-----------------------------------

* Install ``openvpn-u2f-ask-password`` in ``/usr/local/bin``.

* Copy your personal ``keyhandle.dat`` from the server to
  ``/etc/openvpn/client/VPN_NAME/keyhandle.dat`` when ``VPN_NAME.conf``
  holds your VPN config.

* Ensure that your have all dependencies (``python3-pyinotify`` and
  optionally ``python3-gi`` for *GNOME* notification integration).

* Configure so it auto-starts, using *SystemD* (see
  ``openvpn-u2f-ask-password.service``).


Running
-------

If everything is properly configured, a restart of your VPN connection
should trigger a blinking light on your *U2F device (YubiKey)*. Touch it
to log in.

Or don't touch it, and confirm that you cannot log in.

While testing, you can start ``openvpn-u2f-ask-password`` from the
command line (as root) to get a better feel of what's going on.


BUGS/TODO
=========

* Right now, the ``openvpn-u2f-ask-password`` handler does not detect
  when ask-password requests go stale / are removed. So if you don't
  insert a key, you will not get rid of the "please insert u2f"
  notification.

* Document why you'd want to be root. And what you need to not be root.
