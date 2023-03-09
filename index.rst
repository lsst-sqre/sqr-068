:tocdepth: 1

.. sectnum::

.. Metadata such as the title, authors, and description are set in metadata.yaml

.. TODO: Delete the note below before merging new content to the main branch.


Abstract
========

Sasquatch is a service for recording, displaying, and alerting on Rubin Observatory's telemetry data and scalar metrics.

It is a unification of the Engineering and Facilities Database (EFD) :cite:`SQR-034` and SQuaSH :cite:`SQR-009` under the same deployment.

The new features include a REST API for sending data to Sasquatch and two-way data replication between the Summit and USDF.
This way, observatory telemetry produced at the Summit and metrics computed at the USDF or Summit are always available locally.

Sasquatch can be easily extended to record other `time-series data`_ such as camera diagnostic metrics, rapid analysis metrics, scheduler events etc.

In its third generation, we took the opportunity to rebrand it to Sasquatch, and it should be understood and the `service` that manages the EFD and other time-series `databases`.

Sasquatch is currently deployed at Summit, USDF, Tucson test stand and Base test stand through `Phalanx`_.

.. _time-series data: https://www.influxdata.com/what-is-time-series-data
.. _Phalanx: https://phalanx.lsst.io

Overview
========

Sasquatch is based on `InfluxDB`_, an open-source time-series database optimized for efficient storage and analysis of time series data, and `Apache Kafka`_ which is used as a write-ahead log to InfluxDB and for data replication between sites.

We are taking the opportunity to migrate to `InfluxDB OSS 2.x`_  which uses Flux for querying and data analaysis, in addition to InfluxQL.
This version also has a new `task engine`_ to process time-series data with Flux, and a new `Python client`_.

Apache Kafka is now deployed with `Strimzi`_, a Kubernetes operator to manage the Kafka resources.
It also includes resources to manage Mirror Maker 2 used for data replication, which is an improvement compared to the previous deployment (see  :cite:`SQR-050`).
In addition to `Strimzi_` components we also deploy the Confluent REST proxy, used in Sasquatch for connecting HTTP-based clients with Kafka.

Figure 1 shows a diagram of the Sasquatch architecutre highlighting the new functionalities: two-way replication between the Summit and USDF; multiple InfluxDB databases; Flux Tasks; and a REST API based on the Confluent REST proxy.

.. figure:: /_static/sasquatch_overview.svg
   :name: Sasquatch architecture overview.


.. _InfluxDB: https://www.influxdata.com/time-series-database
.. _Apache kafka: https://kafka.apache.org
.. _InfluxDB OSS 2.x: https://docs.influxdata.com/influxdb/latest/
.. _Flux: https://docs.influxdata.com/flux/v0.x/get-started/
.. _buckets: https://docs.influxdata.com/influxdb/latest/organizations/buckets/
.. _task engine: https://docs.influxdata.com/influxdb/latest/process-data/
.. _Python client: https://docs.influxdata.com/influxdb/latest/api-guide/client-libraries/python/
.. _Strimzi: https://strimzi.io
.. _Kafka bridge: https://strimzi.io/docs/bridge/latest/#assembly-kafka-bridge-overview-bridge

Sending data to Sasquatch
=========================

There are two main mechanisms for sending data to Sasquatch.
One is based on the SAL Kafka Producers (`ts_salkafka`_) and the other is based on the Confluent REST proxy.

`ts_salkafka`_ is currently used with Sasquatch at the Summit and test stands to forward DDS messages to Kafka.
Once DDS is replaced by Kafka, the CSCs will write directly to Kafka and the ts_salkafka won't be necessary anymore :cite:`TSTN-033`.

.. _ts_salkafka: https://ts-salkafka.lsst.io

Confluent REST proxy
--------------------

The `Confluent REST proxy`_ provides a REST interface for connecting HTTP-based clients with Kafka.

With the REST proxy, a client can produce messages to or consume messages from Kafka topics using HTTP requests which simplifies the integration with Sasquatch.
In particular, the REST proxy integrates well with the Schema Registry and so this mechanism support sending Avro messages with an schema.

Once the data lands in Kafka, an InfluxDB Sink connector is responsible for consuming the Kafka topic and writing the data to a bucket in InfluxDB.

In addition, everything that is sent to kafka can be replicated to other sites, and
we can also persist data into other formats like Parquet using off-the-shell Kafka connectors.

.. _Confluent REST proxy: https://docs.confluent.io/platform/current/kafka-rest/

Two-way replication between Summit and USDF
===========================================

In the current EFD implementation, data replication between the Summit and USDF is done throught the Kafka Mirror Maker 2 connector (MM2) :cite:`SQR-050`.

The EFD replication service allows for one-way replication (or active/standby replication) from the Summit to the USDF.
We have measured sub-second latency for high throughput topics in the MTM1M3 subsystem in this set up.

In Sasquatch, two-way replication (or active/active replication) is now required.
With two-way replication, metrics computed at USDF (e.g. from Prompt Processing), for example, sent to the USDF instance of Sasquatch can be replicated to the Summit.

In addition to the instance of MM2 configured at USDF to replicate Observatory telemetry, events and metrics from the Summit, Sasquatch adds a second instance of MM2 at the Summit.

The Kafka Topics to be replicated are listed in the MM2 configuration on each Kafka cluster.

Two-way replication requires Kafka Topic renaming.
Usually, in this scenario, the Kafka Topic at the destination cluster is prefixed with the name of the source cluster.
That helps to identify its origin and avoid replicating it back to the source cluster.

Consequently, topic schemas at the destination cluster need to be renamed adding some complexity compared to the one-way replication scneario.

Storing telemetry, metrics and events into multiple databases
=============================================================

In InfluxDB OSS 2.x, a database or `bucket`_ is a named location where time-series data is stored.

By using multiple buckets we can specify different retention policies, time precision, access control and backup strategies.
InfluxDB OSS 2.x provides a `buckets API`_ to programatically interact with buckets.

In the original EFD implementation, telemetry and events from the Observatory are recorded into a single InfluxDB database.
In Sasquatch, when migtrating to InfluxDB OSS 2.x we are planning on storing telemetry and events into separate buckets.
In particular, because the time difference between events is not regular, they need to be stored with higher time precision than telemetry and metrics to avoid overlaping data.

.. _bucket: https://docs.influxdata.com/influxdb/latest/organizations/buckets/
.. _buckets API: https://docs.influxdata.com/influxdb/latest/api/#tag/Buckets


Flux Tasks
==========

InfluxDB OSS 2.x provides a new `task engine`_ that replaces Continuous Queries and Kapacitor used in InfluxDB OSS 1.x.

An InfluxDB task is a scheduled Flux script that takes an input data stream, transforms or analyzes it, and performs some action.

In most cases, the transformed data can be stored into a new InfluxDB bucket, or sent to other destinations using Flux output functions.
An example is sending a notification to Slack, or triggering some computation using the Flux `http.post()`_ function.

InfluxDB OSS 2.x also provides a `tasks API`_ to programatically interact with tasks.

.. _task engine: https://docs.influxdata.com/influxdb/latest/process-data/
.. _Flux output functions: https://docs.influxdata.com/flux/latest/function-types/#outputs
.. _http.post(): https://docs.influxdata.com/flux/v0.x/stdlib/http/post/
.. _tasks API: https://docs.influxdata.com/influxdb/latest/api/#tag/Tasks

Implementation phases
=====================

This section describes the Sasquatch implementation phases.

Phase 1 - Replace EFD deployments
---------------------------------

#. Add Sasquatch to Phalanx.
#. Enable Chronograf authentication through Gafaelfawr.
#. Replace Confluent Kafka with Strimzi Kafka.
#. Automate Strimzi Kafka image builds adding the InfluxDB Sink, Mirror Maker 2, and S3 connectors.
#. Deploy Sasquatch at IDF Dev.
#. Deploy Sasquatch at TTS (Pillan cluster).
#. Add ``csc`` and ``kafka-producer`` subcharts to Sasquatch for end-to-end testing.
#. Add SASL configuration to ``ts_salkafka``.
#. Test connectors and integration with CSCs.
#. Integrate news feeds with rsp_broacast.
#. Implement external listeners in Strimzi Kafka.
#. Migrate Sasquatch monitoring to monitoring.lsst.codes
#. Deploy Sasquatch at USDF (SLAC).
#. Migrate EFD data from the Summit to the Sasquatch instance at USDF.
#. Deploy Sasquatch at the Summit (Yagan cluster).
#. Migrate EFD data from the efd-temp-k3s.cp.lsst.org server to Sasquatch at the Summit.
#. Implement data replication bewteen Sasquatch at the Summit and USDF with Strimzi Kafka.
#. Deploy Sasquatch at the BTS (Manke cluster).

Related goals
^^^^^^^^^^^^^

#. Archive argocd-efd deployment repo, everything is in Phalanx.
#. Remove EFD related charts from the SQuaRE charts repo.
#. Decomissioning efd-temp-k3s.cp.lsst.org cluster.
#. Migrate EFD data from NCSA to SLAC.

Phase 2 - Replace the SQuaSH deployment
---------------------------------------

#. Implement Confluent REST proxy as a replacement for the SQuaSH API in Sasquatch.
#. Implement a Butler data store for Sasquatch.
#. Implement two-way replication in Sasquatch.
#. Migrate SQuaSH data to Sasquatch at USDF.

Related goals
^^^^^^^^^^^^^

#. Remove squash and influxdb-demo clusters on Google


Phase 3 - Migration to InfluxDB OSS 2.x
---------------------------------------

#. Add InfluxDB OSS 2.x to Sasquatch deployment.
#. Connect Chronograf with InfluxDB OSS 2.x (rquires DBRP mapping).
#. Replace InfluxDB Sink connector with Telegraf Kafka Consumer so it works with InfluxDB OSS 2.x.
#. Migrate EFD database to 2.x format (TTS, BTS, Summit, USDF).
#. Exercise InfluxDB OSS 2.x backup/restore tools.
#. Migrate Kapacitor alerts to Flux tasks.
#. Migrate Chronograf 1.x annotations (``_chronograf`` database) to InfluxDB 2.x.
#. Upgrage EFD client to use the InfluxDB OSS 2.x Python client.


.. rubric:: References
..
.. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
    :style: lsst_aa
