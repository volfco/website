---
title: "Exploring Clickhouse Users"
date: 2018-09-29T11:36:33+08:00
draft: true
tags: 
  - MySQL
  - puppet
  - MySQL Galera
  - automation
---

= Overview

MySQL Galera is a hip new way to run MySQL. Not really, but it's the best MySQL Master-Master solution on the market today.

There are a few gotchas with Galera, that I'll try to address in the rest of this article

I wrote this article for myself, but it's the result of hours of trial and error, so it has value for you and everyone else trying to do this.

= Deployment
This is fragile process, sadly. I would recommend running through this guide more then once. Each environment will be different, so I recommend that you make sure your "base" system is fully provisioned before trying to layer on top Galera.

I also use Hiera as much as possible, so a lot of the code examples here are 

NOTE. SELinux does not work with Percona Galera, which is the database I'm using here. 

== CentOS 8 Sanitization
Make sure all MariaDB packages are removed if you're using Percona. 

There are cases where some mariadb libs (mariadb-connector-c) are installed by default on some cloud providers (like DigitalOcean), and these can cause dependency issues.

`dnf remove -y mariadb*`

Just to be save and to avoid frusturation down the road. 

== Puppet Modules
For this guide, I'm not going to use the Puppet Galera module, as I was unable to get it to work under CentOS 8. 

https://forge.puppet.com/puppetlabs/mysql

== Deploy Percona Cluster
First step is to our MySQL server. What's wonderful is the puppet module does a vast majority of the work, we just need to configure it. 

Make sure you validate that mysql is functional and accessable on each node in the cluster. Due to dependency issues, it's always a good idea to run puppet a few times to make sure the runs come back clean. 

== Stop Puppet
Now that we have a valid mysql installation running on each node, we want to stop and disable puppet once we have our Galera configuration copied over.

`puppet agent --disable`

Now, we want to make sure mysql.service is stopped, and the old data directory has been removed. 

`rm -rf /var/lib/mysql/*`

If you're using a non standard directory, make sure that's empty as well. 

= Runbook

== Bringing the Cluster up
This is the fragile part. Galera needs to be started in a specific order. 

When starting a new cluster, we can pick any node in our cluster to run the bootstrap. 

Pick a node, and start mysql@boostrap.service

If this fails, I can't help you. There are a bunch of reasons this could fail. 

Before we can move on to other nodes, we need to configure the first one. 

Puppet can do most of this later, but for now we need to manually change the root password, and make the sstuser

1st step is to change the root user's password. Change this to what you configured the root password to be in puppet. We still want the management functionality in puppet, and this allows it to take over once we re-enable puppet. 

```bash
cat /var/log/mysqld.log | grep password
```

You're looking for a line similar to `A temporary password is generated for root@localhost: ...`. That's our root password. Use this to login to mysql. You might need to modify the puppet generated `~/.my.cnf` file- you only need to edit the client section. Make sure you put this back to what the root password is defined as. 

```sql
ALTER USER 'root'@'localhost' IDENTIFIED BY 'password';
```

```sql
CREATE USER 'sstuser'@'localhost' IDENTIFIED BY 'vNsnocrkrL4v62yhfqSLsRsXdn2Wnbsc';
GRANT PROCESS, RELOAD, LOCK TABLES, REPLICATION CLIENT ON *.* TO 'sstuser'@'localhost';
```

Now is a good time to make sure the rquired firewall ports are open. 3306, 4567, and 4444 are the important onces that need to be opened on all nodes. 

Once this is done, we can continue onto starting the other nodes. Start the `mysql.service` process. If you watch the logs on the node you chose to bootstrap, you should see a bunch of output about syncing and joining the other members. 

If starting of mysql fails on any of the other nodes, it's most likely due to the nodes being unable to sync state from the bootstrap node. 

We can verify that all nodes are online by running the following on any node in the cluster from the mysql shell:

```
mysql> show status like 'wsrep_cluster_size';
+--------------------+-------+
| Variable_name      | Value |
+--------------------+-------+
| wsrep_cluster_size | 3     |
+--------------------+-------+
1 row in set (0.00 sec)

mysql>
```

If that number looks correct, we can move on to finishing up. But before, we need to stop `mysql@bootstrap.service` on our first node. Start mysql like you would 

== Finishing up
The hardest part of this process is getting puppet into a consistent state. Under CentOS 8 I've had seemingly transient issues with package dependencies and package conflicts. 

If `percona-xtrabackup-24` gets installed before `Percona-XtraDB-Cluster-server-57`, you might get package conflicts. Which is why I recommend adding in one component at a time. 

I'm almost hesistant to even recommend the mysql module, but I believe that it will make things easier in the long run. 

= Recovery

This is important to know. In the case of a total shutdown, where every node is offline, you have to turn it on in basically the same order, and you have to validate that all nodes are at the same state before doing it. This is where `grastate.dat` comes in. This describes the last know position of the cluster as the node you're on sees it.

```bash
$ cat /var/lib/mysql/grastate.dat
# GALERA saved state
version: 2.1
uuid:    5dec3951-df7d-11ea-bc5c-2e23ca2c9d4a
seqno:   4510
safe_to_bootstrap: 0
```

In my above example, the cluster was not shutdown cleanly. If it was, there would be a node where `safe_to_bootstrap` was set to 1. 

In a clean shutdown, you want to start the `mysql.service` where `safe_to_bootstrap` is set to 1. Once that node is up, you can start up the other nodes. 

If there was an unclean shutdown, such as the DC loses power, things get a bit more complex. We need to query the database for the last transaction the node knows about. For Percona, we can do 

```bash
$ mysqld_safe --basedir=/usr --wsrep-recover
2020-08-16T05:58:11.554816Z mysqld_safe Logging to '/var/log/mysqld.log'.
2020-08-16T05:58:11.557901Z mysqld_safe Logging to '/var/log/mysqld.log'.
2020-08-16T05:58:11.585871Z mysqld_safe Starting mysqld daemon with databases from /var/lib/mysql
2020-08-16T05:58:11.596217Z mysqld_safe Skipping wsrep-recover for 5dec3951-df7d-11ea-bc5c-2e23ca2c9d4a:4 pair
2020-08-16T05:58:11.597484Z mysqld_safe Assigning 5dec3951-df7d-11ea-bc5c-2e23ca2c9d4a:4 to wsrep_start_position
2020-08-16T05:58:14.524669Z mysqld_safe mysqld from pid file /var/run/mysqld/mysqld.pid ended
```

The line we want, is the line ending in _wsrep\_start\_position_, we can see our node UUID and the seqno of 4. This node has a seqno of 4. Do this on all nodes and pick the highest number. Edit the grastate file, and set safe_to_bootstrap to 1. Now, start the mysql service. Once the node is up, start up the other nodes. 


= References

- https://severalnines.com/blog/updated-how-bootstrap-mysql-or-mariadb-galera-cluster