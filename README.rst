Mímir
=====

When training machine learning models there are often many things we
want to log: training error, validation error, gradient and weights
norms, samples, etc. There are a few considerations:

-  For long-running experiments the log should be streamed to disk so
   that memory-usage doesn't grow.
-  The log should be stored in a format that is portable, easy to
   analyze, and space-efficient.
-  We want to be able to plot and analyze the log while the experiment
   is still running, potentially over the network.

Mímir stores logs as line-delimited JSON data and can stream them to
disk as a `gzipped
files <https://github.com/madler/zlib/blob/master/examples/gzlog.c>`__.
It can also publish new entries over TCP sockets using
`ZeroMQ <http://zeromq.org/>`__, enabling things such as live plotting.

.. contents::

Quickstart
----------

To use Mímir, simply create a logger object:

.. code:: python

    import mimir
    import time

    logger = mimir.Logger()
    for i in range(100):
        logger.log({'iteration': i, 'training_error': 1. / (i + 1)})
        time.sleep(1)

By default the log will just print the entry to standard output and then
discard it. If you don't want to print anything, pass
``formatter=None``, or pass a custom ``formatter`` to change the way the
data is printed.

Accessing entries
~~~~~~~~~~~~~~~~~

If you want to keep entries in memory so that you can access past
entries, pass a nonzero ``maxlen`` argument, which determines the
maximum number of entries kept in memory. This is done so that
long-running experiments don't run out of memory.

.. code:: python

    logger = mimir.Logger(maxlen=10)
    logger.log({'iteration': 0, 'training_error': 10})
    assert logger[-1]['training_error'] == 10

If you're sure that you won't run out of memory you can use
``maxlen=sys.maxsize``.

Saving to disk
~~~~~~~~~~~~~~

We often want to save the log to disk to analyze it afterwards. Mímir
allows you to save the log as line-delimited JSON files.

.. code:: python

    logger = mimir.Logger(filename='log.jsonl.gz')
    for i in range(100):
        logger.log({'iteration': i, 'training_error': 1. / (i + 1)})
        time.sleep(1)

If the filename ends with ``.gz`` the log will be compressed in a
streaming manner using
`gzlog <https://github.com/madler/zlib/blob/master/examples/gzlog.c>`__.

Streaming
~~~~~~~~~

Mímir can stream log entries over a TCP socket which clients can connect
to, both locally as well as over a network. This allows you to do things
like live-plotting your experiments. To enable this, pass
``stream=True``. By default the data is streamed, which means that
clients only get the entries from after when they joined. If you want
clients to receive past log entries as well, there is a
``stream_maxlen`` argument similar to the ``maxlen`` argument.

.. code:: python

    logger = mimir.Logger(stream=True, stream_maxlen=50)
    for i in range(100):
        logger.log({'iteration': i, 'training_error': 1. / (i + 1)})
        time.sleep(1)

To see a live plot of your log, open up a Jupyter notebook and type the
following (requires Bokeh). It will plot the last 50 datapoints, and
then live plot every entry as it comes in.

.. code:: python

    import mimir.plot
    mimir.plot.notebook_plot('iteration', 'training_error')

Context manager
~~~~~~~~~~~~~~~

The logger object can be used as a context manager, in which case all
file objects are closed when the runtime context is exited.

.. code:: python

    with Logger(filename='log.jsonl') as logger:
        logger.log({'iteration': 0, 'training_error': 10})

Log analysis
------------

To analyze the training logs `jq <https://stedolan.github.io/jq/>`__ is
recommended. Most operations can be done easily on the command line.

.. code:: bash

    # Get all training errors
    cat log.jsonl | jq '.training_error'

    # For compressed logs
    gunzip -c log.jsonl.gz | jq '.training_error'

    # Equivalently
    zcat log.jsonl.gz | jq '.training_error'

To operate on the entire log as one array use the ``-s`` (slurp) flag.

.. code:: bash

    cat log.json | jq -s 'min_by(.training_error)'

If your log entries have an irregular set of keys (e.g. if you only draw samples
every *n* iterations) you use the `select` function to filter these out.

.. code:: json

    {"iteration": 0, "training_error": 1.2}
    {"iteration": 1, "training_error": 0.7, "sample": 0.2}
    {"iteration": 2, "training_error": 0.3}

.. code:: bash

    cat log.jsonl | jq 'select(.sample)

If you want to write the log back to a file after operating on use the ``-c``
flag for compact output.

.. code:: bash

    # Sorting the log by a timestamp
    cat log.json | jq -s -c 'sort_by(.timestamp)[]' > sorted_log.json

    # Subsampling the log
    cat log.json | jq 'select(.iteration % 100 == 0).training_error' | less

JSON serialization
------------------

For streaming log entries over TCP sockets and saving logs to disk,
Mímir uses JSON. To serialize non-basic types you need to pass a custom
serialization function. Any keyword arguments passed to the ``Logger``
class will be passed to ``json.dumps``. The following is an example of
logging NumPy objects.

.. code:: python

    import base64
    import gzip
    import json

    import numpy
    from numpy.lib.format import header_data_from_array_1_0

    def serialize_numpy(obj):
        if isinstance(obj, np.ndarray):
            if not obj.flags.c_contiguous and not obj.flags.f_contiguous:
               obj = numpy.ascontiguousarray(obj)
            dct = header_data_from_array_1_0(obj)
            data = base64.b64encode(obj.data)
            dct['__ndarray__'] = data
            return dct
        raise TypeError

    def deserialize_numpy(dct):
        if '__ndarray__' in dct:
            data = base64.b64decode(dct['__ndarray__'])
            obj = numpy.frombuffer(data, dtype=dct['descr'])
            if dct['fortran_order']:
                obj.shape = dct['shape'][::-1]
                obj = obj.transpose()
            else:
                obj.shape = dct['shape']
            return obj
        return dct

    logger = mimir.Logger(filename='log.jsonl.gz', default=serialize_numpy)
    logger.log({'iteration': 0, 'data': numpy.random.rand(10, 10)})

    # The JSON data is written with UTF-8 encoding
    with codecs.getreader('utf-8')(gzip.open('log.jsonl.gz', 'rb')) as f:
        entry = json.loads(f.readline(), obj_hook=deserialize_numpy)
