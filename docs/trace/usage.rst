OpenCensus Trace for Python
===========================

    `Census`_ for Python. Census provides a framework to measure a server's resource
    usage and collect performance stats. This repository contains Python related
    utilities and supporting software needed by Census.

    .. _Census: https://github.com/census-instrumentation

|circleci|

.. |circleci| image:: https://circleci.com/gh/census-instrumentation/opencensus-python.svg?style=shield
   :target: https://circleci.com/gh/census-instrumentation/opencensus-python

-  `API Documentation`_

.. _API Documentation: http://opencensus.io/opencensus-python/trace/usage.html

Installation & basic usage
--------------------------

1. Install the opencensus-trace package using `pip`_ or `pipenv`_:

    ::

        pip install opencensus-trace
        pipenv install opencensus-trace

2. Initialize a tracer for application:

    .. code:: python

        from opencensus.trace import tracer as tracer_module

        tracer = tracer_module.Tracer()

    .. _pip: https://pip.pypa.io
    .. _pipenv: https://docs.pipenv.org/

Usage
-----

You can collect traces using the ``Tracer`` `context manager`_:

.. code:: python

    from opencensus.trace import tracer as tracer_module

    # Initialize a tracer, by default using the `PrintExporter`
    tracer = tracer_module.Tracer()

    # Example for creating nested spans
    with tracer.span(name='span1') as span1:
        do_something_to_trace()
        with span1.span(name='span1_child1') as span1_child1:
            do_something_to_trace()
        with span1.span(name='span1_child2') as span1_child2:
            do_something_to_trace()
    with tracer.span(name='span2') as span2:
        do_something_to_trace()

Census will collect everything within the ``with`` statement as a single span.

Alternatively, you can explicitly start and end a span:

.. code:: python

    from opencensus.trace import tracer as tracer_module

    # Initialize a tracer, by default using the `PrintExporter`
    tracer = tracer_module.Tracer()

    tracer.start_span(name='span1')
    do_something_to_trace()
    tracer.end_span()


.. _context manager: https://docs.python.org/3/reference/datamodel.html#context-managers


Customization
-------------

Samplers
~~~~~~~~

You can specify different samplers when initializing a tracer, default
is using ``AlwaysOnSampler``, the other options are ``AlwaysOffSampler``
and ``ProbabilitySampler``

.. code:: python

    from opencensus.trace.samplers import probability
    from opencensus.trace import tracer as tracer_module

    # Sampling the requests at the rate equals 0.5
    sampler = probability.ProbabilitySampler(rate=0.5)
    tracer = tracer_module.Tracer(sampler=sampler)

Exporters
~~~~~~~~~

You can choose different exporters to send the traces to. By default,
the traces are printed to stdout in JSON format. Other options include
writing to a file, sending to Python logging, or reporting to
Stackdriver.

This example shows how to configure Census to save the traces to a
file:

.. code:: python

    from opencensus.trace.exporters import file_exporter
    from opencensus.trace.tracers import context_tracer

    exporter = file_exporter.FileExporter(file_name='traces')
    tracer = context_tracer.ContextTracer(exporter=exporter)

This example shows how to report the traces to Stackdriver Trace:

.. code:: python

    from opencensus.trace.exporters import stackdriver_exporter
    from opencensus.trace import tracer as tracer_module

    exporter = stackdriver_exporter.StackdriverExporter(
        project_id='your_cloud_project')
    tracer = tracer_module.Tracer(exporter=exporter)

Propagators
~~~~~~~~~~~

You can specify the propagator type for serializing and deserializing the
``SpanContex`` and its headers. There are currently two built in propagators:
``GoogleCloudFormatPropagator`` and ``TextFormatPropagator``.

This example shows how to use the ``GoogleCloudFormatPropagator``:

.. code:: python

    from opencensus.trace.propagation import google_cloud_format

    propagator = google_cloud_format.GoogleCloudFormatPropagator()

    # Deserialize
    span_context = propagator.from_header(header)

    # Serialize
    header = propagator.to_header(span_context)

Blacklist Paths
~~~~~~~~~~~~~~~

You can specify which paths you do not want to trace by configuring the
blacklist paths.

This example shows how to configure the blacklist to ignore the `_ah/health` endpoint
for a Flask application:

.. code:: python

    from opencensus.trace.ext.flask.flask_middleware import FlaskMiddleware

    app = flask.Flask(__name__)

    blacklist_paths = ['_ah/health']
    middleware = FlaskMiddleware(app, blacklist_paths=blacklist_paths)

For Django, you can configure the blacklist in the ``OPENCENSUS_PARAMS`` in ``settings.py``:

.. code:: python

    OPENCENSUS_PARAMS: {
        ...
        'BLACKLIST_PATHS': ['_ah/health',],
    }


.. note:: By default the health check path for the App Engine flexible environment is not traced,
    but you can turn it on by excluding it from the blacklist setting.

Framework Integration
---------------------

Opencensus supports integration with popular web frameworks including
Django, Flask, and Webapp2. When the application receives a HTTP request,
the tracer will automatically generate a span context using the trace
information extracted from the request headers, and propagated to the
child spans.

Flask
~~~~~

In your application, use the middleware to wrap your app and the
requests will be automatically traced.

.. code:: python

    from opencensus.trace.ext.flask.flask_middleware import FlaskMiddleware

    app = flask.Flask(__name__)

    # You can also specify the sampler, exporter, propagator in the middleware,
    # default is using `AlwaysOnSampler` as sampler, `PrintExporter` as exporter,
    # `GoogleCloudFormatPropagator` as propagator.
    middleware = FlaskMiddleware(app)

Django
~~~~~~

For tracing Django requests, you will need to add the following line to
the ``MIDDLEWARE_CLASSES`` section in the Django ``settings.py`` file.

.. code:: python

    MIDDLEWARE_CLASSES = [
        ...
        'opencensus.trace.ext.django.middleware.OpencensusMiddleware',
    ]

And add this line to the ``INSTALLED_APPS`` section:

.. code:: python

    INSTALLED_APPS = [
        ...
        'opencensus.trace.ext.django',
    ]

You can configure the the sampler, exporter, propagator using the ``OPENCENSUS_TRACE`` setting in
``settings.py``:

.. code:: python

    OPENCENSUS_TRACE = {
        'SAMPLER': 'opencensus.trace.samplers.probability.ProbabilitySampler',
        'REPORTER': 'opencensus.trace.exporters.print_exporter.PrintExporter',
        'PROPAGATOR': 'opencensus.trace.propagation.google_cloud_format.'
                      'GoogleCloudFormatPropagator',
    }

You can configure the sampling rate and other parameters using the ``OPENCENSUS_TRACE_PARAMS``
setting in ``settings.py``:

.. code:: python

    OPENCENSUS_TRACE_PARAMS = {
        'BLACKLIST_PATHS': ['/_ah/health'],
        'GCP_EXPORTER_PROJECT': None,
        'SAMPLING_RATE': 0.5,
        'ZIPKIN_EXPORTER_SERVICE_NAME': 'my_service',
        'ZIPKIN_EXPORTER_HOST_NAME': 'localhost',
        'ZIPKIN_EXPORTER_PORT': 9411,
    }

Webapp2
~~~~~~~

.. code:: python

    from opencensus.trace.tracers import webapp2_tracer

    tracer = webapp2_tracer.WebApp2Tracer()

    with tracer.span(name='span1'):
        do_something_to_trace()


Service Integration
-------------------

Opencensus supports integration with various popular outbound services such as
MySQL and Requests. To enable integration you will need to pass the list of
services to census:

.. code:: python

    from opencensus.trace import config_integration
    from opencensus.trace import tracer as tracer_module

    import mysql.connector

    # Trace both mysql-connection and psycopg2
    integration = ['mysql', 'postgresql']

    config_integration.trace_integrations(integration)


MySQL
~~~~~

The integration with MySQL supports the `mysql-connector`_ library and is specified
to ``trace_integrations`` using ``'mysql'``.

.. _mysql-connector: https://pypi.org/project/mysql-connector

PostgreSQL
~~~~~~~~~~

The integration with PostgreSQL supports the `psycopg2`_ library and is specified
to ``trace_integrations`` using ``'postgresql'``.

.. _psycopg2: https://pypi.org/project/psycopg2


SQLAlchemy
~~~~~~~~~~

You can trace usage of `sqlalchemy package`_, regardless of the underlying database, by
specifying ``'sqlalchemy'`` to ``trace_integrations``.

.. _SQLAlchemy package: https://pypi.org/project/SQLAlchemy

.. note:: If you enable tracing of SQLAlchemy and the underlying database
    driver, you will get duplicate spans. Instead, just trace SQLAlchemy.

Requests
~~~~~~~~

Census can trace HTTP requests made with the `Requests`_ library. The request URL,
method, and status will be collected.

You can enable Requests integration by specifying ``'requests'`` to ``trace_integrations``.


Contributing
------------

Contributions to this library are always welcome and highly encouraged.

See `CONTRIBUTING <CONTRIBUTING.md>`__ for more information on how to
get started.


Development
-----------

Tests
~~~~~

::

    cd trace
    tox -e py34
    source .tox/py34/bin/activate

    # Run the unit test
    pip install nox-automation

    # See what's available in the nox suite
    nox -l

    # Run a single nox command
    nox -s "unit_tests(python_version='2.7')"

    # Run all the nox commands
    nox

    # Integration test
    # We don't have script for integration test yet, but can test as below.
    python setup.py bdist_wheel
    cd dist
    pip install opencensus-0.0.1-py2.py3-none-any.whl

    # Then just run the tracers normally as you want to test.

License
-------

Apache 2.0 - See `LICENSE <LICENSE>`__ for more information.

Disclaimer
----------

This is not an official Google product.
