# 定制Patroni模板

Pigsty使用Patroni管理与初始化Postgres数据库集群。

Pigsty使用Patroni完成供给的主体工作，即使用户选择了 [无Patroni模式](v-pg-provision#patroni_mode)，拉起数据库集群也会由Patroni负责，并在创建完成后移除Patroni组件。

用户可以通过Patroni配置文件，完成大部分的PostgreSQL集群定制工作，Patroni配置文件格式详情请参考 [**Patroni官方文档**](https://patroni.readthedocs.io/en/latest/SETTINGS.html)。

## 预定义模板

Pigsty提供了四种预定义的初始化模板，初始化模板是用于初始化数据库集群的定义文件，默认位于[`roles/postgres/templates/`](https://github.com/Vonng/pigsty/tree/master/roles/postgres/templates)。包括：

- [`oltp.yml`](https://github.com/Vonng/pigsty/blob/master/roles/postgres/templates/oltp.yml) OLTP模板，默认配置，针对生产机型优化延迟与性能。
- [`olap.yml`](https://github.com/Vonng/pigsty/blob/master/roles/postgres/templates/olap.yml) OLAP模板，提高并行度，针对吞吐量，长查询进行优化。
- [`crit.yml`](https://github.com/Vonng/pigsty/blob/master/roles/postgres/templates/crit.yml)) 核心业务模板，基于OLTP模板针对RPO、安全性、数据完整性进行优化，启用同步复制与数据校验和。
- [`tiny.yml`](https://github.com/Vonng/pigsty/blob/master/roles/postgres/templates/tiny.yml) 微型数据库模板，针对低资源场景进行优化，例如运行于虚拟机中的演示数据库集群。

通过 [`pg_conf`](v-pg-provision.md#pg_conf) 参数指定所需使用的模板路径，如果使用预制模板，则只需填入模板文件名称即可。

如果使用定制的 [Patroni配置模板](v-pg-provision.md#pg_conf)，通常也应当针对机器节点使用配套的 [节点优化模板](v-node#node_tune)。



## Patroni模板样例

定制您自己的Patroni模板时，您可以用已有的几种基础模板作为基线，在此基础上进行修改。

并放置于[`templates/`](https://github.com/Vonng/pigsty/tree/master/roles/postgres/templates)目录中，以`mode.yml`格式命名即可。

Patroni中的模板变量请保留，否则相关参数可能无法正常工作。

一个典型的Patroni配置文件（OLTP）

```yaml
#!/usr/bin/env patroni
#==============================================================#
# File      :   patroni.yml
# Ctime     :   2020-04-08
# Mtime     :   2020-12-22
# Desc      :   patroni cluster definition for {{ pg_cluster }} (oltp)
# Path      :   /pg/bin/patroni.yml
# Real Path :   /pg/conf/{{ pg_instance }}.yml
# Link      :   /pg/bin/patroni.yml -> /pg/conf/{{ pg_instance}}.yml
# Note      :   Transactional Database Cluster Template
# Doc       :   https://patroni.readthedocs.io/en/latest/SETTINGS.html
# Copyright (C) 2018-2021 Ruohang Feng
#==============================================================#

# OLTP database are optimized for performance, rt latency
# typical spec: 64 Core | 400 GB RAM | PCI-E SSD xTB

---
#------------------------------------------------------------------------------
# identity
#------------------------------------------------------------------------------
namespace: {{ pg_namespace }}/          # namespace
scope: {{ pg_cluster }}                 # cluster name
name: {{ pg_instance }}                 # instance name

#------------------------------------------------------------------------------
# log
#------------------------------------------------------------------------------
log:
  level: INFO                           #  NOTEST|DEBUG|INFO|WARNING|ERROR|CRITICAL
  dir: /pg/log/                         #  default log file: /pg/log/patroni.log
  file_size: 33554432                   #  32MB log triggers log rotation
  file_num: 20                          #  keep at most 30x32MB = 1GB log
  dateformat: '%Y-%m-%d %H:%M:%S %z'    #  IMPORTANT: discard milli timestamp
  format: '%(asctime)s %(levelname)s: %(message)s'

#------------------------------------------------------------------------------
# dcs
#------------------------------------------------------------------------------
consul:
  host: 127.0.0.1:8500
  consistency: default         # default|consistent|stale
  register_service: true
  service_check_interval: 15s
  service_tags:
    - {{ pg_cluster }}

#------------------------------------------------------------------------------
# api
#------------------------------------------------------------------------------
# how to expose patroni service
# listen on all ipv4, connect via public ip, use same credential as dbuser_monitor
restapi:
  listen: 0.0.0.0:{{ patroni_port }}
  connect_address: {{ inventory_hostname }}:{{ patroni_port }}
  authentication:
    verify_client: none                 # none|optional|required
    username: {{ pg_monitor_username }}
    password: '{{ pg_monitor_password }}'

#------------------------------------------------------------------------------
# ctl
#------------------------------------------------------------------------------
ctl:
  optional:
    insecure: true
    # cacert: '/path/to/ca/cert'
    # certfile: '/path/to/cert/file'
    # keyfile: '/path/to/key/file'

#------------------------------------------------------------------------------
# tags
#------------------------------------------------------------------------------
tags:
  nofailover: false
  clonefrom: true
  noloadbalance: false
  nosync: false
{% if pg_upstream is defined %}
  replicatefrom: {{ pg_upstream }}    # clone from another replica rather than primary
{% endif %}

#------------------------------------------------------------------------------
# watchdog
#------------------------------------------------------------------------------
# available mode: off|automatic|required
watchdog:
  mode: {{ patroni_watchdog_mode }}
  device: /dev/watchdog
  # safety_margin: 10s

#------------------------------------------------------------------------------
# bootstrap
#------------------------------------------------------------------------------
bootstrap:

  #----------------------------------------------------------------------------
  # bootstrap method
  #----------------------------------------------------------------------------
  method: initdb
  # add custom bootstrap method here

  # default bootstrap method: initdb
  initdb:
{% if pg_encoding != '' %}
    - encoding: {{ pg_encoding }}
{% endif %}
{% if pg_locale != '' %}
    - locale: {{ pg_locale }}
{% endif %}
{% if pg_lc_collate != '' %}
    - lc-collate: {{ pg_lc_collate }}
{% endif %}
{% if pg_lc_ctype != '' %}
    - lc-ctype: {{ pg_lc_ctype }}
{% endif %}

  #----------------------------------------------------------------------------
  # bootstrap users
  #---------------------------------------------------------------------------
  # additional users which need to be created after initializing new cluster
  # replication user and monitor user are required
  users:
    {{ pg_replication_username }}:
      password: '{{ pg_replication_password }}'
    {{ pg_monitor_username }}:
      password: '{{ pg_monitor_password }}'
    {{ pg_admin_username }}:
      password: '{{ pg_admin_password }}'

  # bootstrap hba, allow local and intranet password access & replication
  # will be overwritten later
  pg_hba:
    - local   all             postgres                                ident
    - local   all             all                                     md5
    - host    all             all            0.0.0.0/0                md5
    - local   replication     postgres                                ident
    - local   replication     all                                     md5
    - host    replication     all            0.0.0.0/0                md5


  #----------------------------------------------------------------------------
  # template
  #---------------------------------------------------------------------------
  # post_init: /pg/bin/pg-init

  #----------------------------------------------------------------------------
  # bootstrap config
  #---------------------------------------------------------------------------
  # this section will be written to /{{ pg_namespace }}/{{ pg_cluster }}/config
  # if will NOT take any effect after cluster bootstrap
  dcs:

{% if pg_role == 'primary' and pg_upstream is defined %}
    #----------------------------------------------------------------------------
    # standby cluster definition
    #---------------------------------------------------------------------------
    standby_cluster:
      host: {{ pg_upstream }}
      port: {{ pg_port }}
      # primary_slot_name: patroni     # must be create manually on upstream server, if specified
      create_replica_methods:
        - basebackup
{% endif %}

    #----------------------------------------------------------------------------
    # important parameters
    #---------------------------------------------------------------------------
    # constraint: ttl >: loop_wait + retry_timeout * 2

    # the number of seconds the loop will sleep. Default value: 10
    # this is patroni check loop interval
    loop_wait: 10

    # the TTL to acquire the leader lock (in seconds). Think of it as the length of time before initiation of the automatic failover process. Default value: 30
    # config this according to your network condition to avoid false-positive failover
    ttl: 30

    # timeout for DCS and PostgreSQL operation retries (in seconds). DCS or network issues shorter than this will not cause Patroni to demote the leader. Default value: 10
    retry_timeout: 10

    # the amount of time a master is allowed to recover from failures before failover is triggered (in seconds)
    # Max RTO: 2 loop wait + master_start_timeout
    master_start_timeout: 10

    # import: candidate will not be promoted if replication lag is higher than this
    # maximum RPO: 1MB
    maximum_lag_on_failover: 1048576

    # The number of seconds Patroni is allowed to wait when stopping Postgres and effective only when synchronous_mode is enabled
    master_stop_timeout: 30

    # turns on synchronous replication mode. In this mode a replica will be chosen as synchronous and only the latest leader and synchronous replica are able to participate in leader election
    # set to true for RPO mode
    synchronous_mode: false

    # prevents disabling synchronous replication if no synchronous replicas are available, blocking all client writes to the master
    synchronous_mode_strict: false


    #----------------------------------------------------------------------------
    # postgres parameters
    #---------------------------------------------------------------------------
    postgresql:
      use_slots: true
      use_pg_rewind: true
      remove_data_directory_on_rewind_failure: true


      parameters:
        #----------------------------------------------------------------------
        # IMPORTANT PARAMETERS
        #----------------------------------------------------------------------
        max_connections: 800                    # 100 -> 800
        superuser_reserved_connections: 10      # reserve 10 connection for su
        max_locks_per_transaction: 128          # 64 -> 128
        max_prepared_transactions: 0            # 0 disable 2PC
        track_commit_timestamp: on              # enabled xact timestamp
        max_worker_processes: 64                # default 8 -> 64, set to cpu core 64
        wal_level: logical                      # logical
        wal_log_hints: on                       # wal log hints to support rewind
        max_wal_senders: 24                     # 10 -> 24
        max_replication_slots: 16               # 10 -> 16
        wal_keep_size: 100GB                    # keep at least 100GB WAL
        password_encryption: md5                # use traditional md5 auth

        #----------------------------------------------------------------------
        # RESOURCE USAGE (except WAL)
        #----------------------------------------------------------------------
        # memory: shared_buffers and maintenance_work_mem will be dynamically set
        shared_buffers: {{ pg_shared_buffers }}
        maintenance_work_mem: {{ pg_maintenance_work_mem }}
        work_mem: 32MB                          # 4MB -> 32MB
        huge_pages: try                         # try huge pages
        temp_file_limit: 100GB                  # 0 -> 100GB
        vacuum_cost_delay: 2ms                  # wait 2ms per 10000 cost
        vacuum_cost_limit: 10000                # 10000 cost each round
        bgwriter_delay: 10ms                    # check dirty page every 10ms
        bgwriter_lru_maxpages: 800              # 100 -> 800
        bgwriter_lru_multiplier: 5.0            # 2.0 -> 5.0  more cushion buffer
        max_parallel_workers: 32                # default 8 -> 32, limit by max_worker_processes
        max_parallel_maintenance_workers: 8     # default 2 -> 8, limit by parallel worker
        max_parallel_workers_per_gather: 0      # default 2 -> 0, disable parallel query in OLTP mode

        #----------------------------------------------------------------------
        # WAL
        #----------------------------------------------------------------------
        wal_buffers: 16MB                       # max to 16MB
        wal_writer_delay: 20ms                  # wait period
        wal_writer_flush_after: 1MB             # max allowed data loss
        min_wal_size: 100GB                     # at least 100GB WAL
        max_wal_size: 400GB                     # at most 400GB WAL
        commit_delay: 20                        # 200ms -> 20ms, increase speed
        commit_siblings: 10                     # 5 -> 10
        checkpoint_timeout: 60min               # checkpoint 5min -> 1h
        checkpoint_completion_target: 0.95      # 0.5 -> 0.95
        archive_mode: on
        archive_command: 'wal_dir=/pg/arcwal; [[ $(date +%H%M) == 1200 ]] && rm -rf ${wal_dir}/$(date -d"yesterday" +%Y%m%d); /bin/mkdir -p ${wal_dir}/$(date +%Y%m%d) && /usr/bin/lz4 -q -z %p > ${wal_dir}/$(date +%Y%m%d)/%f.lz4'

        #----------------------------------------------------------------------
        # REPLICATION
        #----------------------------------------------------------------------
        # synchronous_standby_names: ''
        vacuum_defer_cleanup_age: 50000         # 0->50000 last 50000 xact changes will not be vacuumed
        promote_trigger_file: promote.signal    # default promote trigger file path
        max_standby_archive_delay: 10min        # max delay before canceling queries when reading WAL from archive;
        max_standby_streaming_delay: 3min       # max delay before canceling queries when reading streaming WAL;
        wal_receiver_status_interval: 1s        # send replies at least this often
        hot_standby_feedback: on                # send info from standby to prevent query conflicts
        wal_receiver_timeout: 60s               # time that receiver waits for
        max_logical_replication_workers: 8      # 4 -> 8, 6 sync worker + 1~2 apply worker
        max_sync_workers_per_subscription: 6    # 2 -> 6, 6 sync worker

        #----------------------------------------------------------------------
        # QUERY TUNING
        #----------------------------------------------------------------------
        # planner
        # enable_partitionwise_join: on
        random_page_cost: 1.1                   # 4 for HDD, 1.1 for SSD
        effective_cache_size: 320GB             # max mem - shared buffer
        default_statistics_target: 1000         # stat bucket 100 -> 1000

        #----------------------------------------------------------------------
        # REPORTING AND LOGGING
        #----------------------------------------------------------------------
        log_destination: csvlog                 # use standard csv log
        logging_collector: on                   # enable csvlog
        log_directory: log                      # default log dir: /pg/data/log
        # log_filename: 'postgresql-%a.log'     # weekly auto-recycle
        log_filename: 'postgresql-%Y-%m-%d.log' # YYYY-MM-DD full log retention
        log_checkpoints: on                     # log checkpoint info
        log_lock_waits: on                      # log lock wait info
        log_replication_commands: on            # log replication info
        log_statement: ddl                      # log ddl change
        log_min_duration_statement: 100         # log slow query (>100ms)

        #----------------------------------------------------------------------
        # STATISTICS
        #----------------------------------------------------------------------
        track_io_timing: on                     # collect io statistics
        track_functions: all                    # track all functions (none|pl|all)
        track_activity_query_size: 8192         # max query length in pg_stat_activity

        #----------------------------------------------------------------------
        # AUTOVACUUM
        #----------------------------------------------------------------------
        log_autovacuum_min_duration: 1s         # log autovacuum activity take more than 1s
        autovacuum_max_workers: 3               # default autovacuum worker 3
        autovacuum_naptime: 1min                # default autovacuum naptime 1min
        autovacuum_vacuum_scale_factor: 0.08    # fraction of table size before vacuum   20% -> 8%
        autovacuum_analyze_scale_factor: 0.04   # fraction of table size before analyze  10% -> 4%
        autovacuum_vacuum_cost_delay: -1        # default vacuum cost delay: same as vacuum_cost_delay
        autovacuum_vacuum_cost_limit: -1        # default vacuum cost limit: same as vacuum_cost_limit
        autovacuum_freeze_max_age: 100000000    # age > 1 billion triggers force vacuum

        #----------------------------------------------------------------------
        # CLIENT
        #----------------------------------------------------------------------
        deadlock_timeout: 50ms                  # 50ms for deadlock
        idle_in_transaction_session_timeout: 10min  # 10min timeout for idle in transaction

        #----------------------------------------------------------------------
        # CUSTOMIZED OPTIONS
        #----------------------------------------------------------------------
        # extensions
        shared_preload_libraries: '{{ pg_shared_libraries | default("pg_stat_statements, auto_explain") }}'

        # auto_explain
        auto_explain.log_min_duration: 1s       # auto explain query slower than 1s
        auto_explain.log_analyze: true          # explain analyze
        auto_explain.log_verbose: true          # explain verbose
        auto_explain.log_timing: true           # explain timing
        auto_explain.log_nested_statements: true

        # pg_stat_statements
        pg_stat_statements.max: 10000           # 5000 -> 10000 queries
        pg_stat_statements.track: all           # track all statements (all|top|none)
        pg_stat_statements.track_utility: off   # do not track query other than CRUD
        pg_stat_statements.track_planning: off  # do not track planning metrics


#------------------------------------------------------------------------------
# postgres
#------------------------------------------------------------------------------
postgresql:

  #----------------------------------------------------------------------------
  # how to connect to postgres
  #----------------------------------------------------------------------------
  bin_dir: {{ pg_bin_dir }}
  data_dir: {{ pg_data }}
  config_dir: {{ pg_data }}
  pgpass: {{ pg_dbsu_home }}/.pgpass
  listen: {{ pg_listen }}:{{ pg_port }}
  connect_address: {{ inventory_hostname }}:{{ pg_port }}
  use_unix_socket: true # default: /var/run/postgresql, /tmp

  #----------------------------------------------------------------------------
  # who to connect to postgres
  #----------------------------------------------------------------------------
  authentication:
    superuser:
      username: {{ pg_dbsu }}
    replication:
      username: {{ pg_replication_username }}
      password: '{{ pg_replication_password }}'
    rewind:
      username: {{ pg_replication_username }}
      password: '{{ pg_replication_password }}'

  #----------------------------------------------------------------------------
  # how to react to database operations
  #----------------------------------------------------------------------------
  # event callback script log: /pg/log/callback.log
  callbacks:
    on_start: /pg/bin/pg-failover-callback
    on_stop: /pg/bin/pg-failover-callback
    on_reload: /pg/bin/pg-failover-callback
    on_restart: /pg/bin/pg-failover-callback
    on_role_change: /pg/bin/pg-failover-callback

  # rewind policy: data checksum should be enabled before using rewind
  use_pg_rewind: true
  remove_data_directory_on_rewind_failure: true
  remove_data_directory_on_diverged_timelines: false

  #----------------------------------------------------------------------------
  # how to create replica
  #----------------------------------------------------------------------------
  # create replica method: default pg_basebackup
  create_replica_methods:
    - basebackup
  basebackup:
    - max-rate: '1000M'
    - checkpoint: fast
    - status-interva: 1s
    - verbose
    - progress

  #----------------------------------------------------------------------------
  # ad hoc parameters (overwrite with default)
  #----------------------------------------------------------------------------
  # parameters:

  #----------------------------------------------------------------------------
  # host based authentication, overwrite default pg_hba.conf
  #----------------------------------------------------------------------------
  # pg_hba:
  #   - local   all             postgres                                ident
  #   - local   all             all                                     md5
  #   - host    all             all            0.0.0.0/0                md5
  #   - local   replication     postgres                                ident
  #   - local   replication     all                                     md5
  #   - host    replication     all            0.0.0.0/0                md5

...
```

