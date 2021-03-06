The GeoMesa API
===============

This chapter describes the data structures and usage of the GeoMesa API.

GeoTools DataStore
------------------

GeoMesa provides multiple classes that implement the `GeoTools <http://geotools.org>`_ |geotools_version| `DataStore <http://docs.geotools.org/latest/userguide/library/api/datastore.html>`_ class, each of which uses a different backing store. These include ``DataStore``\ s that use Accumulo or Kafka.

Accumulo DataStore
^^^^^^^^^^^^^^^^^^
The data store module contains all of the Accumulo-related code for GeoMesa. This includes client code and distributed iterator code for the Accumulo tablet servers.

Creating a Data Store
~~~~~~~~~~~~~~~~~~~~~

An instance of an Accumulo data store can be obtained through the normal GeoTools discovery methods, assuming that the GeoMesa code is on the classpath:

.. code-block:: java

    Map<String, String> parameters = new HashMap<>;
    parameters.put("instanceId", "myInstance");
    parameters.put("zookeepers", "zoo1,zoo2,zoo3");
    parameters.put("user", "myUser");
    parameters.put("password", "myPassword");
    parameters.put("tableName", "my_table");
    org.geotools.data.DataStore dataStore = org.geotools.data.DataStoreFinder.getDataStore(parameters);

More information on using GeoTools can be found in the `GeoTools user guide <http://docs.geotools.org/stable/userguide/>`_.

Indexing Strategies
~~~~~~~~~~~~~~~~~~~

GeoMesa uses several different strategies to index simple features. In the code, these strategies are abstracted as 'tables'. For details on how GeoMesa encodes and indexes data, see tables. For details on how GeoMesa chooses and executes queries, see the ``org.locationtech.geomesa.accumulo.index.QueryPlanner`` and ``org.locationtech.geomesa.accumulo.index.QueryStrategyDecider`` classes.

Explaining:  Query Plans
++++++++++++++++++++++++

Given a data store and a query, you can ask GeoMesa to explain its plan for how to execute the query:

.. code-block:: java

    dataStore.getQueryPlan(query, explainer = new ExplainPrintln);

Instead of ``ExplainPrintln``, you can also use ``ExplainString`` or ``ExplainLogging`` to redirect the explainer output elsewhere.  (For the ``ExplainLogging``, it may be helpful to refer to GeoServer's `Advanced log configuration <http://docs.geoserver.org/2.8.x/en/user/advanced/logging.html>`_ documentation for the specifics of how and where to manage the GeoServer logs.)

Knowing the plan -- including information such as the indexing strategy -- can be useful when you need to debug slow queries.  It can suggest when indexes should be added as well as when query-hints may expedite execution times.

Iterator Stack
~~~~~~~~~~~~~~

GeoMesa uses Accumulo iterators to push processing out to the whole cluster. The iterator stack can be considered a 'tight inner loop' - generally, every feature returned will be processed in the iterators. As such, the iterators have been written for performance over readability.

We use several techniques to improve iterator performance. For one, we only deserialize the attributes of a simple feature that we need to evaluate a given query. When retrieving attributes, we always look them up by index, instead of by name. For aggregating queries, we create partial aggregates in the iterators, instead of doing all the processing in the client. The main goals are to minimize disk reads, processing and bandwidth as much as possible.

For more details, see the ``org.locationtech.geomesa.accumulo.iterators`` package.

Kafka DataStore
^^^^^^^^^^^^^^^

The ``KafkaDataStore`` is an implementation of the GeoTools
``DataStore`` interface that is backed by Apache Kafka. The
implementation supports the ability for feature producers to instantiate
a ``KafkaDataStore`` in *producer* mode to persist data into the data
store and for consumers to instantiate a ``KafkaDataStore`` in
*consumer* mode to read data from the data store. The producer and
consumer data stores can be run on separate servers. The only
requirement is that they can connect to the same instance of Apache
Kafka.

A ``KafkaDataStore`` in consumer mode supports two types of
``SimpleFeatureSource``'s: *live* and *replay*. A Kafka Consumer Feature
Source operating in *live* mode continually pulls data from the end of
the message queue (e.g. latest time) and always represents the latest
state of the simple features. A Kafka Consumer Feature Source operating
in *replay* mode will pull data from a specified time interval in the
past and can provide features as they existed at any point in time
within that interval.

Usage/Configuration
~~~~~~~~~~~~~~~~~~~

To create a ``KafkaDataStore`` there are two required properties, one
for the Apache Kafka connection, "brokers", and one for the Apache
Zookeeper connection, "zookeepers". An optional parameter, "zkPath" is
used to specify a path in Zookeeper under which schemas are stored. If
no "zkPath" is specified then a default path will be used. Another
optional parameter, "isProducer", is used to create a ``KafkaDataStore``
in *producer* or *consumer* mode. This parameter defaults to false, i.e.
by default a Kafka Consumer Data Store will be created. The same set of
configuration parameters, with the exception of "isProducer" must be
used to create both the Kafka Producer Data Store and the Kafka Consumer
Data Store.

After a ``KafkaDataStore`` has been created, additional simple feature
type specific hints must be provided. These hints are stored in the user
data of the ``SimpleFeatureType``. Use the ``KafkaDataStoreHelper`` to
create a copy of your ``SimpleFeatureType`` with the hints added. Then
call ``dataStore.createSchema(sft)`` where ``sft`` is the
``SimpleFeatureType`` returned by the ``KafkaDataStoreHelper``. This
must be done once per Simple Feature Type. The Simple Feature Type along
with hints are stored in Zookeeper so if ``createSchema(sft)`` is called
on the Kafka Data Store Producer it cannot be called on the Kafka
Consumer Data Store Consumer.

Data Producers
~~~~~~~~~~~~~~

First, create the data store. For example:

.. code-block:: scala

    String brokers = ...
    String zookeepers = ...
    String zkPath = ...

    // build parameters map
    Map<String, Serializable> params = new HashMap<>();
    params.put("brokers", brokers);
    params.put("zookeepers", zookeepers);
    params.put("isProducer", Boolean.TRUE);

    // optional
    params.put("zkPath", zkPath);

    // create the data store
    KafkaDataStoreFactory factory = new KafkaDataStoreFactory();
    DataStore producerDs = factory.createDataStore(params);

Next, create the schema. Each data store can have one or many schemas.
For example:

.. code-block:: scala

    SimpleFeatureType sft = ...
    SimpleFeatureType streamingSFT = KafkaDataStoreHelper.createStreamingSFT(sft, zkPath);
    producerDs.createSchema(streamingSFT);

The call to ``KafkaDataStoreHelper.createSchema`` creates a copy of the
``sft`` with the required hint added. In this case the hint is the name
of the Kafka topic. The ``zkPath`` parameter is uses to make the Kafka
topic name unique to the ``zkPath`` used by the ``KafkaDataStore`` so
that the same ``SimpleFeatureType`` can be used by multiple
``KafkaDataStore``\ s where each data store has a different ``zkPath``.
Specifically, the resulting Kafka topic's name will be zkPath-sftName
where the forward slashes in the ``zkPath`` are replaced by ``-``.  e.g. creating a schema
with a ``zkPath`` of /geomesa/ds/kafka with an ``sft`` called example_sft will
create a Kafka topic called geomesa-ds-kafka-example_sft.

The ``createSchema`` method will throw an exception if the given
``SimpleFeatureType`` does not contain the required hint, i.e., if it
was not created by the ``KafkaDataStoreHelper``.

Now, you can create or update simple features:

.. code-block:: scala

    // the name of the simple feature type -  will be the same as sft.getTypeName();
    String typeName = streamingSFT.getTypeName();

    FeatureWriter<SimpleFeatureType, SimpleFeature> fw =
            producerDs.getFeatureWriter(typeName, null, Transaction.AUTO_COMMIT);
    SimpleFeature sf = fw.next();
    // set properties on sf
    fw.write();

Delete simple features:

.. code-block:: scala

    SimpleFeatureStore producerStore = (SimpleFeatureStore) producerDs.getFeatureSource(typeName);
    FilterFactory2 ff = CommonFactoryFinder.getFilterFactory2();

    String id = ...
    producerStore.removeFeatures(ff.id(ff.featureId(id)));

And, clear (delete all) features:

.. code-block:: scala

    producerStore.removeFeatures(Filter.INCLUDE);

Each operation that creates, modifies, deletes, or clears simple
features results in a message being sent to the Kafka topic.

Data Consumers
~~~~~~~~~~~~~~

First, create the data store. For example:

::

    String brokers = ...
    String zookeepers = ...
    String zkPath = ...

    // build parameters map
    Map<String, Serializable> params = new HashMap<>();
    params.put("brokers", brokers);
    params.put("zookeepers", zookeepers);

    // optional - the default is false
    params.put("isProducer", Boolean.FALSE);

    // optional
    params.put("zkPath", zkPath);

    // create the data store
    KafkaDataStoreFactory factory = new KafkaDataStoreFactory();
    DataStore consumerDs = factory.createDataStore(params);

The ``brokers``, ``zookeepers``, and ``zkPath`` parameters must be
consistent with the values used to create the Kafka Data Store Producer.

Because ``createSchema`` was called on the Kafka Data Store Producer, it
does not need to be called on the Consumer. Calling ``createSchema``
with a ``SimpleFeatureType`` that has already been created will result
in an exception being thrown. Note that all ``SimpleFeature``\ s
returned by the Kafka Data Store consumer will have a
``SimpleFeatureType`` equal to the ``streamingSFT`` created when setting
up the producer, i.e. the ``SimpleFeatureType`` will include the hint
added by ``KafkaDataStoreHelper.createStreamingSFT``.

Now that the Kafka Data Store Consumer has been created it can be
queried in either *live* or *replay* mode.

Live Mode
~~~~~~~~~

Live mode is the default and requires no extra setup. In this mode the
``SimpleFeatureSource`` contains the current state of the
``KafkaDataStore``. As ``SimpleFeatures`` are created, modified,
deleted, or cleared by the Kafka Data Store Producer, the current state
is updated. All queries to the ``SimpleFeatureSource`` are queries
against the current state. For example:

::

    String typeName = ...
    SimpleFeatureSource liveFeatureSource = consumerDs.getFeatureSource(typeName);

    Filter filter = ...
    liveFeatureSource.getFeatures(filter);

It is also possible to provide a CQL filter to the getFeatureSource method call which will ensure
the resulting ``FeatureSource`` only contains certain records. Providing a filter to reduce the number of
returned records will provide a performance boost when using the featureSource.

::

    String typeName = ...
    SimpleFeatureSource liveFeatureSource = consumerDs.getFeatureSource(typeName, filter);

Replay Mode
~~~~~~~~~~~

Replay mode allows the a user to query the ``KafkaDataStore`` as it
existed at any point in the past. Queries against a Kafka Replay Simple
Feature source specify a historical time to query and only the set and
version of ``SimpleFeature``\ s that existed at that point in time will
be used to answer the query.

In order to use Replay mode some additional hints are required: the
start and end times of the replay window and a read behind duration:

::

    Instant replayStart = ...
    Instant replayEnd = ...
    Duration readBehind = ...
    ReplayConfig replayConfig = new ReplayConfig(replayStart, replayEnd, readBehind);

The replay window is simply an optimization that allows the Kafka Replay
Feature Source to load, at initialization time, all state changes that
occur within the window. Any query for a time outside of the window will
return no results even if features existed at that time.

The read behind is the amount of time used to rebuild state. For
example, if ``readBehind = 5s`` then for a query requesting state at
``time = t`` all state changes that occurred between ``t - 5s`` and
``t`` will be used to build the state at time ``t`` which will then be
used to answer the query. Selecting an appropriate read behind requires
an understanding of the producer. The expected uses case is a producer
that updates every simple feature, even if it hasn't changed, at a
regular interval. For example, if the producer is updating every ``x``
seconds then a read behind of ``x + 1s`` might be appropriate.

During initialization of the Kafka Replay Feature Source all state
changes from ``replayStart - readBehind`` to ``replayEnd`` will be read
and cached. As the size of the replay window and read behind increases
so does the amount of data that must be read and cashed. So, both the
size of the window and the read behind should be kept as small as
possible.

After creating the ``ReplayConfig`` pass it, along with the
``streamingSFT`` to the ``KafkaDataStoreHelper``:

::

    SimpleFeatureType streamingSFT = consumerDs.getSchema(typeName);
    SimpleFeatureType replaySFT = KafkaDataStoreHelper.createReplaySFT(streamingSFT, replayConfig);

The ``streamingSFT`` passed to ``createReplaySFT`` must contain the
hints added by ``KafkaDataStoreHelper.createStreamingSFT``. The easiest
way to ensure this is to call ``consumerDs.getSchema(typeName)``. The
``SimpleFeatureType`` returned by ``createReplaySFT`` will contain the
hint added by ``createStreamingSFT`` as well as a a hint containing the
``ReplayConfig``. Additionally the ``replaySFT`` will have a different
name then then ``streamingSFT``. This is to differentiate *live* and
*replay* ``SimpleFeatureType``\ s. The ``replaySFT`` will also contain
an additional attribute, ``KafkaLogTime``, of type ``java.util.Date``
which represents the historical query time.

After creating the ``replaySFT`` the Kafka Replay Feature Source may be
created:

::

    consumerDs.createSchema(replaySFT);

    String replayTimeName = replaySFT.getTypeName();
    SimpleFeatureSource replayFeatureSource = consumerDs.getFeatureSource(replayTimeName);

The call to ``createSchema`` is required because the ``replaySFT`` is a
new ``SimpleFeatureType``.

Finally the Kafka Replay Consumer Feature Source can be queried:

::

    Instant historicalTime = ...
    Filter timeFilter = ff.and(filter, ReplayTimeHelper.toFilter(historicalTime));

    replayFeatureSource.getFeatures(timeFilter);

Command Line Tools
~~~~~~~~~~~~~~~~~~

The KafkaGeoMessageFormatter, part of geomesa-kafka-datastore, may be
used with the ``kafka-console-consumer``, part of Apache Kafka. In order
to use this formatter call the kafka-console consumer with these
additional arguments:

::

    --formatter org.locationtech.geomesa.kafka.KafkaGeoMessageFormatter
    --property sft.name={sftName}
    --property sft.spec={sftSpec}

In order to pass the spec via a command argument all ``%`` characters
must be replaced by ``%37`` and all ``=`` characters must be replaced by
``%61``.

A slightly easier to use but slightly less flexible alternative is to
use the ``KafkaDataStoreLogViewer`` instead of the
``kafka-console-consumer``. To use the ``KafkaDataStoreLogViewer`` first
copy the geomesa-kafka-geoserver-plugin.jar to $KAFKA\_HOME/libs. Then
create a copy of $KAFKA\_HOME/bin/kafka-console-consumer.sh called
"kafka-ds-log-viewer" and in the copy replace the classname in the exec
command at the end of the script with
``org.locationtech.geomesa.kafka.KafkaDataStoreLogViewer``.

The ``KafkaDataStoreLogViewer`` requires three arguments:
``--zookeeper``, ``--zkPath``, and ``--sftName``. It also supports an
optional argument ``--from`` which accepts values ``oldest`` and
``newest``. ``oldest`` is equivalent to specifying ``--from-beginning``
when using the ``kafka-console-consumer`` and ``newest`` is equivalent
to not specifying ``--from-beginning``.

For example:

.. code-block:: bash

    $ kafka-ds-log-viewer --zookeeper {zookeeper} --zkPath {zkPath} --sftName {sftName}

The ``KafkaDataStoreLogViewer`` loads the ``SimpleFeatureType`` from
Zookeeper so it does not need to be passed via the command line.
