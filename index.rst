:tocdepth: 1

.. sectnum::

.. Metadata such as the title, authors, and description are set in metadata.yaml

.. TODO: Delete the note below before merging new content to the main branch.


Abstract
========

Sasquatch is a service for recording, displaying, and alerting on Rubin's engineering data.

It is a unification of SQuaSH :cite:`SQR-009` used for tracking science performance metrics and the Engineering and Facilities Database (EFD) :cite:`SQR-034` used to record observatory telemetry data.

The main features include two-way data replication between the Summit and USDF to record science performance metrics computed at the USDF and observatory telemetry and events produced at the Summit in the same place.

It can be easily extended to record other `time-series data`_ such as camera diagnostic metrics, rapid analysis metrics, scheduler events etc.

In its third generation, we took the opportunity to rebrand the service to Sasquatch.
Sasquatch is the `service` that manages the EFD and other time-series `databases`.

Sasquatch is currently deployed at the test stands, Summit, and USDF through `Phalanx`_ integrated with Rubin's Science Platform.

.. _time-series data: https://www.influxdata.com/what-is-time-series-data
.. _Phalanx: https://phalanx.lsst.io

Overview
========

Sasquatch architecture is based on `InfluxDB`_, an open-source time-series database optimized for efficient storage and analysis of time series data, and `Apache Kafka`_ which is used as a write-ahead log to InfluxDB and for data replication between sites.

`InfluxDB OSS 2.x`_  introduces Flux as its native language combining querying and data analysis functionalities.
This version also has a new `task engine`_ to process time-series data with Flux, and a new `Python client`_.

Apache Kafka is now deployed with `Strimzi`_, a Kubernetes operator to manage the Kafka resources.
It also brings `Kafka bridge`_, a component used in Sasquatch for connecting HTTP-based clients with Kafka.

Figure 1 shows a diagram of the Sasquatch architecutre highlighting the new functionalities: two-way replication between the Summit and USDF; multiple InfluxDB databases; Flux Tasks; and a REST API based on Strimzi Kafka bridge.

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
One is based on the SAL Kafka Producers (`ts_salkafka`_) and the other is based on the Strimzi Kafka Bridge REST API.

`ts_salkafka`_ is currently used with Sasquatch at the Summit and test stands to forward DDS messages to Kafka.
Once DDS is replaced by Kafka, ts_salkafka won't be longer necessary  :cite:`TSTN-033`.

.. _ts_salkafka: https://ts-salkafka.lsst.io

Strimzi Kafka bridge
--------------------

Strimzi Kafka bridge provides a REST interface for connecting HTTP-based clients with Kafka.

With Kafka Bridge a client can send messages to or receive messages from Kafka topics using HTTP requests.
In particular, a client can produce messages to topics in JSON format by using the `topics`_ endpoint.

Once the data lands in Kafka, an InfluxDB Sink connector is responsible for consuming the Kafka topic and writing the data to a bucket in InfluxDB.

In SQuaSH we use the SQuaSH REST API to connect the `lsst.verify`_ client with InfluxDB.
In Sasquatch, the SQuaSH REST API is replaced by the Strimzi Kafka bridge and a new InfluxDB connector.

A new client that needs to send data to Sasquatch would use the same pattern.

In addition, because we send data to Kafka, we can easily replicated data between sites and persist data into other formats like Parquet using off-the-shell Kafka connectors.

.. _topics: https://strimzi.io/docs/bridge/latest/#_send
.. _lsst.verify: https://pipelines.lsst.io/v/daily/modules/lsst.verify


Two-way replication between Summit and USDF
===========================================

In the current EFD implementation, data replication between the Summit and USDF is done throught the Kafka Mirror Maker 2 connector (MM2) :cite:`SQR-050`.

The EFD replication service allows for one-way replication (or active/standby replication) from the Summit to the USDF.
We have measured sub-second latency for a high throughput topic from the MTM1M3 subsystem in that set up.

In Sasquatch, two-way replication (or active/active replication) is now required.
With two-way replication, metrics computed at USDF (e.g. from Prompt Processing), for example, are sent to the USDF instance of Sasquatch and replicated to the Summit.

In addition to the instance of MM2 configured at USDF to replicate Observatory telemetry, events and metrics from the Summit, Sasquatch adds a second instance of MM2 at the Summit.

The Kafka Topics to be replicated are listed in the MM2 configuration on each Kafka cluster.

Two-way replication requires Kafka Topic renaming.
Usually, in this scenario, the Kafka Topic at the destination cluster is prefixed with the name of the source cluster.
That helps to identify its origin and avoid replicating it back to the source cluster.
Consequently, any topic schemas at the destination cluster need to be translated adding more complexity compared to the one-way replication scenario.


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

Mapping Kafka topics to connector instances and buckets
-------------------------------------------------------

When using the Strimzi Kafka bridge it makes sense to map Kafka topics to connector instances and buckets.

For example, the ``analysis_tools`` topic in Kafka holds the ``lsst.verify`` measurements.
The ``analysis_tools`` connector instance is configured to extract the measurements and metadata from Kafka and write them to the ``analysis_tools`` bucket in InfluxDB.

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
As of August 2022 we are completing phase 1 and starting phase 2.

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
#. Deploy Sasquatch at the BTS (Antus cluster).

Related goals
^^^^^^^^^^^^^

#. Archive argocd-efd deployment repo, everything is in Phalanx.
#. Remove EFD related charts from the SQuaRE charts repo.
#. Decomissioning efd-temp-k3s.cp.lsst.org cluster.
#. Migrate USDF deployment from NCSA to SLAC.

Phase 2 - Replace the SQuaSH deployment
---------------------------------------

#. Implement Strimzi Kafka bridge as a replacement for the SQuaSH API in Sasquatch.
#. Configure InfluxDB Sink connector to parse ``lsst.verify`` job messages.
#. Implement two-way replication in Sasquatch.
#. Deploy Sasquatch on IDF int.
#. Migrate SQuaSH data to Sasquatch at USDF.

Related goals
^^^^^^^^^^^^^

#. Remove squash and influxdb-demo clusters on Google


Phase 3 - Migration to InfluxDB OSS 2.x
---------------------------------------

#. Add InfluxDB OSS 2.x to Sasquatch deployment.
#. Test InfluxDB Sink connector with InfluxDB OSS 2.x.
#. Migrate EFD database to 2.x format (TTS, BTS, Summit, USDF).
#. Exercise InfluxDB OSS 2.x backup/restore tools.
#. Connect Chronograf with InfluxDB OSS 2.x (rquires DBRP mapping).
#. Migrate Kapacitor alerts to Flux tasks.
#. Migrate Chronograf 1.x annotations (``_chronograf`` database) to InfluxDB 2.x.
#. Upgrage EFD client to use the InfluxDB OSS 2.x Python client.


.. rubric:: References
..
.. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
    :style: lsst_aa
