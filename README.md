# Supported tags and respective `Dockerfile` links

-   [`latest` (*latest/Dockerfile*)](https://github.com/babim/docker-mariadb-galera/blob/master/Dockerfile)

# How to use this image.

When you run the image without giving any command, the entrypoint will listen at port 13306 waiting for a command.
To send a command, jun run:

    echo "command here" | nc ip_of_container port

So, basically, you can get a cluster running by following the steps:

- start each container with:

without password
```
docker run \
    -v /path/to/my.cnf:/etc/mysql/my.cnf \
    -v /path/to/mariadb:/var/lib/mysql \
    -v /path/to/certs/:/etc/ssl/mysql:ro \
    -e MYSQL_ALLOW_EMPTY_PASSWORD=true \
    babim/mariadb-galera
```
with password
```
docker run \
    -v /path/to/my.cnf:/etc/mysql/my.cnf \
    -v /path/to/mariadb:/var/lib/mysql \
    -v /path/to/certs/:/etc/ssl/mysql:ro \
    -e MYSQL_ROOT_PASSWORD==123456 \
    babim/mariadb-galera
```

- concat all containers IP into a `gcomm://` string, like:

```
gcomm://10.0.0.1,10.0.0.2,10.0.0.3
```

- start first container as new cluster:

```
echo "mysqld --wsrep-new-cluster --wsrep-cluster-address=gcomm://10.0.0.1,10.0.0.2,10.0.0.3" | nc 10.0.0.1 13306
```

- on all the subsequent containers, one at a time, starts it as part of existing cluster:

```
echo "mysqld --wsrep-cluster-address=gcomm://10.0.0.1,10.0.0.2,10.0.0.3" | nc 10.0.0.2 13306
echo "mysqld --wsrep-cluster-address=gcomm://10.0.0.1,10.0.0.2,10.0.0.3" | nc 10.0.0.3 13306
```

That's it, now you should have a running MariaDB cluster.

## SSL certificates

You can generate self-signed certificate, mount the directory inside the container and set the appropriate `[sst]` settings at `/etc/mysql/my.cnf`.

## Example `my.cnf` file:

```
[mysqld]
datadir=/var/lib/mysql
socket=/var/run/mysqld/mysqld.sock
user=mysql
binlog-format=ROW
default-storage-engine=InnoDB
innodb-autoinc-lock-mode=2
innodb-flush-log-at-trx-commit=0
innodb-buffer-pool-size=122M
innodb-doublewrite=1
innodb-flush-method=O_DIRECT
wsrep-provider=/usr/lib/galera/libgalera_smm.so
wsrep-cluster-name=my_super_cluster
wsrep-sst-method=rsync
query-cache-size=524288
log-error=/dev/stderr

[mysql_safe]
log-error=/dev/stderr
pid-file=/var/run/mysqld/mysqld.pid

[sst]
tca=/etc/ssl/mysql/ca.pem
tcert=/etc/ssl/mysql/server-cert.pem
tkey=/etc/ssl/mysql/server-key.pem
```

# check status:
```
docker exec -ti database01 mysql -e "SHOW STATUS LIKE 'wsrep_cluster%'"
+--------------------------+--------------------------------------+
| Variable_name            | Value                                |
+--------------------------+--------------------------------------+
| wsrep_cluster_conf_id    | 3                                    |
| wsrep_cluster_size       | 3                                    |
| wsrep_cluster_state_uuid | 2882bcb7-ab3b-11e5-ab75-2b510ef0ec6f |
| wsrep_cluster_status     | Primary                              |
+--------------------------+--------------------------------------+
```
