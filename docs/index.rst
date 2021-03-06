.. rst-class:: hide-header

************************************
pyngrok - a Python wrapper for ngrok
************************************

.. image:: _html/logo.png
   :alt: pyngrok - a Python wrapper for ngrok
   :align: center

.. image:: https://badge.fury.io/py/pyngrok.svg
   :target: https://badge.fury.io/py/pyngrok
.. image:: https://travis-ci.org/alexdlaird/pyngrok.svg?branch=master
   :target: https://travis-ci.org/alexdlaird/pyngrok
.. image:: https://codecov.io/gh/alexdlaird/pyngrok/branch/master/graph/badge.svg
   :target: https://codecov.io/gh/alexdlaird/pyngrok
.. image:: https://readthedocs.org/projects/pyngrok/badge/?version=latest
   :target: https://pyngrok.readthedocs.io/en/latest/?badge=latest
.. image:: https://img.shields.io/pypi/pyversions/pyngrok.svg
   :target: https://pypi.org/project/pyngrok/
.. image:: https://img.shields.io/pypi/l/pyngrok.svg
   :target: https://pypi.org/project/pyngrok/
.. image:: https://img.shields.io/twitter/url/http/shields.io.svg?style=social
   :target: https://twitter.com/intent/tweet?text=Check+out+%23pyngrok%2C+a+Python+wrapper+for+%23ngrok+that+lets+you+programmatically+open+secure+%23tunnels+to+local+web+servers%2C+build+%23webhook+integrations%2C+enable+SSH+access%2C+test+chatbots%2C+demo+from+your+own+machine%2C+and+more.%0D%0A%0D%0A&url=https://github.com/alexdlaird/pyngrok&via=alexdlaird

``pyngrok`` is a Python wrapper for ``ngrok`` that manages its own binary and puts it on your path,
making ``ngrok`` readily available from anywhere on the command line and via a convenient Python API.

`ngrok <https://ngrok.com>`_ is a reverse proxy tool that opens secure tunnels from public URLs to localhost, perfect
for exposing local web servers, building webhook integrations, enabling SSH access, testing chatbots, demoing from
your own machine, and more, and its made even more powerful with native Python integration through ``pyngrok``.

Installation
============

``pyngrok`` is available on `PyPI <https://pypi.org/project/pyngrok/>`_ and can be installed
using ``pip``:

.. code-block:: sh

    pip install pyngrok

or ``conda``:

.. code-block:: sh

    conda install -c conda-forge pyngrok

That's it! ``pyngrok`` is now available `as a package to our Python projects <#open-a-tunnel>`_,
and ``ngrok`` is now available `from the command line <#command-line-usage>`_.

4.1.x to 5.0.0
==============

The next release, 5.0.0, contains breaking changes, including dropping support for Python 2.7. 4.2.x is meant to ease
migration between 4.1.x and 5.0.0 and should not be pinned, as it will not be supported after 5.0.0 is released. To
prepare for these breaking changes, `see the changelog <https://github.com/alexdlaird/pyngrok/blob/master/CHANGELOG.md#420---2020-10-11>`_
To avoid these breaking changes altogether, or if Python 2.7 support is still needed, pin ``pyngrok>=4.1,<4.2``.

Open a Tunnel
=============

To open a tunnel, use the :func:`~pyngrok.ngrok.connect` method, which returns the public URL generated by
``ngrok``.

.. code-block:: python

    from pyngrok import ngrok

    # Open a HTTP tunnel on the default port 80
    public_url = ngrok.connect()
    # Open a SSH tunnel
    ssh_url = ngrok.connect(22, "tcp")

The :func:`~pyngrok.ngrok.connect` method takes an optional ``options`` parameter, which allows us to pass
additional properties that are `supported by ngrok <https://ngrok.com/docs#tunnel-definitions>`_,
`as shown below <#passing-options>`__.

Get Active Tunnels
==================

It can be useful to ask the ``ngrok`` client what tunnels are currently open. This can be
accomplished with the :func:`~pyngrok.ngrok.get_tunnels` method, which returns a list of
:class:`~pyngrok.ngrok.NgrokTunnel` objects.

.. code-block:: python

    from pyngrok import ngrok

    tunnels = ngrok.get_tunnels()
    # A public ngrok URL that tunnels to port 80
    # (ex. http://<public_sub>.ngrok.io)
    public_url = tunnels[0].public_url

Close a Tunnel
==============

All open tunnels will automatically be closed when the Python process terminates, but we can
also close them manually with :func:`~pyngrok.ngrok.disconnect`.

.. code-block:: python

    from pyngrok import ngrok

    public_url = "http://<public_sub>.ngrok.io"

    ngrok.disconnect(public_url)

The ``ngrok`` Process
=====================

Opening a tunnel will start the ``ngrok`` process. This process will remain alive, and the tunnels
open, until :func:`~pyngrok.ngrok.kill()` is invoked, or until the Python process terminates.

If we are building a short-lived app, for instance a CLI, we may want to block on the ``ngrok``
process so tunnels stay open until the user intervenes. We can do that by accessing the
:class:`~pyngrok.process.NgrokProcess`.

.. code-block:: python

    from pyngrok import ngrok

    ngrok_process = ngrok.get_ngrok_process()

    try:
        # Block until CTRL-C or some other terminating event
        ngrok_process.proc.wait()
    except KeyboardInterrupt:
        print(" Shutting down server.")

        ngrok.kill()

The :class:`~pyngrok.process.NgrokProcess` contains an ``api_url`` variable, usually initialized to
http://127.0.0.1:4040, from which we can access the `ngrok client API <https://ngrok.com/docs#client-api>`_.

.. note::

    If some feature we need is not available in this package, the client API is accessible to us via the
    :func:`~pyngrok.ngrok.api_request` method. Additionally, the :class:`~pyngrok.ngrok.NgrokTunnel` objects expose a
    ``uri`` variable, which contains the relative path used to manipulate that resource against the client API.

    This package also gives us access to ``ngrok`` from the command line, `as shown below <#command-line-usage>`__.

Event Logs
==========

When ``ngrok`` emits logs, ``pyngrok`` can surface them to a callback function. To register this
callback, use :class:`~pyngrok.conf.PyngrokConfig` and pass the function as ``log_event_callback``. Each time a
log is processed, this function will be called, passing a :class:`~pyngrok.process.NgrokLog` as its only parameter.

.. code-block:: python

    from pyngrok.conf import PyngrokConfig
    from pyngrok import ngrok

    def log_event_callback(log):
        print(str(log))

    pyngrok_config = PyngrokConfig(log_event_callback=log_event_callback)

    ngrok.connect(pyngrok_config=pyngrok_config)

If these events aren't necessary for our use case, some resources can be freed up by turning them off.

Either use :class:`~pyngrok.conf.PyngrokConfig` to not start the thread in the first place:

.. code-block:: python

    from pyngrok.conf import PyngrokConfig
    from pyngrok import ngrok

    pyngrok_config = PyngrokConfig(monitor_thread=False)

    ngrok.connect(pyngrok_config=pyngrok_config)

or call the :func:`~pyngrok.process.NgrokProcess.stop_monitor_thread` method when we're done using it:

.. code-block:: python

    import time

    from pyngrok import ngrok

    ngrok.connect()
    time.sleep(1)
    ngrok.get_ngrok_process().stop_monitor_thread()

Expose Other Services
=====================

Using ``ngrok`` we can expose any number of non-HTTP services, for instances databases, game servers, etc. This
can be accomplished by using ``pyngrok`` to open a ``tcp`` tunnel to the desired service.

.. code-block:: python

    from pyngrok import ngrok

    # Open a tunnel to MySQL with a Reserved TCP Address
    ngrok.connect(3306, "tcp",
                  options={"remote_addr": "1.tcp.ngrok.io:12345"})


We can also serve up local directories via `ngrok's built-in fileserver <https://ngrok.com/docs#http-file-urls>`_.

.. code-block:: python

    from pyngrok import ngrok

    # Open a tunnel to a local file server
    ngrok.connect("file:///")


Configuration
=============

``PyngrokConfig``
-----------------

``pyngrok``'s interactions with the ``ngrok`` binary (and other things) can be configured using
:class:`~pyngrok.conf.PyngrokConfig`. Most methods accept ``pyngrok_config`` as a keyword argument, and
:class:`~pyngrok.process.NgrokProcess` will maintain a reference to its own :class:`~pyngrok.conf.PyngrokConfig` once a
process has been started. If ``pyngrok_config`` is not given, its documented defaults will be used.

The ``pyngrok_config`` argument is only used when the ``ngrok`` process is first started, which will only be
the first time most methods in the :mod:`~pyngrok.ngrok` module are called. You can check if a process is already or
still running by calling its :func:`~pyngrok.process.NgrokProcess.healthy` method.

.. note::

    If ``ngrok`` is not already installed at the ``ngrok_path`` in :class:`~pyngrok.conf.PyngrokConfig`, it
    will be installed the first time most methods in the :mod:`~pyngrok.ngrok` module are called.

    If we need to customize the installation of ``ngrok``, perhaps specifying a timeout, proxy, use a custom mirror
    for the download, etc. we can do so by leveraging the :mod:`~pyngrok.installer` module. Keyword arguments in this
    module are ultimately passed down to :py:func:`urllib.request.urlopen`, so as long as we use the
    :mod:`~pyngrok.installer` module ourselves prior to invoking any :mod:`~pyngrok.ngrok` methods, we can can control
    how ``ngrok`` is installed and from where.

Setting the ``authtoken``
-------------------------

Running ``ngrok`` with an auth token enables additional features available on our account (for
instance, the ability to open multiple tunnels concurrently). We can obtain our auth token from
the `ngrok dashboard <https://dashboard.ngrok.com>`_ and install it to ``ngrok``'s config file like this:

.. code-block:: python

    from pyngrok import ngrok

    ngrok.set_auth_token("<NGROK_AUTH_TOKEN>")

    # Once an auth token is set, we are able to open
    # multiple tunnels at the same time
    ngrok.connect()
    ngrok.connect(8000)

We can also override ``ngrok``'s installed auth token using :class:`~pyngrok.conf.PyngrokConfig`:

.. code-block:: python

    from pyngrok.conf import PyngrokConfig
    from pyngrok import ngrok

    pyngrok_config = PyngrokConfig(auth_token="<NGROK_AUTH_TOKEN>")

    ngrok.connect(pyngrok_config=pyngrok_config)

Setting the ``region``
----------------------

By default, ``ngrok`` will open a tunnel in the ``us`` region. To override this, use
the ``region`` parameter in :class:`~pyngrok.conf.PyngrokConfig`:

.. code-block:: python

    from pyngrok.conf import PyngrokConfig
    from pyngrok import ngrok

    pyngrok_config = PyngrokConfig(region="au")

    url = ngrok.connect(pyngrok_config=pyngrok_config)

Passing ``options``
-------------------

It is possible to configure the tunnel when it is created, for instance adding authentication,
a subdomain, or other tunnel properties `supported by ngrok <https://ngrok.com/docs#tunnel-definitions>`_.
These can be passed to the tunnel with the ``options`` parameter.

Here is an example starting ``ngrok`` in Australia, then opening a tunnel with subdomain
``foo`` that requires basic authentication for requests.

.. code-block:: python

    from pyngrok.conf import PyngrokConfig
    from pyngrok import ngrok

    options = {"subdomain": "foo", "auth": "username:password"}
    pyngrok_config = PyngrokConfig(region="au")

    url = ngrok.connect(options=options,
                        pyngrok_config=pyngrok_config)

Config File
-----------

By default, `ngrok will look for its config file <https://ngrok.com/docs#config>`_ in the home directory's ``.ngrok2``
folder. We can override this behavior in one of two ways.

Either use :class:`~pyngrok.conf.PyngrokConfig`:

.. code-block:: python

    from pyngrok.conf import PyngrokConfig
    from pyngrok import ngrok

    pyngrok_config = PyngrokConfig(config_path="/opt/ngrok/config.yml")

    ngrok.get_tunnels(pyngrok_config=pyngrok_config)

or override it on the default:

.. code-block:: python

    from pyngrok import ngrok, conf

    conf.DEFAULT_PYNGROK_CONFIG.config_path = "/opt/ngrok/config.yml"

    ngrok.get_tunnels()

Binary Path
-----------

The ``pyngrok`` package manages its own ``ngrok`` binary. However, we can use our ``ngrok`` binary if we
want in one of two ways.

Either use :class:`~pyngrok.conf.PyngrokConfig`:

.. code-block:: python

    from pyngrok.conf import PyngrokConfig
    from pyngrok import ngrok

    pyngrok_config = PyngrokConfig(ngrok_path="/usr/local/bin/ngrok")

    ngrok.connect(pyngrok_config=pyngrok_config)

or override it on the default:

.. code-block:: python

    from pyngrok import ngrok, conf

    conf.DEFAULT_PYNGROK_CONFIG.ngrok_path = "/usr/local/bin/ngrok"

    ngrok.connect()

Command Line Usage
==================

This package puts the default ``ngrok`` binary on our path, so all features of ``ngrok`` are also
available on the command line.

.. code-block:: sh

    ngrok http 80

For details on how to fully leverage ``ngrok`` from the command line, see `ngrok's official documentation <https://ngrok.com/docs>`_.

Python 2.7
==========

The last version of ``pyngrok`` that supports Python 2.7 is 4.1.x, so we need to pin ``pyngrok>=4.1,<4.2`` if we still
want to use ``pyngrok`` with this version of Python.

Dive Deeper
===========

For more advanced usage, integration examples, and tips to troubleshoot common issues, dive deeper in to the rest of
the documentation.

.. toctree::
   :maxdepth: 2

   api
   integrations
   troubleshooting

.. include:: ../CONTRIBUTING.rst
