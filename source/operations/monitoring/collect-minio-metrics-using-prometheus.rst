.. _minio-metrics-collect-using-prometheus:
.. _minio-metrics-and-alerts:

======================================
Collect MinIO Metrics Using Prometheus
======================================

.. default-domain:: minio

.. contents:: Table of Contents
   :local:
   :depth: 1

MinIO leverages `Prometheus <https://prometheus.io/>`__ for metrics and alerts.
MinIO publishes Prometheus-compatible scraping endpoints for cluster and
node-level metrics. See :ref:`minio-metrics-and-alerts-endpoints` for more
information.

The procedure on this page documents scraping the MinIO metrics
endpoints using a Prometheus instance, including deploying and configuring
a simple Prometheus server for collecting metrics. 

This procedure is not a replacement for the official
:prometheus-docs:`Prometheus Documentation <>`. Any specific guidance
related to configuring, deploying, and using Prometheus is made on a best-effort
basis.

Requirements
------------

Install and Configure ``mc`` with Access to the MinIO Cluster
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This procedure uses :mc:`mc` for performing operations on the MinIO
deployment. Install ``mc`` on a machine with network access to the
deployment. See the ``mc`` :ref:`Installation Quickstart <mc-install>` for
more complete instructions.

Prometheus Service
~~~~~~~~~~~~~~~~~~

This procedure provides instruction for deploying Prometheus for rapid local
evaluation and development. All other environments should have an existing
Prometheus or Prometheus-compatible service with access to the MinIO cluster. 

Procedure
---------

1) Generate the Bearer Token
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

MinIO by default requires authentication for requests made to the metrics
endpoints. While step is not required for MinIO deployments started with 
:envvar:`MINIO_PROMETHEUS_AUTH_TYPE` set to ``"public"``, you can still use the
command output for retrieving a Prometheus ``scrape_configs`` entry.

Use the :mc-cmd:`mc admin prometheus generate` command to generate a
JWT bearer token for use by Prometheus in making authenticated scraping
requests:

.. code-block:: shell
   :class: copyable

   mc admin prometheus generate ALIAS

Replace :mc-cmd:`ALIAS <mc admin prometheus generate TARGET>` with the
:mc:`alias <mc alias>` of the MinIO deployment.

The command returns output similar to the following:

.. code-block:: yaml
   :class: copyable

   scrape_configs:
   - job_name: minio-job
     bearer_token: TOKEN
     metrics_path: /minio/v2/metrics/cluster
     scheme: https
     static_configs:
     - targets: [minio.example.net]

The ``targets`` array can contain the hostname for any node in the deployment.
For clusters with a load balancer managing connections between MinIO nodes,
specify the address of the load balancer.

Specify the output block to the 
:prometheus-docs:`scrape_config 
<prometheus/latest/configuration/configuration/#scrape_config>` section of
the Prometheus configuration. 

2) Configure and Run Prometheus
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Follow the Prometheus :prometheus-docs:`Getting Started 
<prometheus/latest/getting_started/#downloading-and-running-prometheus>` guide
to download and run Prometheus locally.

Append the ``scrape_configs`` job generated in the previous step to the
configuration file:

.. code-block:: yaml
   :class: copyable

   global:
      scrape_interval: 15s
   
   scrape_configs:
      - job_name: minio-job
        bearer_token: TOKEN
        metrics_path: /minio/v2/metrics/cluster
        scheme: https
        static_configs:
        - targets: [minio.example.net]

Start the Prometheus cluster using the configuration file:

.. code-block:: shell
   :class: copyable

   prometheus --config.file=prometheus.yaml

3) Analyze Collected Metrics
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Prometheus includes a 
:prometheus-docs:`expression browser 
<prometheus/latest/getting_started/#using-the-expression-browser>`. You can
execute queries here to analyze the collected metrics.

The following query examples return metrics collected by Prometheus:

.. code-block:: shell
   :class: copyable

   minio_cluster_disk_online_total{job="minio-job"}[5m]
   minio_cluster_disk_offline_total{job="minio-job"}[5m]
   
   minio_bucket_usage_object_total{job="minio-job"}[5m]

   minio_cluster_capacity_usable_free_bytes{job="minio-job"}[5m]

See :ref:`minio-metrics-and-alerts-available-metrics` for a complete
list of published metrics.

4) Visualize Collected Metrics
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The :minio-git:`MinIO Console <console>` supports visualizing collected metrics
from Prometheus. Specify the URL of the Prometheus service to the
:envvar:`MINIO_PROMETHEUS_URL` environment variable to each MinIO server
in the deployment:

.. code-block:: shell
   :class: copyable

   export MINIO_PROMETHEUS_URL="https://prometheus.example.net"

If you set a custom ``job_name`` for the Prometheus scraping job, you must also
set :envvar:`MINIO_PROMETHEUS_JOB_ID` to match that job name.

Restart the deployment using :mc-cmd:`mc admin service restart` to apply the
changes.

The MinIO Console uses the metrics collected by the ``minio-job`` scraping
job to populate the Dashboard metrics:

.. image:: /images/minio-console/console-metrics.png
   :width: 600px
   :alt: MinIO Console Dashboard displaying Monitoring Data
   :align: center

MinIO also publishes a `Grafana Dashboard
<https://grafana.com/grafana/dashboards/13502>`_ for visualizing collected
metrics. For more complete documentation on configuring a Prometheus data source
for Grafana, see :prometheus-docs:`Grafana Support for Prometheus
<visualization/grafana/>`.

Prometheus includes a :prometheus-docs:`graphing interface
<prometheus/latest/getting_started/#using-the-graphing-interface>` for
visualizing collected metrics. 

.. _minio-metrics-and-alerts-endpoints:

Metrics
-------

MinIO provides a scraping endpoint for cluster-level metrics:

.. code-block:: shell
   :class: copyable

   http://minio.example.net:9000/minio/v2/metrics/cluster

Replace ``http://minio.example.net`` with the hostname of any node in the MinIO
deployment. For deployments with a load balancer managing connections between
MinIO nodes, specify the address of the load balancer.

Create a new :prometheus-docs:`scraping configuration
<prometheus/latest/configuration/configuration/#scrape_config>` to begin
collecting metrics from the MinIO deployment. See
:ref:`minio-metrics-collect-using-prometheus` for a complete tutorial.

The following example describes a ``scrape_configs`` entry for collecting
cluster metrics. 

.. code-block:: yaml
   :class: copyable

   scrape_configs:
   - job_name: minio-job
     bearer_token: <secret>
     metrics_path: /minio/v2/metrics/cluster
     scheme: https
     static_configs:
     - targets: ['minio.example.net:9000']

.. list-table::
   :stub-columns: 1
   :widths: 20 80
   :width: 100%

   * - ``job_name``
     - The name of the scraping job.

   * - ``bearer_token``
     - The JWT token generated by :mc-cmd:`mc admin prometheus generate`.

       Omit this field if the MinIO deployment was started with
       :envvar:`MINIO_PROMETHEUS_AUTH_TYPE` set to ``public``.

   * - ``targets``
     - The endpoint for the MinIO deployment. You can specify any node in the
       deployment for collecting cluster metrics. For clusters with a load
       balancer managing connections between MinIO nodes, specify the
       address of the load balancer.

MinIO by default requires authentication for scraping the metrics endpoints.
Use the :mc-cmd:`mc admin prometheus generate` command to generate the
necessary bearer tokens for use with configuring the
``scrape_configs.bearer_token`` field. You can alternatively disable
metrics endpoint authentication by setting
:envvar:`MINIO_PROMETHEUS_AUTH_TYPE` to ``public``.

Visualizing Metrics
~~~~~~~~~~~~~~~~~~~

The MinIO Console uses the metrics collected by Prometheus to populate the
Dashboard metrics:

.. image:: /images/minio-console/console-metrics.png
   :width: 600px
   :alt: MinIO Console displaying Prometheus-backed Monitoring Data
   :align: center

Set the :envvar:`MINIO_PROMETHEUS_URL` environment variable to the URL of the
Prometheus service to allow the Console to retrieve and display collected
metrics. See :ref:`minio-metrics-collect-using-prometheus` for a complete
example.

MinIO also publishes a `Grafana Dashboard
<https://grafana.com/grafana/dashboards/13502>`_ for visualizing collected
metrics. For more complete documentation on configuring a Prometheus data source
for Grafana, see :prometheus-docs:`Grafana Support for Prometheus
<visualization/grafana/>`.

.. _minio-metrics-and-alerts-available-metrics:

Available Metrics
~~~~~~~~~~~~~~~~~

MinIO publishes the following metrics, where each metric includes a label for
the MinIO server which generated that metric.

Object Metrics
++++++++++++++

.. metric:: minio_bucket_objects_size_distribution

   Distribution of object sizes in the bucket, includes label for the bucket 
   name.

Replication Metrics
+++++++++++++++++++

These metrics are only populated for MinIO clusters with 
:ref:`minio-bucket-replication-serverside` enabled.

.. metric:: minio_bucket_replication_failed_bytes

   Total number of bytes failed at least once to replicate.

.. metric:: minio_bucket_replication_pending_bytes

   Total bytes pending to replicate.

.. metric:: minio_bucket_replication_received_bytes

   Total number of bytes replicated to this bucket from another source bucket.

.. metric:: minio_bucket_replication_sent_bytes

   Total number of bytes replicated to the target bucket.

.. metric:: minio_bucket_replication_pending_count

   Total number of replication operations pending for this bucket.

.. metric:: minio_bucket_replication_failed_count

   Total number of replication operations failed for this bucket.

Bucket Metrics
++++++++++++++

.. metric:: minio_bucket_usage_object_total

   Total number of objects

.. metric:: minio_bucket_usage_total_bytes

   Total bucket size in bytes

Cache Metrics
+++++++++++++

.. metric:: minio_cache_hits_total

   Total number of disk cache hits

.. metric:: minio_cache_missed_total

   Total number of disk cache misses

.. metric:: minio_cache_sent_bytes

   Total number of bytes served from cache

.. metric:: minio_cache_total_bytes

   Total size of cache disk in bytes

.. metric:: minio_cache_usage_info

   Total percentage cache usage, value of 1 indicates high and 0 low, label
   level is set as well

.. metric:: minio_cache_used_bytes

   Current cache usage in bytes

Cluster Metrics
+++++++++++++++

.. metric:: minio_cluster_capacity_raw_free_bytes

   Total free capacity online in the cluster.

.. metric:: minio_cluster_capacity_raw_total_bytes

   Total capacity online in the cluster.

.. metric:: minio_cluster_capacity_usable_free_bytes

   Total free usable capacity online in the cluster.

.. metric:: minio_cluster_capacity_usable_total_bytes

   Total usable capacity online in the cluster.

Node Metrics
++++++++++++

.. metric:: minio_cluster_nodes_offline_total

   Total number of MinIO nodes offline.

.. metric:: minio_cluster_nodes_online_total

   Total number of MinIO nodes online.

.. metric:: minio_heal_objects_error_total

   Objects for which healing failed in current self healing run

.. metric:: minio_heal_objects_heal_total

   Objects healed in current self healing run

.. metric:: minio_heal_objects_total

   Objects scanned in current self healing run

.. metric:: minio_heal_time_last_activity_nano_seconds

   Time elapsed (in nano seconds) since last self healing activity. This is set
   to -1 until initial self heal

.. metric:: minio_inter_node_traffic_received_bytes

   Total number of bytes received from other peer nodes.

.. metric:: minio_inter_node_traffic_sent_bytes

   Total number of bytes sent to the other peer nodes.

.. metric:: minio_node_disk_free_bytes

   Total storage available on a disk.

.. metric:: minio_node_disk_total_bytes

   Total storage on a disk.

.. metric:: minio_node_disk_used_bytes

   Total storage used on a disk.

.. metric:: minio_node_file_descriptor_limit_total

   Limit on total number of open file descriptors for the MinIO Server process.

.. metric:: minio_node_file_descriptor_open_total

   Total number of open file descriptors by the MinIO Server process.

.. metric:: minio_node_io_rchar_bytes

   Total bytes read by the process from the underlying storage system including
   cache, ``/proc/[pid]/io rchar``

.. metric:: minio_node_io_read_bytes

   Total bytes read by the process from the underlying storage system, 
   ``/proc/[pid]/io read_bytes``

.. metric:: minio_node_io_wchar_bytes

   Total bytes written by the process to the underlying storage system including 
   page cache, ``/proc/[pid]/io wchar``

.. metric:: minio_node_io_write_bytes

   Total bytes written by the process to the underlying storage system, 
   ``/proc/[pid]/io write_bytes``

.. metric:: minio_node_process_starttime_seconds

   Start time for MinIO process per node, time in seconds since Unix epoch.

.. metric:: minio_node_process_uptime_seconds

   Uptime for MinIO process per node in seconds.

.. metric:: minio_node_scanner_bucket_scans_finished

   Total number of bucket scans finished since server start.

.. metric:: minio_node_scanner_bucket_scans_started

   Total number of bucket scans started since server start.

.. metric:: minio_node_scanner_directories_scanned

   Total number of directories scanned since server start.

.. metric:: minio_node_scanner_objects_scanned

   Total number of unique objects scanned since server start.

.. metric:: minio_node_scanner_versions_scanned

   Total number of object versions scanned since server start.

.. metric:: minio_node_syscall_read_total

   Total read SysCalls to the kernel. ``/proc/[pid]/io syscr``

.. metric:: minio_node_syscall_write_total

   Total write SysCalls to the kernel. ``/proc/[pid]/io syscw``

S3 Metrics
++++++++++

.. metric:: minio_s3_requests_error_total

   Total number S3 requests with errors

.. metric:: minio_s3_requests_inflight_total

   Total number of S3 requests currently in flight

.. metric:: minio_s3_requests_total

   Total number S3 requests

.. metric:: minio_s3_time_ttbf_seconds_distribution

   Distribution of the time to first byte across API calls.

.. metric:: minio_s3_traffic_received_bytes

   Total number of s3 bytes received.

.. metric:: minio_s3_traffic_sent_bytes

   Total number of s3 bytes sent

Software Metrics
++++++++++++++++

.. metric:: minio_software_commit_info

   Git commit hash for the MinIO release.

.. metric:: minio_software_version_info

   MinIO Release tag for the server

.. _minio-metrics-and-alerts-alerting:

Alerts
------

You can configure alerts using Prometheus :prometheus-docs:`Alerting Rules
<prometheus/latest/configuration/alerting_rules/>` based on the collected MinIO
metrics. The Prometheus :prometheus-docs:`Alert Manager
<alerting/latest/overview/>` supports managing alerts produced by the configured
alerting rules. Prometheus also supports a :prometheus-docs:`Webhook Receiver
<operating/integrations/#alertmanager-webhook-receiver>` for publishing alerts
to mechanisms not supported by Prometheus AlertManager.