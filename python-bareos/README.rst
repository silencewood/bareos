python-bareos
=============

`python-bareos` is a Python module to access a http://www.bareos.org backup system.

Packages for `python-bareos` are included in the Bareos core distribution and available via https://pypi.org/.


.. note::

   By default, the Bareos Director (>= 18.2.4) uses TLS-PSK when communicating through the network.

   Unfortenatly the Python core module ``ssl``
   does not support TLS-PSK.
   For testing ``python-bareos`` should be used without TLS.
   The section `Transport Encryption (TLS-PSK)`_ describes
   how to use ``python-bareos`` with TLS-PSK
   and about the limitations.


Preparations
============

Best create a named console for testing.

.. code-block:: shell-session

   bconsole

   configure add console name=testuser password=secret TlsEnable=no profile=operator


Console user with name `testuser` and the profile `operator`.
The `operator` profile is a default profile that comes with the Bareos Director.
It does allow most commands. It only deny some dangerous commands
so it is well suited for this purpose.
(see ``show profile=operator``).
Futhermore, TLS enforcement is disabled for this console user.

TODO: TlsEnable=no not required.

Examples
========

Calling bareos-director user agent commands
-------------------------------------------

.. code:: python

   >>> import bareos.bsock
   >>> directorconsole = bareos.bsock.DirectorConsole(address='localhost', port=9101, password='secret')
   >>> print(directorconsole.call('help').decode("utf-8"))

To connect to a named console instead, use the `name` parameter:

.. code:: python

   >>> directorconsole=bareos.bsock.DirectorConsole(address='localhost', port=9101, name='user1', password='secret')

The result of the call method is a ``bytes`` object. In most cases, it has to be decoded to UTF-8.



Simple version of the bconsole in Python
----------------------------------------

.. code:: python

   >>> import bareos.bsock
   >>> directorconsole = bareos.bsock.DirectorConsole(address='localhost', port=9101, password='secret')
   >>> directorconsole.interactive()

Or use the included bconsole.py script:

.. code-block:: shell-session

   bconsole.py --debug --name=user1 --password=secret localhost


Use JSON objects of the API mode 2
----------------------------------

Requires: Bareos >= 15.2

For general information about API mode 2 and what data structures to expect,
see https://docs.bareos.org/master/DeveloperGuide/api.html#api-mode-2-json

Example:

.. code:: python

   >>> import bareos.bsock
   >>> directorconsole = bareos.bsock.DirectorConsoleJson(address='localhost', port=9101, password='secret')
   >>> pools = directorconsole.call('list pools')
   >>> for pool in pools["pools"]:
   ...   print(pool["name"])
   ...
   Scratch
   Incremental
   Full
   Differential


The results the the call method is a ``dict`` object.

In case of an error, an exception, derived from ``bareos.exceptions.Error`` is raised.

Example:


.. code:: python

   >>> result = directorconsole.call("test it")
   Traceback (most recent call last):
   ...
   bareos.exceptions.JsonRpcErrorReceivedException: failed: test it: is an invalid command.



.. _section-python-bareos-tls-psk:

Transport Encryption (TLS-PSK)
==============================

Since Bareos >= 18.2.4, Bareos supports TLS-PSK (Transport-Layer-Security Pre-Shared-Key) to secure its network connections and uses this by default.

sslpsk
------

Unfortenatly, the Python core module `ssl` does not support TLS-PSK.
There is limited support by the extra module `sslpsk` (see https://github.com/drbild/sslpsk).
At the time of writing, the lasted version installable via pip is 1.0.0 (https://pypi.org/project/sslpsk/), which is not working with Python > 3.

If python-bareos should use TLS-PSK with Python > 3, the latest version from https://github.com/drbild/sslpsk must by installed manually.

.. code:: shell

   git clone https://github.com/drbild/sslpsk.git
   cd sslpsk
   python setup.py build
   python setup.py install

`python-bareos` will detect, that `sslpsk` is available and will use it automatically.


.. code:: python

   >>> import bareos.bsock
   >>> bareos.bsock.DirectorConsole.is_tls_psk_available()
   True

Another limitation of the current `sslpsk` version is,
that it is not able to autodetect the TLS protocol version to use.

In order to use it:

.. code:: python

   >>> import ssl
   >>> import bareos.bsock
   >>> directorconsole = bareos.bsock.DirectorConsoleJson(address='localhost', user='testuser', password='secret', tls_version=ssl.PROTOCOL_TLSv1_2)

bareos.bsock.DirectorConsoleJson(address='localhost', name="admin-tls", password="secret", port=42311, tls_psk_require=True)

Failed to connect via TLS-PSK. Trying plain connection.
socket error: Conversation terminated (-4)
Failed to connect using protocol version 2. Trying protocol version 1. 

bareos.exceptions.AuthenticationError: failed (in response)

d = bareos.bsock.DirectorConsoleJson(address='localhost', name="admin-tls", password="secret", port=42311, tls_version=ssl.PROTOCOL_TLSv1_2, tls_psk_require=True)


tls-psk-require=True

Bareos, named console, different modes.
TlsEnable.

By default python-bareos

stunnel workaround
------------------

Bareos Password is stored as MD5. For stunnel, a byte sequence is required.

.. code:: shell

	 BAREOS_CONSOLE="admin"
	 BAREOS_PASSWORD="secret"

	 STUNNEL_PASSWORD=$(printf "$BAREOS_PASSWORD" | md5sum |  awk '{printf $1}' | hexdump -v -e '1/1 "%02X:"')
	 echo -e 'R_CONSOLE\x1e${BAREOS_USER}:${STUNNEL_PASSWORD}' > stunnel-bareos-psk-credential.conf

bareos_client.conf:

.. code:: cfg

   debug           = 7
   syslog          = no
   foreground = yes
   pid =
   [PSK client 1]
   client = yes
   accept = 127.0.0.1:19101
   connect = bareos.example.com:9101
   PSKsecrets = /tmp/stunnel/client_bareos_psk1.conf

  
stunnel-bareos-psk-credential.conf
  
.. code:: cfg
   
   R_CONSOLE^^admin:35:65:62:65:32:32:39:34:65:63:64:30:65:30:66:30:38:65:61:62:37:36:39:30:64:32:61:36:65:65:36:39
