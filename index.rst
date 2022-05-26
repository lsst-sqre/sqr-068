:tocdepth: 1

.. sectnum::

.. Metadata such as the title, authors, and description are set in metadata.yaml

.. TODO: Delete the note below before merging new content to the main branch.

.. note::

   **This technote is a work-in-progress.**

Abstract
========

:dmtn:`082` :cite:`DMTN-082` presents a high-level architecture to enable real-time analysis of the Engineering and Facilities Database (EFD) through the Rubin Science Platform (RSP).

The current EFD implementation is described in :sqr:`034` :cite:`SQR-034` and deployed at the Summit, test stands, and USDF.

The EFD architecture is based on `InfluxDB`_, an open-source time-series platform optimized for efficient storage and analysis of time series data, and `Apache Kafka`_ which is used as a write-ahead log to InfluxDB and for data replication between the Summit and the USDF.

The EFD terminology, however, is confusing as the system is more than a database.
In fact, it can be extended to store and analize other scalar metrics such as science performance metrics computed by `lsst.faro`_, scheduler events, camera diagnostic metrics, etc.

In this technote, we introduce Sasquatch (Scientific Analysis of Scalar QUantities and Telemetry Curation Harness) a unified service to store and analyze telemetry, events and metrics for the Rubin Observatory.

.. _InfluxDB: https://www.influxdata.com/time-series-database
.. _Apache kafka: https://kafka.apache.org
.. _lsst.faro: https://pipelines.lsst.io/v/daily/modules/lsst.faro

Overview
========

Sasquatch architecture is based on InfluxDB OSS 2.x and Strimzi.

`InfluxDB OSS 2.x`_ uses Flux as its native language.
`Flux`_ combines query and data analysis functionalities.
Other features of interest for Sasquatch that are included in the new InfluxDB version are `buckets`_, a `task engine`_ to process time-series data with Flux, and a new `Python client`_.

`Strimzi`_ simplifies the process of running Apache Kafka in a Kubernetes cluster, providing a set of operators to manage the Kafka resources.
It also brings `Kafka bridge`_, a component used in Sasquatch for connecting HTTP-based clients with Kafka.

Figure 1 shows a diagram of the Sasquatch architecutre highlighting the new functionalities: a REST API based on Strimzi Kafka bridge; two-way replication between the Summit and USDF; InfluxDB buckets; and Flux Tasks.

.. figure:: /_static/sasquatch_overview.svg
   :name: Sasquatch architecture overview.


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
One is based on the `ts_salkafka`_ producer and the other is based on Strimzi Kafka Bridge.

`ts_salkafka`_ was implemented to send DDS messages to the EFD Kafka cluster.
It can still be used with Sasquatch at the Summit and test stands, but won't be necessary if DDS is replaced by Kafka (see `TSTN-033`_ :cite:`TSTN-033`).

.. _ts_salkafka: https://ts-salkafka.lsst.io
.. _TSTN-033: https://tstn-033.lsst.io

Strimzi Kafka bridge
--------------------

Strimzi Kafka bridge provides a REST interface for connecting HTTP-based clients with Kafka.

With Kafka Bridge a client can send messages to or receive messages from Kafka topics using HTTP requests.
In particular, a client can produce messages to topics in JSON format by using the `topics`_ endpoint.

Once the data lands in Kafka, an InfluxDB Sink connector is responsible for consuming the Kafka topic and writing the data to a bucket in InfluxDB.

Any client that needs to send data to Sasquatch would follow that pattern.
In general, the following steps are required to add a new client (1) create a Kafka Topic in Strimzi; (2) configure a new InfluxDB Sink connector; and (3) create a new bucket in InfluxDB.

In SQuaSH (see :sqr:`009` :cite:`SQR-009`) we use the SQuaSH API to connect ``dispatch_verify.py``, the HTTP client in the `lsst.verify`_ package, with InfluxDB.

In Sasquatch, the Strimzi Kafka bridge combined with an InfluxDB connector is a replacement for the SQuaSH API.
Because we send that to Kafka, it can be replicated or persisted into another format using Kafka connectors.

.. _topics: https://strimzi.io/docs/bridge/latest/#_send
.. _lsst.verify: https://pipelines.lsst.io/v/daily/modules/lsst.verify


Two-way replication between Summit and USDF
===========================================

In the current EFD implementation, data replication between the Summit and USDF is done throught the Kafka Mirror Maker 2 connector (MM2) (see :sqr:`050` :cite:`SQR-050`).

The EFD replication service allows for one-way replication (or active/standby replication) from the Summit to the USDF.
We have measured sub-second latency for a high throughput topic from the MTM1M3 subsystem in that set up.

In Sasquatch, two-way replication (or active/active replication) is now required.
With two-way replication, metrics computed at USDF (e.g. from Prompt Processing), for example, are sent to the USDF instance of Sasquatch and replicated to the Summit.

In addition to the instance of MM2 configured at USDF to replicate Observatory telemetry, events and metrics from the Summit, Sasquatch adds a second instance of MM2 at the Summit.

The Kafka Topics to be replicated are listed in the MM2 configuration on each Kafka cluster.

Two-way replication requires Kafka Topic renaming.
Usually, in this scenario, the Kafka Topic at the destination cluster is prefixed with the name of the source cluster.
That helps to identify its origin and avoid replicating it back to the source cluster.
Consequently, any topic schemas at the destination cluster need to be translated, which adds more complexity compared to the one-way replication scenario.


Storing telemetry, metrics and events into multiple buckets
===========================================================

In InfluxDB OSS 2.x, a `bucket`_ is a named location where time series data is stored.

By using multiple buckets we can specify different retention policies, time precision, access control and backup strategies.
InfluxDB OSS 2.x provides a `buckets API`_ to programatically interact with buckets.

In the current EFD implementation, telemetry and events from the Observatory are being recorded into a single EFD database, the equivalent to a bucket in InfluxDB OSS 1.x.

In Sasquatch, we are considering storing telemetry and events into separate buckets.
In particular, because the time difference between events is not regular, events need to be stored with higher time precision than telemetry and metrics to avoid losing data.

.. _bucket: https://docs.influxdata.com/influxdb/latest/organizations/buckets/
.. _buckets API: https://docs.influxdata.com/influxdb/latest/api/#tag/Buckets

Mapping Kafka topics to connector instances and buckets
-------------------------------------------------------

When using the Strimzi Kafka bridge it makes sense to have a 1:1 mapping between Kafka topics, connector instances and buckets.

For example, a ``faro`` topic in Kafka would hold the ``lsst.verify`` job messages produced by ``lsst.faro``.
A ``faro`` InfluxDB connector instance would have the configuration to extract the metric values and metadata from those messages, and would write them to a ``faro`` bucket in InfluxDB.

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

Phase 1 - Replace EFD deployments
---------------------------------

#. Add Sasquatch to Phalanx.
#. Enable Chronograf authentication through Gafaelfawr.
#. Replace Confluent Kafka with Strimzi Kafka.
#. Automate Strimzi Kafka image builds adding the InfluxDB Sink, Mirror Maker 2, and S3 connectors.
#. Deploy Sasquatch at IDF Dev (side application: monitor JupyterHub metrics).
#. Deploy Sasquatch at NCA Int (test Mirror Maker 2 and S3 connectors).
#. Deploy Sasquatch at NCSA Stable.
#. Migrate EFD data to Sasquatch at NCSA Stable.
#. Add ``csc`` and ``kafka-producer`` subcharts to Sasquatch for end-to-end testing.
#. Deploy Sasquatch at TTS (Pillan cluster).
#. Add SASL configuration to ``ts_salkafka``.
#. Test connectors and integration with CSCs.
#. Deploy Sasquatch at the Base (Antus cluster).
#. Deploy Sasquatch at the Summit (Yagan cluster).
#. Migrate EFD data from the efd-temp-k3s.cp.lsst.org server to Sasquatch at the Summit.

Related goals
^^^^^^^^^^^^^

#. Archive argocd-efd deployment repo.
#. Remove EFD related charts from the SQuaRE charts repo.
#. Remove efd-temp-k3s.cp.lsst.org.

Phase 2 - Replace the SQuaSH deployment
---------------------------------------

#. Implement Strimzi Kafka bridge as a replacement for the SQuaSH API in Sasquatch.
#. Configure InfluxDB Sink connector to parse ``lsst.verify`` job messages.
#. Implement two-way replication in Sasquatch.
#. Deploy Sasquatch on IDF int.
#. Deploy Sasquatch on IDF prod.
#. Migrate SQuaSH data to Sasquatch at IDF (or USDF).

Related goals
^^^^^^^^^^^^^

#. Remove squash and influxdb-demo clusters on Google


Phase 3 - Migration to InfluxDB OSS 2.x
---------------------------------------

#. Add InfluxDB OSS 2.x to Sasquatch deployment.
#. Test InfluxDB Sink connector with InfluxDB OSS 2.x.
#. Migrate EFD database to 2.x format (TTS, Base, Summit, NCSA Int, NCSA Stable).
#. Exercise InfluxDB OSS 2.x backup/restore tools.
#. Connect Chronograf with InfluxDB OSS 2.x (rquires DBRP mapping).
#. Migrate Kapacitor alerts to Flux tasks.
#. Migrate Chronograf 1.x annotations (``_chronograf`` database) to InfluxDB 2.x.
#. Upgrage EFD client to use the InfluxDB OSS 2.x Python client.


.. rubric:: References
..
.. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
    :style: lsst_aa
