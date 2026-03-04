#######################
EFD Schema Improvements
#######################

.. abstract::

   Describe current problems with the EFD schema design, discuss schema design changes and implementation plan.

The "packed data" problem
=========================

"Packed data" refers to the practice of storing multiple pieces of information in a single field in InfluxDB, which makes it difficult to query and lead to performance issues.

"Packed data" is present in several telemetry topics in the EFD, but it is most evident in the SAL MTCamera telemetry due to the highly hierarchical structure of the LSSTCam and its large number of elements.

We describe the "packed data" problem in detail for the SAL MTCamera telemetry, the implications of querying the data in this format and database compaction errors caused by this design choice.
We show how the schema redesign implemented by the Camera Control System (CCS) team solved this problem.
We discuss the implications of stop publishing and recording the MTCamera telemetry in the old SAL format while keeping the new CCS MTCamera format only.

Finally, we identify other examples of "packed data" in the EFD and outline a plan to solve this problem for other subsystems in the EFD.

Case study: Camera telemetry
============================

SAL MTCamera schema  
-------------------

"Packed data" is present in 19 MTCamera telemetry topics, which are published by the SAL and recorded in the EFD.

A typical example of "packed data" can be found in the ``lsst.sal.MTCamera.focal_plane_Segment`` topic.

The camera ``Raft``, ``Sensor`` and ``Segment`` information is encoded in the ``location`` field and the electronic currents (mA) for each Segment are stored in an array of 3024 values in the ``i`` field.

.. code:: 

   {
      <SAL private fields>, 
      "i": [ ... 3024 values ... ],
      "location": "R01S00Seg00:R01S00Seg01:R01S00Seg02:R01S00Seg03:R01S00Seg04:R01S00Seg05..."
    }

Kafdrop can be used to inspect `lsst.sal.MTCamera.focal_plane_Segment`_ messages in this format.

This schema design leads to large messages making it difficult to tune Kafka producers and consumers performance through parameters like batch size, since "packed data" is already batched in a single message.

After recorded in InfluxDB, these are examples of queries to retrieve the location information and electronic currents in the focal plane:

.. code:: SQL

   SELECT "location" FROM "efd"."autogen"."lsst.sal.MTCamera.focal_plane_Segment" LIMIT 1
   
   name: lsst.sal.MTCamera.focal_plane_Segment
   time                location
   ----                --------
   1749067202294455000 R01S00Seg00:R01S00Seg01:R01S00Seg02:R01S00Seg03:R01S00Seg04:R01S00Seg05...

and

.. code:: SQL
   
   SELECT /i/ FROM "efd"."autogen"."lsst.sal.MTCamera.focal_plane_Segment" LIMIT 1
   
   name: lsst.sal.MTCamera.focal_plane_Segment
   time                i0 ... i3023
   ----                -
   1749067202294455000 i0 ... i3023


The output of these queries includes the location information and electric currents for all the Segments in the focal plane (16 Segments per Sensor * 9 Sensors per Raft * 21 Rafts = 3024 Segments) for each timestamp. 
This particular topic is published at 1Hz.

With this schema design it is not possible to query the data for a specific Raft, Sensor or Segment without retrieving the entire array of 3024 locations and values and then "unpacking" them.
Consequently, it is not possible to visualize this data in Chronograf.

We also found that this design choice leads to database compaction errors, caused by the large strings stored in the location field, resulting in slow queries for other topics in the same shard.
See details in DM-52368_.

.. code:: bash

   compact-shard failed: compaction failed: block read error on 
   /var/lib/influxdb/data/efd/autogen/6752/000062200-000000179.tsm: 
   encode error: unable to compress block type string for key 
   'lsst.sal.MTCamera.focal_plane_Segment#!~#location': 
   StringArrayEncodeAll: source length too large


CCS MTCamera schema redesign 
----------------------------

The Camera Control System (CCS) team has redesigned the schema for the MTCamera telemetry topics to address the "packed data" problem.

For ``lsst.MTCamera.focal_plane_Segment`` telemetry, the messages add new fields for the Raft, Reb, Sensor, Segment and Agent information and store single electronic current values for each segment in the ``I`` field.

.. code::

   {
      "timestamp": 1772003223720,
      "I": 1.173095703125,
      "I_state": "NOMINAL",
      "Raft": "R03",
      "Reb": "Reb2",
      "Sensor": "S20",
      "Segment": "Seg11",
      "Agent": "focal-plane"
   }

Example of messages in this format can be found in Kafdrop for the `lsst.MTCamera.focal_plane_Segment`_ topic.

In InfluxDB the ``Raft``, ``Reb``, ``Sensor``, ``Segment``, and ``Agent`` information are now tags to store and query this data more efficiently.

In the improved schema, a query to retrieve the electric current for a specific Segment looks like this:

.. code:: SQL

   SELECT "I" FROM "lsst.MTCamera.focal_plane_Segment" WHERE "Raft"='R03' AND "Sensor"='S20' AND "Segment"='Seg11' LIMIT 1

   name: lsst.MTCamera.focal_plane_Segment
   time                I
   ----                -
   1772003223720       1.173095703125

There are several benefits with the new schema design:

- Smaller messages are more efficient in Kafka.
- The use of tags in InfluxDB allows for more efficient storage and querying of the data.
- Special code to "unpack" the data is not necessary. 
- It is possible to query and visualize the data in Chronograf.


How to move forward with Camera telemetry?
------------------------------------------

If SAL MTCamera telemetry is not needed for any use case, we propose to stop publishing it and keep only the new CCS MTCamera telemetry in both Kafka and the EFD.
If this is not possible, we should at least stop recording the old SAL MTCamera telemetry in the EFD. 

Option 1: stop publishing the old SAL MTCamera telemetry
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This is the preferred option since we are publishing and recording MTCamera telemetry twice in the old SAL format and in the new CCS format. 
By stop publishing the old SAL MTCamera telemetry we also remove the duplicated topics and schemas in Kafka.

This will fix the data duplication and avoid potential database compaction errors caused by the long strings of "packed data" in some of the MTCamera telemetry topics.

Option 2: keep publishing the old SAL MTCamera telemetry while stop recording it in the EFD
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This alleviates the problem of data duplication in the EFD and avoid potential database compaction errors.
However we still have duplicated topics and schemas in Kafka.


.. note::

   In any case, MTCamera commands and events topics are not affected and will continue to be published and recorded in the SAL format.
   

Keeping the MTCamera telemetry on its own database
--------------------------------------------------

Currently CCS MTCamera telemetry is recorded in a separate database in InfluxDB, ``lsst.MTCamera``.
We propose keeping the MTCamera telemetry on its own database, since the schema design is different from the rest of the EFD telemetry published by SAL.
(e.g. the schema is not on ``ts_xml``, and it doesn't have the SAL private fields).

Users are already familiar with how to `query Camera telemetry with the EFD client`_.

Other examples of "packed data" in the EFD
==========================================

A non exhaustive search of the EFD telemetry topics shows that "packed data" is present in other subsystems such as ATMCS, MTM1M3, MTM2, ESS and MTMount.

ATMCS telemetry 
---------------

ATMCS telemetry is produced by cRIO controllers which send arrays of 100 values in each message.

See for example the `lsst.sal.ATMCS.azEl_mountMotorEncoders`_ topic in Kafdrop.

This data is unpacked with special code in the EFD client such as `select_packed_time_series()`.

In total ATMCS telemetry has 7 topics with "packed data": 

.. code::

   lsst.sal.ATMCS.azEl_mountMotorEncoders
   lsst.sal.ATMCS.measuredMotorVelocity
   lsst.sal.ATMCS.measuredTorque
   lsst.sal.ATMCS.mount_AzEl_Encoders
   lsst.sal.ATMCS.mount_Nasmyth_Encoders
   lsst.sal.ATMCS.nasmyth_m3_mountMotorEncoders
   lsst.sal.ATMCS.nasymth_m3_mountMotorEncoders
   lsst.sal.ATMCS.torqueDemand
   lsst.sal.ATMCS.trajectory

MTM1M3 telemetry
----------------

Telemetry for the MTM1M3 cell actuators are packed in arrays of 12, 100, and 157 values.

See for example the `lsst.sal.MTM1M3.appliedForces`_ topic in Kafdrop.

MTM2 telemetry
--------------

Telemetry for the MTM2 cell actuators are packed in arrays of 72 values.

See for example the `lsst.sal.MTM2.axialForce`_ topic in Kafdrop.

ESS telemetry
-------------

Multiple ESS telemetry topics have "packed data", for example the `lsst.sal.ESS.accelerometer`_ topic has fields with arrays of 400 values.

MTMount telemetry
-----------------

Multiple MTMount telemetry topics have "packed data", for example the `lsst.sal.MTMount.balancing`_ topic seems to pack 4 timestamps and 4 values for each field.


Moving forward
==============

To solve the "packed data" problem in the EFD, we propose an incremental approach starting with the subsystems with the most critical query problems or performance issues.

From the experience with the MTCamera telemetry, we know that solving the "packed data" problem requires a schema redesign and changes in the Kafka producers to publish the data in the new format.
This is a significant effort that might not be feasible during the pre-operations phase of the project where we are seeking stability of the Control System software.

A less invasive option would consist in improving the EFD schema design without changing the Avro schemas or the Kafka producers.
This would require changes at the Telegraf connector level to transform the data from the "packed data" format to a more efficient format before recording it in InfluxDB.
A drawback of this option is that the EFD and the Avro schemas would differ.
Currently, the Avro schemas are the source of schema information for the EFD such as field descriptions and units.

Following the MTCamera telemetry example, we recommend keeping the new telemetry format for each subsystem in its own database in InfluxDB.

We also recommend this work to be done in collaboration with the corresponding teams responsible for each subsystem and users of the EFD.
Rubin Summit Operations Diagnosis & Analysis Group is a good candidate to lead this effort for the ATMCS, MTM1M3, MTM2, ESS, MTMount and potentially other subsystems.


.. _DM-52368: https://rubinobs.atlassian.net/browse/DM-52368
.. _lsst.sal.MTCamera.focal_plane_Segment: https://usdf-rsp.slac.stanford.edu/kafdrop-remote/topic/summit.lsst.sal.MTCamera.focal_plane_Segment/messages
.. _lsst.MTCamera.focal_plane_Segment: https://usdf-rsp.slac.stanford.edu/kafdrop-remote/topic/summit.lsst.MTCamera.focal_plane_Segment/messages
.. _query Camera telemetry with the EFD client: https://efd-client.lsst.io/user-guide/camera-telemetry.html
.. _lsst.sal.ATMCS.azEl_mountMotorEncoders: https://usdf-rsp.slac.stanford.edu/kafdrop-remote/topic/summit.lsst.sal.ATMCS.azEl_mountMotorEncoders/messages
.. _lsst.sal.MTM1M3.appliedForces: https://usdf-rsp.slac.stanford.edu/kafdrop-remote/topic/summit.lsst.sal.MTM1M3.appliedForces/messages
.. _lsst.sal.MTM2.axialForce: https://usdf-rsp.slac.stanford.edu/kafdrop-remote/topic/summit.lsst.sal.MTM2.axialForce/messages
.. _lsst.sal.ESS.accelerometer: https://usdf-rsp.slac.stanford.edu/kafdrop-remote/topic/summit.lsst.sal.ESS.accelerometer/messages
.. _lsst.sal.MTMount.balancing: https://usdf-rsp.slac.stanford.edu/kafdrop-remote/topic/summit.lsst.sal.MTMount.balancing/messages
