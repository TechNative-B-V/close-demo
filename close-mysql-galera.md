# ProxySQL with Galera Cluster

## Author
Mathijs van Veluw  
mathijs@technative.nl  
![technative](file:///home/mathijs/Documents/TechNative/technative-small.png "Logo Title Text 1")


## Install and Configure Galera Cluster

### Update MariaDB from 10.3 to 10.4

The following needs to be done on all database nodes.  
Update the repo version in `/etc/apt/sources.list.d/mariadb_mirror_pcextreme_nl_repo_10_3_ubuntu.list`  
After that update the apt repo.

```bash
apt update
apt install --reinstall mariadb-server galera-4
```

Maybe you need to restart the mariadb service to be sure it is running the latest version.

```bash
systemctl restart mariadb.service
```

### Disable Master-Master/Slave replication on existing env

Do not restart the node or the mariadb/mysql service until stated to do so, else stuff will break.  
Now edit the file `/etc/mysql/my.cnf`  
And comment out the following items.

```ini
[mysqld]
#server-id = ...
#auto_increment_increment = ...
#auto_increment_offset = ...
#log_bin = ...
#log_bin_index = ...
#binlog_format = ...
#sync_binlog = ...
#relay_log = ...
#relay_log_index = ...
#relay_log_info_file = ...
```

Now add the following below the [galera] header.  
Change the values of `wsrep_node_name` and `wsrep_node_address` to the correct values for that node.

```ini
[galera]
wsrep_cluster_name = close-cluster
wsrep_node_name = close-db[1|2|3]
wsrep_node_address = 192.168.4.4[0|1|2]
wsrep_on = ON
wsrep_provider = /usr/lib/galera/libgalera_smm.so
wsrep_cluster_address = "gcomm://192.168.4.40,192.168.4.41,192.168.4.42"
binlog_format = row
default_storage_engine = InnoDB
innodb_autoinc_lock_mode = 2
innodb_force_primary_key = 1
innodb_doublewrite = 1
```

We do not need the previous master-master/slave construction any more, but we want to prepare for changing the binlog_format from mixed to row which is mandatory for a galera cluster.

Also the slave(s) are going to be decommissioned and stopped.  
The following commands need to be executed using the mysql root user.

On the standby master run

```sql
STOP SLAVE;
```

On the active master run

```sql
--- Try to copy/past the 5 lines below at the same time for less interference
FLUSH TABLES WITH READ LOCK;
FLUSH LOGS;
SET GLOBAL binlog_format = 'ROW'; 
FLUSH LOGS; 
UNLOCK TABLES;

--- Now stop and clear the slave info
STOP SLAVE;
RESET SLAVE ALL;
```

On the standby master run

```sql
--- Try to copy/past the 5 lines below at the same time for less interference
FLUSH TABLES WITH READ LOCK;
FLUSH LOGS;
SET GLOBAL binlog_format = 'ROW'; 
FLUSH LOGS; 
UNLOCK TABLES;

--- Now clear the slave info
RESET SLAVE ALL;
```

On both systems check if the slave status is empty.

```sql
SHOW SLAVE STATUS \G
```

Now we have decommissioned the master-master/slave construction and are running on just one database master at this point!
You can shutdown the standby master now if you want.

```bash
systemctl stop mariadb.service
```

### Prepare for galera

Galera needs some other ports besides 3306 to work.  
For this we need to configure the firewall to allow these ports.  
For UFW execute the following commands on all the database nodes.

```bash
ufw allow from 192.168.4.0/24 to any port 4444 proto tcp
ufw allow from 192.168.4.0/24 to any port 4567 proto udp
ufw allow from 192.168.4.0/24 to any port 4567 proto tcp
ufw allow from 192.168.4.0/24 to any port 4568 proto tcp
```

Now we are going to make the current active master the base galera cluster system.  
It will bootstrap the system to be the first galera node in the cluster.  
You need to stop the current running mariadb and start it again via a special script.  
It could take a few seconds for the last command to finish, but should be relatively quick.

```bash
systemctl stop mariadb.service
galera_new_cluster
```

Now we can check if the galera cluster is up and running by connecting as root via the `mysql` client and execute the following query.

```sql
SHOW STATUS LIKE 'wsrep_cluster_%';
+----------------------------+--------------------------------------+
| Variable_name              | Value                                |
+----------------------------+--------------------------------------+
| wsrep_cluster_weight       | 1                                    |
| wsrep_cluster_capabilities |                                      |
| wsrep_cluster_conf_id      | 1                                    |
| wsrep_cluster_size         | 1                                    |
| wsrep_cluster_state_uuid   | fe6317b2-95e6-11ea-a2ae-876abee8bb23 |
| wsrep_cluster_status       | Primary                              |
+----------------------------+--------------------------------------+
6 rows in set (0.001 sec)
```

If it looks something like above all is well!  
Now we can add the other 2 database nodes into the cluster by just starting them.  
Make sure the configuration is changed as described above.  
Just run the following command on the two other database nodes.  
Please note that this restart/start command can take a long time depending on how large the database is.  
Maybe execute this command within a tmux or screen.

```bash
systemctl restart mariadb.service
```

If you are still connected with the mysql client on the first node and execute the previous query again, it should look like something below.

```sql
SHOW STATUS LIKE 'wsrep_cluster_%';
+----------------------------+--------------------------------------+
| Variable_name              | Value                                |
+----------------------------+--------------------------------------+
| wsrep_cluster_weight       | 3                                    |
| wsrep_cluster_capabilities |                                      |
| wsrep_cluster_conf_id      | 4                                    |
| wsrep_cluster_size         | 3                                    |
| wsrep_cluster_state_uuid   | fe6317b2-95e6-11ea-a2ae-876abee8bb23 |
| wsrep_cluster_status       | Primary                              |
+----------------------------+--------------------------------------+
```

## Install and Configure ProxySQL

### Install ProxySQL

ProxySQL packages can be found from there github website: https://github.com/sysown/proxysql/releases  
Download the correct version for your OS and install it using the correct tools like `dpkg` or `rpm`.

### Configure ProxySQL

Configuring ProxySQL is mostly done via the CLI using a mysql client connection.  
Some values can be configured via a file `/etc/proxysql.cnf` like the monitor user and password.  
Also the admin interface username and password and if the web-stats interface should be enabled or not.

Below some values you can add or change like the monitor_username and monitor_password.

```js
admin_variables=
{
    admin_credentials="admin:admin"
    mysql_ifaces="127.0.0.1:6032"
    refresh_interval=2000
    web_enabled=true
    web_port=6080
    stats_credentials="stats:admin"
}

mysql_variables=
{
    have_compress=false
    interfaces="127.0.0.1:3306"
    monitor_username="proxysql"
    monitor_password="S3cr3tPassword!"
    monitor_galera_healthcheck_interval=2000
    monitor_galera_healthcheck_timeout=800
}
```

The monitor user needs to be configured on all the mysql backends where proxysql is going to be connected to.  
Some make sure you have configured this with a query like below. You can of course make the host part more restrictive if you want.

```sql
--- Run this on the mysql backends
CREATE USER 'proxysql'@'%' IDENTIFIED BY 'S3cr3tPassword!';
GRANT USAGE ON *.* TO 'proxysql'@'%';
```

Now lets start and connect to the ProxySQL.

```bash
sudo systemctl start proxysql.service

mysql -u admin -padmin -h 127.0.0.1 -P6032 --prompt 'ProxySQL> '
```

You should now see a prompt that starts with `ProxySQL> `.  
Here we can configure ProxySQL.

Lets first check if all is empty, assuming you didn't configure these in the `proxysql.cnf` file.

```sql
SELECT * FROM mysql_servers;
Empty set (0.00 sec)

SELECT * FROM mysql_users;
Empty set (0.00 sec)

SELECT * from mysql_galera_hostgroups;
Empty set (0.00 sec)

SELECT * from mysql_query_rules;
Empty set (0.00 sec)
```

If all is empty, lets first configure the servers it needs to connect to.

```sql
INSERT INTO mysql_servers (hostgroup_id,hostname,port) values (10,'192.168.4.40',3306);
INSERT INTO mysql_servers (hostgroup_id,hostname,port) values (10,'192.168.4.41',3306);
INSERT INTO mysql_servers (hostgroup_id,hostname,port) values (10,'192.168.4.42',3306);

--- Check the inserts
SELECT * FROM mysql_servers;
```

If all is good, then you can first load the configuration into the runtime environment and afterwards save it to disks.  
ProxySQL first stores everything in the current connected session so you can verify it before making it active by loading it into the runtime environment. If all works well, then save it to disk to be persistent.

```sql
--- Make the configuration active
LOAD MYSQL SERVERS TO RUNTIME;

--- Save to disk
SAVE MYSQL SERVERS TO DISK;
```

You should now also see some connection statistics if ProxySQL is able to connect to the backends or not.

```sql
--- Check connection stats
SELECT * FROM monitor.mysql_server_connect_log ORDER BY time_start_us DESC LIMIT 6;
SELECT * FROM monitor.mysql_server_ping_log ORDER BY time_start_us DESC LIMIT 6;
```

If that is working we can configure the galera hostgroups so that ProxySQL knows this is a galera cluster.  
The only thing we need to do is add a line to the mysql_galera_hostgroups table.   
This will use hostgroup `10` as it's default writer connection which is the hostgroup used when adding the mysql servers above.  
ProxySQL will create the other hostgroups by it's self.

```sql
INSERT INTO mysql_galera_hostgroups (writer_hostgroup,backup_writer_hostgroup,reader_hostgroup,
	offline_hostgroup,active,max_writers,writer_is_also_reader,max_transactions_behind) 
VALUES (10, 20, 30, 666, 1, 1, 2, 20);

--- Now save this also to runtime and disk
LOAD MYSQL SERVERS TO RUNTIME;
SAVE MYSQL SERVERS TO DISK;
```

When this is configured you should be able to see the generated hostgroups by executing the following query.

```sql
SELECT * FROM stats.stats_mysql_connection_pool;
```

Now the only thing left to do is to add the database user with the same credentials as on the backends to ProxySQL.  
This because ProxySQL needs to know this before allowing any connection and determine the hostgroup and query routing.

```sql
INSERT INTO mysql_users(username,password,default_hostgroup,transaction_persistent) 
VALUES ('{username}','{password}', 10, 1);

--- Load to runtime
LOAD MYSQL USERS TO RUNTIME;

-- Save from runtime - This will make the password stored via the default mysql password hash method.
SAVE MYSQL USERS FROM RUNTIME;

-- Save to disk
SAVE MYSQL USERS TO DISK;

-- Check the created user record
SELECT * FROM mysql_users;
```

Now you should be able to connect to ProxySQL using the username and password just entered and run queries on the backends.  
Because ProxySQL has built-in galera cluster checking since version 2.x it will automatically failover to a different backend if one server dies. The only thing that could happen is that any active connection to that backend will be disconnected, but any following requests should be redirected to one of the other backends.

After running some queries check the stats again.

```sql
SELECT * FROM stats.stats_mysql_connection_pool;
```

### Adding special query routing rules

If you want to route specific SELECT queries to always use the reader hostgroup instead of the master to spread the load you can add a query like this.  
These rules are sorted by there rule_id so keep that in mind when adding them. There fore it is save to start with a high number like 100 so that you can add some rules in front of it. Else you would need to reshuffle them.

```sql
INSERT INTO mysql_query_rules(rule_id,active,match_pattern,destination_hostgroup,apply) 
    VALUES (200, 1, "^SELECT .* FROM <TABLENAME>", 20, 1)
LOAD MYSQL QUERY RULES TO RUNTIME;
SAVE MYSQL QUERY RULES TO DISK;
```

Or if you want to have all selects to be routed to the reader hostgroup.

```sql
INSERT INTO mysql_query_rules(rule_id,active,match_pattern,destination_hostgroup,apply) 
    VALUES (200, 1, "^SELECT .* FROM", 20, 1)
LOAD MYSQL QUERY RULES TO RUNTIME;
SAVE MYSQL QUERY RULES TO DISK;
```

There are some special queries which you probably always want to execute on the master writer, like queries which use the `FOR UPDATE` statement.  
Just execute the following.

```sql
INSERT INTO mysql_query_rules(rule_id,active,match_pattern,destination_hostgroup,apply) 
    VALUES (100, 1, "^SELECT .* FOR UPDATE", 10, 1);
LOAD MYSQL QUERY RULES TO RUNTIME;
SAVE MYSQL QUERY RULES TO DISK;
```

And just to be sure to route all other queries to the current master. Execute the following.

```sql
INSERT INTO mysql_query_rules(rule_id,active,match_pattern,destination_hostgroup,apply) 
    VALUES (300, 1, ".*", 10, 1);
LOAD MYSQL QUERY RULES TO RUNTIME;
SAVE MYSQL QUERY RULES TO DISK;
```

You can also specify the username which is used during the connection to have only that user routed to a different system.  
Just add the username into the query as a column to update.

```sql
INSERT INTO mysql_query_rules(rule_id,active,username,match_pattern,destination_hostgroup,apply) 
    VALUES (190, 1, "cron-user", "^SELECT .* FROM", 20, 1);
LOAD MYSQL QUERY RULES TO RUNTIME;
SAVE MYSQL QUERY RULES TO DISK;
```
