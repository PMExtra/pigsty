# File Hierarchy Structure

> How files are organized in Pigsty.


----------------

## Pigsty FHS

```bash
#------------------------------------------------------------------------------
# pigsty
#  ^-----@app                    # extra demo application resources
#  ^-----@bin                    # bin scripts
#  ^-----@docs                   # document (can be docsified)
#  ^-----@files                  # ansible file resources 
#            ^-----@pigsty       # pigsty config template files
#            ^-----@prometheus   # prometheus rules definition
#            ^-----@grafana      # grafana dashboards
#            ^-----@postgres     # /pg/bin/ scripts
#            ^-----@migration    # pgsql migration task definition
#            ^-----@pki          # self-signed CA & certs

#  ^-----@roles                  # ansible business logic
#  ^-----@templates              # ansible templates
#  ^-----@vagrant                # vagrant local VM template
#  ^-----@terraform              # terraform cloud VM template
#  ^-----configure               # configure wizard script
#  ^-----ansible.cfg             # default ansible config file
#  ^-----pigsty.yml              # default config file
#  ^-----*.yml                   # ansible playbooks

#------------------------------------------------------------------------------
# /etc/pigsty/
#  ^-----@targets                # file based service discovery targets definition
#  ^-----@dashboards             # static grafana dashboards
#  ^-----@datasources            # static grafana datasource
#  ^-----@playbooks              # extra ansible playbooks
#------------------------------------------------------------------------------
```


----------------

## CA FHS

Pigsty's self-signed CA is located on `files/pki/` directory under pigsty home.

**YOU HAVE TO SECURE THE CA KEY PROPERLY**: `files/pki/ca/ca.key`,
which is generated by the [`ca`](https://github.com/Vonng/pigsty/tree/master/roles/ca) role during `install.yml` or `infra.yml`.

```bash
# pigsty/files/pki
#  ^-----@ca                      # self-signed CA key & cert
#         ^-----@ca.key           # VERY IMPORTANT: keep it secret
#         ^-----@ca.crt           # VERY IMPORTANT: trusted everywhere
#  ^-----@csr                     # signing request csr
#  ^-----@misc                    # misc certs, issued certs
#  ^-----@etcd                    # etcd server certs
#  ^-----@minio                   # minio server certs
#  ^-----@nginx                   # nginx SSL certs
#  ^-----@infra                   # infra client certs
#  ^-----@pgsql                   # pgsql server certs
#  ^-----@mongo                   # mongodb/ferretdb server certs
#  ^-----@mysql                   # mysql server certs
```

The managed nodes will have the following files installed:

```
/etc/pki/ca.crt                             # all nodes
/etc/pki/ca-trust/source/anchors/ca.crt     # soft link and trusted anchor
```

All infra nodes will have the following certs:

```
/etc/pki/infra.crt                          # infra nodes cert
/etc/pki/infra.key                          # infra nodes key
```

In case of admin node failure, you have to keep `files/pki` and `pigsty.yml` safe.
You can `rsync` them to another admin node to make a backup admin node.

```bash
# run on meta-1, rsync to meta2
cd ~/pigsty;
rsync -avz ./ meta-2:~/pigsty  
```



----------------

## NODE FHS

Node main data dir is specified by [`node_data`](param#node_data) parameter, which is `/data` by default.

The data dir is owned by root with mode `0777`. All modules' local data will be stored under this directory by default.

```bash
/data
#  ^-----@postgres                   # postgres main data dir
#  ^-----@backups                    # postgres backup data dir (if no dedicated backup disk)
#  ^-----@redis                      # redis data dir (shared by multiple redis instances)
#  ^-----@minio                      # minio data dir (default when in single node single disk mode)
#  ^-----@etcd                       # etcd main data dir
#  ^-----@prometheus                 # prometheus time series data dir
#  ^-----@loki                       # Loki data dir for logs
#  ^-----@docker                     # Docker data dir
#  ^-----@...                        # other modules
```



----------------

## Prometheus FHS

The prometheus bin / rules are located on [`files/prometheus/`](https://github.com/Vonng/pigsty/tree/master/files/prometheus) directory under pigsty home.

While the main config file is located on [`roles/infra/templates/prometheus/prometheus.yml.j2`](https://github.com/Vonng/pigsty/blob/master/roles/infra/templates/prometheus/prometheus.yml.j2) and rendered to `/etc/prometheus/prometheus.yml` on infra nodes.

```bash
# /etc/prometheus/
#  ^-----prometheus.yml              # prometheus main config file
#  ^-----@bin                        # util scripts: check,reload,status,new
#  ^-----@rules                      # record & alerting rules definition
#            ^-----agent.yml         # agent rules & alert
#            ^-----infra.yml         # infra rules & alert
#            ^-----node.yml          # node  rules & alert
#            ^-----pgsql.yml         # pgsql rules & alert
#            ^-----redis.yml         # redis rules & alert
#            ^-----minio.yml         # minio rules & alert
#            ^-----etcd.yml          # etcd  rules & alert
#            ^-----mongo.yml         # mongo rules & alert
#            ^-----mysql.yml         # mysql rules & alert (placeholder)
#  ^-----@targets                    # file based service discovery targets definition
#            ^-----@infra            # infra static targets definition
#            ^-----@node             # nodes static targets definition
#            ^-----@etcd             # etcd static targets definition
#            ^-----@minio            # minio static targets definition
#            ^-----@ping             # blackbox ping targets definition
#            ^-----@pgsql            # pgsql static targets definition
#            ^-----@pgrds            # pgsql remote rds static targets
#            ^-----@redis            # redis static targets definition
#            ^-----@mongo            # mongo static targets definition
#            ^-----@mysql            # mysql static targets definition
#            ^-----@ping             # ping  static target definition
#            ^-----@patroni          # patroni static target defintion (when ssl enabled)
#            ^-----@.....            # other targets
# /etc/alertmanager.yml              # alertmanager main config file
# /etc/blackbox.yml                  # blackbox exporter main config file
```



----------------

## Postgres FHS

The following parameters are related to the PostgreSQL database dir:

* [pg_dbsu_home](PARAM#pg_dbsu_home): Postgres default user's home dir, default is `/var/lib/pgsql`.
* [pg_bin_dir](PARAM#pg_bin_dir): Postgres binary dir, defaults to `/usr/pgsql/bin/`.
* [pg_data](PARAM#pg_data): Postgres database dir, default is `/pg/data`.
* [pg_fs_main](PARAM#pg_fs_main): Postgres main data disk mount point, default is `/data`.
* [pg_fs_bkup](PARAM#pg_fs_bkup): Postgres backup disk mount point, default is `/data/backups` (used when using local backup repo).

```yaml
#--------------------------------------------------------------#
# Create Directory
#--------------------------------------------------------------#
# assumption:
#   {{ pg_fs_main }} for main data   , default: `/data`              [fast ssd]
#   {{ pg_fs_bkup }} for backup data , default: `/data/backups`     [cheap hdd]
#--------------------------------------------------------------#
# default variable:
#     pg_fs_main = /data             fast ssd
#     pg_fs_bkup = /data/backups     cheap hdd (optional)
#
#     /pg      -> /data/postgres/pg-test-15    (soft link)
#     /pg/data -> /data/postgres/pg-test-15/data
#--------------------------------------------------------------#
- name: create postgresql directories
  tags: pg_dir
  become: yes
  block:

    - name: make main and backup data dir
      file: path={{ item }} state=directory owner=root mode=0777
      with_items:
        - "{{ pg_fs_main }}"
        - "{{ pg_fs_bkup }}"

    # pg_cluster_dir:    "{{ pg_fs_main }}/postgres/{{ pg_cluster }}-{{ pg_version }}"
    - name: create postgres directories
      file: path={{ item }} state=directory owner={{ pg_dbsu }} group=postgres mode=0700
      with_items:
        - "{{ pg_fs_main }}/postgres"
        - "{{ pg_cluster_dir }}"
        - "{{ pg_cluster_dir }}/bin"
        - "{{ pg_cluster_dir }}/log"
        - "{{ pg_cluster_dir }}/tmp"
        - "{{ pg_cluster_dir }}/cert"
        - "{{ pg_cluster_dir }}/conf"
        - "{{ pg_cluster_dir }}/data"
        - "{{ pg_cluster_dir }}/meta"
        - "{{ pg_cluster_dir }}/stat"
        - "{{ pg_cluster_dir }}/change"
        - "{{ pg_backup_dir }}/backup"
```


**Data FHS**

```bash
# real dirs
{{ pg_fs_main }}     /data                      # top level data directory, usually a SSD mountpoint
{{ pg_dir_main }}    /data/postgres             # contains postgres data
{{ pg_cluster_dir }} /data/postgres/pg-test-15  # contains cluster `pg-test` data (of version 15)
                     /data/postgres/pg-test-15/bin            # bin scripts
                     /data/postgres/pg-test-15/log            # logs: postgres/pgbouncer/patroni/pgbackrest
                     /data/postgres/pg-test-15/tmp            # tmp, sql files, rendered results
                     /data/postgres/pg-test-15/cert           # postgres server certificates
                     /data/postgres/pg-test-15/conf           # patroni config, links to related config
                     /data/postgres/pg-test-15/data           # main data directory
                     /data/postgres/pg-test-15/meta           # identity information
                     /data/postgres/pg-test-15/stat           # stats information, summary, log report
                     /data/postgres/pg-test-15/change         # changing records
                     /data/postgres/pg-test-15/backup         # soft link to backup dir

{{ pg_fs_bkup }}     /data/backups                            # could be a cheap & large HDD mountpoint
                     /data/backups/postgres/pg-test-15/backup # local backup repo path

# soft links
/pg             ->   /data/postgres/pg-test-15                # pg root link
/pg/data        ->   /data/postgres/pg-test-15/data           # real data dir
/pg/backup      ->   /var/backups/postgres/pg-test-15/backup  # base backup
```


**Binary FHS**

On EL releases, the default path for PostgreSQL binaries is:

```bash
/usr/pgsql-${pg_version}/
```

Pigsty will create a softlink `/usr/pgsql` to the currently installed version specified by [`pg_version`](PARAM#pg_version).

```bash
/usr/pgsql -> /usr/pgsql-15
```

Therefore, the default [`pg_bin_dir`](PARAM#pg_bin_dir) will be `/usr/pgsql/bin/`, and this path is added to the `PATH` environment via `/etc/profile.d/pgsql.sh`.

```bash
export PATH="/usr/pgsql/bin:/pg/bin:$PATH"
export PGHOME=/usr/pgsql
export PGDATA=/pg/data
```

For Ubuntu / Debian, the default path for PostgreSQL binaries is:

```bash
/usr/lib/postgresql/${pg_version}/bin
```


----------------

## Pgbouncer FHS

Pgbouncer is run using the Postgres user, and the config file is located in `/etc/pgbouncer`. The config file includes.

* `pgbouncer.ini`: pgbouncer main config
* `database.txt`: pgbouncer database list
* `userlist.txt`: pgbouncer user list
* `useropts.txt`: pgbouncer user options (user-level parameter overrides)
* `pgb_hba.conf`: lists the access privileges of the connection pool users



----------------

## Redis FHS

Pigsty provides essential support for Redis deployment and monitoring.

Redis binaries are installed in `/bin/` using RPM-packages or copied binaries, including:

```bash
redis-server    
redis-server    
redis-cli       
redis-sentinel  
redis-check-rdb 
redis-check-aof 
redis-benchmark 
/usr/libexec/redis-shutdown
```

For a Redis instance named `redis-test-1-6379`, the resources associated with it are shown below:

```bash
/usr/lib/systemd/system/redis-test-1-6379.service               # Services ('/lib/systemd' in debian)
/etc/redis/redis-test-1-6379.conf                               # Config 
/data/redis/redis-test-1-6379                                   # Database Catalog
/data/redis/redis-test-1-6379/redis-test-1-6379.rdb             # RDB File
/data/redis/redis-test-1-6379/redis-test-1-6379.aof             # AOF file
/var/log/redis/redis-test-1-6379.log                            # Log
/var/run/redis/redis-test-1-6379.pid                            # PID
```


For Ubuntu / Debian, the default systemd service dir is `/lib/systemd/system/` instead of `/usr/lib/systemd/system/`.
