S3Fs
====

S3Fs is a Pythonic file interface to S3.  It builds on top of boto3_.

The top-level class ``S3FileSystem`` holds connection information and allows
typical file-system style operations like ``cp``, ``mv``, ``ls``, ``du``,
``glob``, etc., as well as put/get of local files to/from S3.

The connection can be anonymous - in which case only publicly-available,
read-only buckets are accessible - or via credentials explicitly supplied
or in configuration files.

Calling ``open()`` on a ``S3FileSystem`` (typically using a context manager)
provides an ``S3File`` for read or write access to a particular key. The object
emulates the standard ``File`` protocol (``read``, ``write``, ``tell``,
``seek``), such that functions expecting a file can access S3. Only binary read
and write modes are implemented, with blocked caching.

S3Fs uses and is based upon `fsspec`_.

.. _fsspec: https://filesystem-spec.readthedocs.io/en/latest/

Examples
--------

Simple locate and read a file:

.. code-block:: python

   >>> import s3fs
   >>> fs = s3fs.S3FileSystem(anon=True)
   >>> fs.ls('my-bucket')
   ['my-file.txt']
   >>> with fs.open('my-bucket/my-file.txt', 'rb') as f:
   ...     print(f.read())
   b'Hello, world'

(see also ``walk`` and ``glob``)

Reading with delimited blocks:

.. code-block:: python

   >>> s3.read_block(path, offset=1000, length=10, delimiter=b'\n')
   b'A whole line of text\n'

Writing with blocked caching:

.. code-block:: python

   >>> s3 = s3fs.S3FileSystem(anon=False)  # uses default credentials
   >>> with s3.open('mybucket/new-file', 'wb') as f:
   ...     f.write(2*2**20 * b'a')
   ...     f.write(2*2**20 * b'a') # data is flushed and file closed
   >>> s3.du('mybucket/new-file')
   {'mybucket/new-file': 4194304}

Because S3Fs faithfully copies the Python file interface it can be used
smoothly with other projects that consume the file interface like ``gzip`` or
``pandas``.

.. code-block:: python

   >>> with s3.open('mybucket/my-file.csv.gz', 'rb') as f:
   ...     g = gzip.GzipFile(fileobj=f)  # Decompress data with gzip
   ...     df = pd.read_csv(g)           # Read CSV file with Pandas

Limitations
-----------

This project is meant for convenience, rather than feature completeness.
The following are known current omissions:

- file access is always binary (although ``readline`` and iterating by line
  are possible)

- no permissions/access-control (i.e., no ``chmod``/``chown`` methods)


Logging
-------

The logger ``s3fs.core.logger`` provides information about the operations of the
file system. To see messages, set its level to ``DEBUG``. You can also achieve this via
an environment variable ``S3FS_LOGGING_LEVEL=DEBUG``.

Credentials
-----------

The AWS key and secret may be provided explicitly when creating an ``S3FileSystem``.
A more secure way, not including the credentials directly in code, is to allow
boto to establish the credentials automatically. Boto will try the following
methods, in order:

- ``aws_access_key_id``, ``aws_secret_access_key``, and ``aws_session_token``
  environment variables

- configuration files such as ``~/.aws/credentials``

- for nodes on EC2, the IAM metadata provider

In a distributed environment, it is not expected that raw credentials should
be passed between machines. In the explicitly provided credentials case, the
method ``get_delegated_s3pars()`` can be used to obtain temporary credentials.
When not using explicit credentials, it should be expected that every machine
also has the appropriate environment variables, config files or IAM roles
available.

If none of the credential methods are available, only anonymous access will
work, and ``anon=True`` must be passed to the constructor.

Furthermore, ``S3FileSystem.current()`` will return the most-recently created
instance, so this method could be used in preference to the constructor in
cases where the code must be agnostic of the credentials/config used.

Requester Pays Buckets
----------------------

Some buckets, such as the `arXiv raw data
<https://arxiv.org/help/bulk_data_s3>`__, are configured so that the
requester of the data pays any transfer fees.  You must be
authenticated to access these buckets and (because these charges maybe
unexpected) amazon requires an additional key on many of the API
calls. To enable ``RequesterPays`` create your file system as


.. code-block:: python

   >>> s3 = s3fs.S3FileSystem(anon=False, requester_pays=True)


Serverside Encryption
---------------------

For some buckets/files you may want to use some of s3's server side encryption
features. ``s3fs`` supports these in a few ways


.. code-block:: python

   >>> s3 = s3fs.S3FileSystem(
   ...     s3_additional_kwargs={'ServerSideEncryption': 'AES256'})

This will create an s3 filesystem instance that will append the
ServerSideEncryption argument to all s3 calls (where applicable).

The same applies for ``s3.open``.  Most of the methods on the filesystem object
will also accept and forward keyword arguments to the underlying calls.  The
most recently specified argument is applied last in the case where both
``s3_additional_kwargs`` and a method's ``**kwargs`` are used.

The ``s3.utils.SSEParams`` provides some convenient helpers for the serverside
encryption parameters in particular.  An instance can be passed instead of a
regular python dictionary as the ``s3_additional_kwargs`` parameter.


Bucket Version Awareness
------------------------

If your bucket has object versioning enabled then you can add version-aware support
to ``s3fs``.  This ensures that if a file is opened at a particular point in time that
version will be used for reading.

This mitigates the issue where more than one user is concurrently reading and writing
to the same object.

.. code-block:: python

   >>> s3 = s3fs.S3FileSytem(version_aware=True)
   # Open the file at the latest version
   >>> fo = s3.open('versioned_bucket/object')
   >>> versions = s3.object_version_info('versioned_bucket/object')
   # Open the file at a particular version
   >>> fo_old_version = s3.open('versioned_bucket/object', version_id='SOMEVERSIONID')

In order for this to function the user must have the necessary IAM permissions to perform
a GetObjectVersion


Contents
========

.. toctree::
   install
   api
   :maxdepth: 2


.. _boto3: https://boto3.readthedocs.io/en/latest/

Indices and tables
==================

* :ref:`genindex`
* :ref:`modindex`
* :ref:`search`
