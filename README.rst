openvpn-u2f-setup
=================

Configuration and howto to use the YubiKey U2F as time based second
factor for OpenVPN logins.

Components
----------

* OpenVPN client:

  - openvpn daemon, with sane certificates already set up;

  - a *YubiKey* or other U2F/FIDO2 device;

  - ``u2f-host`` client application to sign a challenge based on the
    current timestamp;

  - a `SystemD ask-password agent
    <https://systemd.io/PASSWORD_AGENTS/>`_ to pick up
    ``auth-user-pass`` requests from OpenVPN: ``openvpn-u2f-ask-password``

* OpenVPN server:

  - openvpn daemon, with sane certificates already set up;

  - u2f-server (client application);

  - an `auth-pass-verify` script that receives the *YubiKey* key handle
    *as username* and the challenge and response *as password*:
    ``openvpn-verify``

The ``u2f-host(1)`` and ``u2f-server(1)`` CLI applications are in charge
of the U2F heavy lifting. *YubiKey* key handles, challenges and
responses are sent to the *OpenVPN* server using the *username* and
*password* additional authentication.

**Personal certificates are your first line of defence. U2F is the second.**


Caveats
-------

* The challenge is not random. But because it uses the unixtime which
  should be the same everywhere, we do guard against replay attacks
  (outside a short time window).

* Right now ``openvpn-u2f-ask-password`` (*SystemD ask-password* helper)
  does not know *who* is querying for credentials: this means that if
  you secure multiple VPNs with this method, *you must reuse a single
  YubiKey public key registration.*

These two caveats are both due to the fact that openvpn does not
support passing a challenge to the client. If it did, we could pass
the required keyhandle and supply a truly random challenge.


OpenVPN server setup
--------------------

server.conf:

::

    # Use via-file because we'd have to set --script-security 3 for via-env:
    auth-user-pass-verify /etc/openvpn/openvpn-u2f-setup/openvpn-verify via-file

Further, you'll need to create a ``keyhandle.dat`` and ``userkey.dat``
and place them in ``/etc/openvpn/u2f/<CN>/`` where <CN> is the
certificate *commonName*. See CREATING HANDLES below.

And ``/etc/openvpn/openvpn-u2f-setup/openvpn-verify`` needs to work. It
will mainly require you to have ``u2f-server(1)`` installed.


OpenVPN client setup
--------------------

client.conf:

::

    # When OpenVPN is tied to SystemD, this will trigger ask-password support.
    # You'll need to have the openvpn-u2f-ask-password daemon available to
    # notify you that YubiKey authentication is needed and interact with it.
    auth-user-pass


CREATING HANDLES
----------------

On your laptop/desktop, ``u2f-host(1)`` needs to be installed. It will
handle the communication with the *YubiKey* through the
``openvpn-u2f-ask-password`` helper.

When configuring the U2F support, you will need to run a registration
step, preferably directly on the *OpenVPN* server:

.. code-block:: console

    # CN=yourCommonName && mkdir -p /etc/openvpn/u2f/$CN
    # ORIGIN=pam://myorigin && APPID=myappid

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

Now touch the *YubiKey*. The ``u2f-host`` will output something like this:

.. code-block:: data

    { "registrationData": "BQS...", "clientData": "eyAiY..." }

Feed the ``registrationData`` back to the ``u2f-server``, and end
*stdin* with a ^D (control-D).

It will say ``Registration successful`` and you should now have two files:

.. code-block:: console

    # ls /etc/openvpn/u2f/$CN
    -rw-rw-r-- 1 root root 86 jan 29 17:47 keyhandle.dat
    -rw-rw-r-- 1 root root 65 jan 29 17:47 userkey.dat

.. code-block:: console

    # cat /etc/openvpn/u2f/$CN/keyhandle.dat
    b6Ac2BI...

You'll need to pass this keyhandle to ``openvpn-u2f-ask-password``.
Currently it is hardcoded at the top of the file.


Configuring the ask-password helper
-----------------------------------

* Install ``openvpn-u2f-ask-password`` in ``/usr/local/bin``.

* Edit it, and set ``KEYHANDLE`` at the top of the file.

* Ensure that your have all dependencies (``python3-pyinotify``).

* Configure so it auto-starts, using *SystemD* (see
  ``openvpn-u2f-ask-password.service``).


Running
-------

If everything is properly configured, a restart of your VPN connection
should trigger a blinking *YubiKey* light. Touch it to log in.

Or don't and confirm that you cannot log in.

While testing, you can start ``openvpn-u2f-ask-password`` from the
command line to get a better feel of what's going on.
