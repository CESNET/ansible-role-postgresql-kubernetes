# cesnet.postgresql_kubernetes

Ansible role for creating a high-availability [PostgreSQL](https://www.postgresql.org/) cluster in [Kubernetes](https://kubernetes.io/)
using [CloudNativePG](https://cloudnative-pg.io/documentation/current/) Operator.
The cluster has a single database, two users (one as the database owner, second named pgwatch for monitoring),
creates continuous backups in an S3 bucket, and is accessible from outside from selected IP addresses.

Requirements
------------

The role uses Ansible modules
- [kubernetes.core.helm](https://docs.ansible.com/ansible/latest/collections/kubernetes/core/helm_module.html)
- [amazon.aws.s3_bucket](https://docs.ansible.com/ansible/latest/collections/amazon/aws/s3_bucket_module.html)

running on localhost, check their dependencies.

Variables must be provided that match settings in ```~/.kube/config``` file:
- **k8s_namespace** — Kubernetes namespace
- **k8s_context** — Kubernetes context

Role Variables
--------------
- **cnpg_cluster_name** — name of Helm App and prefix for all Kubernetes objects related to this cluster (default cnpgdb)
- **cnpg_cluster_external_hostname** — external DNS hostname used for accessing the database (default {{ cnpg_cluster_name }}.{{ k8s_namespace }}.dyn.cloud.e-infra.cz)
- **cnpg_cluster_image** — container image name for cluster nodes, see [CNPG PostgreSQL Container Images](https://github.com/cloudnative-pg/postgres-containers/pkgs/container/postgresql) and [tagged versions](https://github.com/cloudnative-pg/postgres-containers/pkgs/container/postgresql/versions?filters%5Bversion_type%5D=tagged)
- **cnpg_cluster_db_name** — name of database, default is "app"
- **cnpg_cluster_db_owner** — database owner user, default is "app"
- **cnpg_cluster_db_parameters** — options for [createdb](https://www.postgresql.org/docs/current/app-createdb.html#id-1.9.4.4.6), like ```encoding```, ```localeCType```, ```localeCollate```
- **cnpg_cluster_postgresql_timezone** — timezone for server, affects SQL type [timestamp with timezone](https://www.postgresql.org/docs/current/datatype-datetime.html#DATATYPE-TIMEZONES), default is "Europe/Prague", use "Etc/UTC" for UTC
- **cnpg_cluster_postgresql_parameters** - options for [postgresql.conf](https://cloudnative-pg.io/documentation/current/postgresql_conf/#the-postgresql-section)
- **cnpg_cluster_enable_superuser_access** — whether to enable access to user postgres from outside, default is true 
- **cnpg_cluster_user_password** — password of db owner
- **cnpg_cluster_pgwatch_user_password** — password for pgwatch user
- **cnpg_cluster_storage_size** — store allocated on local disk for each replica
- **cnpg_cluster_wal_keep_size** — size for Write-Ahead-Logs files
- **cnpg_cluster_requests_mem** — memory requested for each replica
- **cnpg_cluster_requests_cpu** — CPU requested for each replica
- **cnpg_cluster_limits_mem** — memory limit for each replica
- **cnpg_cluster_limits_cpu** —  CPU limit for each replica
- **cnpg_sidecar_requests_mem** — [sidecar container](https://cloudnative-pg.io/plugin-barman-cloud/docs/usage/#configuring-the-plugin-instance-sidecar) with Barman backup plugin memory requested
- **cnpg_sidecar_requests_cpu** — sidecar container with Barman backup plugin cpu requested
- **cnpg_sidecar_limits_mem** — sidecar container with Barman backup plugin memory limit 
- **cnpg_sidecar_limits_cpu** — sidecar container with Barman backup plugin CPU limit
- **cnpg_cluster_retention_policy** — [retention policy](https://cloudnative-pg.io/plugin-barman-cloud/docs/retention/) of backup files in S3
- **cnpg_cluster_instances** — number of servers, at least 1 for a single master, recommended is at least 3
- **cnpg_cluster_firewall_ports** — ports to open in firewall, default is CESNET and MUNI VPNs
- **cnpg_cluster_firewall_additional_ports** — additional ports to open, default is empty 
- **cnpg_cluster_s3_bucket** — name of S3 bucket for backups (full backups and WAL), must be initially empty
- **cnpg_cluster_aws_access_key_id** — S3 access key id
- **cnpg_cluster_aws_secret_access_key** — S3 access key secret
- **cnpg_cluster_aws_endpointURL** — S3 endpointURL
- **cnpg_cluster_full_backup_schedule** — [cron schedule](https://pkg.go.dev/github.com/robfig/cron#hdr-CRON_Expression_Format) for full backups, in UTC timezone
- **cnpg_cluster_backup_wal_compression** — [compression algorithm](https://cloudnative-pg.io/plugin-barman-cloud/docs/compression/) for WAL files (default zstd)
- **cnpg_cluster_backup_data_compression** — [compression algorithm](https://cloudnative-pg.io/plugin-barman-cloud/docs/compression/) for full backups (default snappy)
- **cnpg_cluster_ready_wait_seconds** — how many seconds to wait for cluster to become ready, default is 400 seconds
- **cnpg_cluster_synchronous** — if defined, its content is added as spec/synchronous, see [Synchronous Replication](https://cloudnative-pg.io/documentation/current/replication/#synchronous-replication) 
- **cnpg_cluster_recovery** — if true, do a recovery from an S3 bucket specified by the `cnpg_cluster_s3_recovery_bucket` variable, default is false
- **cnpg_cluster_s3_recovery_bucket** — the name of the S3 bucket with backup files, default is undefined 

Example
-------
```yaml
- name: "installs CNPG cluster in Kubernetes"
  hosts:
    - my-ns
  connection: local
  gather_facts: no
  roles:
    - role: cesnet.postgresql_kubernetes
      vars:
        k8s_namespace: my-ns
        k8s_context: my-context
        cnpg_cluster_name: mydbcluster
        cnpg_cluster_db_name: mydb
        cnpg_cluster_db_owner: myuser
        cnpg_cluster_db_parameters:
          encoding: UTF8
          localeCType: en_US.UTF-8
          localeCollate: en_US.UTF-8
        cnpg_cluster_postgresql_parameters:
          max_connections: 200
          max_pred_locks_per_transaction: 8192
          shared_buffers: 256MB
        cnpg_cluster_storage_size: 20Gi
        cnpg_cluster_synchronous:
          method: any
          number: 1
          dataDurability: preferred
        cnpg_cluster_image: ghcr.io/cloudnative-pg/postgresql:18.1-standard-trixie
        cnpg_cluster_s3_bucket: mydb-backups
        cnpg_cluster_user_password: "{{ user_password_vault }}"
        cnpg_cluster_pgwatch_user_password: "{{ pgwatch_user_password_vault }}"
        cnpg_cluster_aws_access_key_id: "{{ aws_access_key_id_vault }}"
        cnpg_cluster_aws_secret_access_key: "{{ aws_secret_access_key_vault }}"
```

Database recovery
-----------------
The database cluster stores backups (both full backups once a day and WAL files every 5 minutes) in an S3 bucket
with the name defined in the `cnpg_cluster_s3_bucket` variable. If you need to recover a lost cluster from the backup,
create a playbook like this:
```yaml
- name: "recover CNPG cluster from an S3 backup"
  hosts:
    - my-ns
  connection: local
  gather_facts: no
  vars:
    cnpg_cluster_recovery: true
    cnpg_cluster_s3_recovery_bucket: mydbbackups
    cnpg_cluster_s3_bucket: mydbbackups2
  tasks:
    - import_role:
        name: cesnet.postgresql_kubernetes
```
The variable `cnpg_cluster_s3_recovery_bucket` must contain the name of the S3 bucket containing the backups
to recover from, and the variable `cnpg_cluster_s3_bucket` must contain the name of a non-existing or empty bucket 
where new logs will be written after the recovery. You cannot reuse the old bucket. 

After the cluster is recovered, you can run the original playbook again to replace the recovery settings with normal
settings, just do not forget to change the `cnpg_cluster_s3_bucket` to the new bucket.

Backup compression
------------------
Full backups can be compressed by three algorithms only. A 10GB backup was compressed to:

| algorithm | compress time | decompress time | size | ratio |
|:---------:|:-------------:|:---------------:|:----:|:-----:|
|     bzip2 |         422 s |           188 s | 2.1G | 4.7:1 |
|      gzip |         157 s |            46 s | 2.6G | 3.8:1 |
|    snappy |          15 s |            15 s | 3.5G | 2.8:1 |

WAL files have wider selection of algorithms. An uncompressed WAL file of the size 16777216 bytes was compressed to: 

| algorithm |  time  |  size  | ratio  |
|:---------:|:------:|:------:|--------|
|    snappy | 0.008s | 792065 | 21:1   |
|       lz4 | 0.019s |  71331 | 235:1  |
|      gzip | 0.064s |  19999 | 838:1  |
|        xz | 0.217s |   4440 | 3778:1 |
|      zstd | 0.021s |   3702 | 4531:1 |
|  zstd -10 | 0.044s |   3407 | 4924:1 |
|     bzip2 | 0.100s |   2907 | 5771:1 |

The default algorithms were selected to be snappy and zstd because of fast compression times with reasonable compression ratio.