
## Setup

### Setting up the primary

1. Create the replica role

    We are creating a role on your primary to allow an incoming connection to replicate the database.

    - `username` replace this with the username you want your replica to use to connect to the primary.
    - `password` replace with your replica connection password.

    ```sql
    sudo -u postgres psql
    postgres=# CREATE ROLE username WITH REPLICATION PASSWORD 'password' LOGIN;
    Output
    CREATE ROLE
    ```

2. Create a replication slot

    You can replace `replica1` with anything you want to identify your read replica. 
    Note: that this needs to be a unique name per replica you connect.

    ```sql
    select pg_create_physical_replication_slot(‘replica1’);
    ```

3. Allow the replica communication

    Edit the `pg_hba.conf` file on the **primary** database to allow incoming communication via the configured above username

    ```bash
    /etc/postgresql/12/main/pg_hba.conf
    ```

    - `username` - This is the role username configured in step 1 above.
    - `replica-IP` - This is the IP address of your replica.

    ```bash
    ...
    host    replication     username    replica-IP/32   md5
    ```

4. Restart Postgres

    ```bash
    sudo systemctl restart postgresql
    ```

### Setup the Replica

1. Install Postgres. Note it must be the same version as primary.

2. Clear the replica data directory.

    2a. Find the data directory first by the command below:

      ```sql
      sudo -u postgres psql
      postgres=#SHOW data_directory;
          data_directory
      -----------------------------
      /var/lib/postgresql/12/main
      (1 row)
      ```

    2b. Delete the contents of the data directory on the **replica**.

    ```bash
    sudo -u postgres rm -r /var/lib/postgresql/12/main
    sudo -u postgres mkdir /var/lib/postgresql/12/main
    sudo -u postgres chmod 700 /var/lib/postgresql/12/main
    ```

3. Backup the primary database onto the replica

    The backup will take some time, depending on the database size. Testing with a 16GB database took ~4 minutes.

    - The `-U username` option allows you to specify the user you connect to the **primary** cluster as. This is the role you created in the previous step.
    - It will prompt for a password. This will be what was specified in the prior step

    ```bash
    sudo -u postgres pg_basebackup -h primary-ip-addr -p 5432 -U username -D /var/lib/postgresql/12/main/ -Fp -Xs -R
    ```

4. Edit the read replica `postgres.conf` file to use the slot name.

    This `primary_slot_name` is the same slot you configured above in Step 2 of setting up the primary.

    File location - `/etc/postgres/12/main/postgres.conf`

    ```bash
    primary_slot_name = 'replica1'
    ```

5. Restart postgres

    ```bash
    sudo systemctl restart postgresql
    ```

6. You should see a log like below on the **replica**.

  ```
   started streaming WAL from primary at 0/12000000 on timeline 1
  ```

### Verify connectivity

1. Connect to PostgreSQL on the **primary** server.

    ```sql
    sudo -u postgres psql
    ```

2. Copy and run the below command

    ```sql
    SELECT client_addr, state, pid, slot_name, active
    FROM pg_stat_replication
    INNER JOIN pg_replication_slots on pid = pg_replication_slots.active_pid;
    ```

    You should see:

    ```sql
    client_addr  |   state   |  pid   | slot_name | active
    ---------------+-----------+--------+-----------+--------
    x.x.x.x       | streaming | 116774 | replica1   | t
    ```

### Replica Lag

Next step is to setup replica lag and see the docs. 


## Useful commands

### On Primary
- To view existing replication slots - `select * from pg_replication_slots;`
- Check replication slot lag - `SELECT redo_lsn, slot_name,restart_lsn,`
- Remove a replication slot - `select pg_drop_replication_slot(‘ocean’);`

## Troubleshooting

### Replica has fallen out of sync

If a read replica fails, when it attempts to recover, it will be unable to because the primary database will not have stored the WAL log it requires to sync back up. You'll see the below in the logs usually if this happens.

```
2022-03-25 04:38:27.369 UTC [77723] mmuser@mattermost FATAL:  the database system is in recovery mode
...
2022-03-25 04:44:12.456 UTC [77947] LOG:  entering standby mode
2022-03-25 04:44:12.465 UTC [77947] LOG:  redo starts at 6/5FA18838
2022-03-25 04:44:12.969 UTC [77950] mmuser@mattermost FATAL:  the database system is starting up
2022-03-25 04:44:12.971 UTC [77951] mmuser@mattermost FATAL:  the database system is starting up
2022-03-25 04:44:13.574 UTC [77947] LOG:  consistent recovery state reached at 6/6DA47060
2022-03-25 04:44:13.575 UTC [77946] LOG:  database system is ready to accept read only connections
2022-03-25 04:44:23.290 UTC [77947] LOG:  invalid resource manager ID 215 at 6/7236A070
2022-03-25 04:44:23.306 UTC [77975] LOG:  started streaming WAL from primary at 6/72000000 on timeline 1
2022-03-25 04:44:23.306 UTC [77975] FATAL:  could not receive data from WAL stream: ERROR:  requested WAL segment 000000010000000600000072 has already been removed
```

You can run the below query and get the output of replication on the **primary**. If this is empty, then no streaming is currently happening.

```sql
postgres=# select * from pg_stat_replication \x\g\x
-[ RECORD 1 ]----+------------------------------
pid              | 4643
usesysid         | 16384
usename          | test
application_name | 12/main
client_addr      | 172.31.70.183
client_hostname  |
client_port      | 41482
backend_start    | 2022-03-28 13:11:48.668763+00
backend_xmin     |
state            | streaming
sent_lsn         | 14/61502C98
write_lsn        | 14/61502C98
flush_lsn        | 14/61502C98
replay_lsn       | 14/61502C98
write_lag        |
flush_lag        |
replay_lag       |
sync_priority    | 0
sync_state       | async
reply_time       | 2022-03-28 13:40:28.312833+00
```

Check how bad the delay is between the primary and replica. The higher the number, the more out of sync the replica is. The query will return the difference in bytes for how out of sync it is. 

So, `200000000` will be `200MB` out of sync. Ideally, this metric is below `10MB` out of sync, and when it starts to reach high numbers, that usually means the replica has gone down and requires a restart.

```sql
select usename, client_addr, state, pg_wal_lsn_diff(pg_current_wal_lsn(),replay_lsn) as delay from pg_stat_replication;
 usename |  client_addr  |   state   | delay
---------+---------------+-----------+-------
 test    | 172.31.70.183 | streaming |     0
(1 row)
```


### Resyncing a Replica

Usually, this is as simple as just restarting the PostgreSQL service on the read replica. If the read replica was misconfigured or fallen too far behind the WAL archive on the primary, you might need to rebuild the replica's database with the below commands.

1. Delete all the postgres data on the **read replica**.

    ```bash
    sudo -u postgres rm -r /var/lib/postgresql/12/main
    sudo -u postgres mkdir /var/lib/postgresql/12/main
    sudo -u postgres chmod 700 /var/lib/postgresql/12/main
    ```

2. Redownload the data from the primary database.

    Use this command on the **replica**.

    - `-h primary-ip-addr` - This will be the IP address of your primary database.
    - `-U username` - the `username` here is the role that you configured within the primary database for the replica to connect with. You can check the name by using `\du+` on the **primary** database.
    - It will prompt for the password you configured for that role. 

    ```
    sudo -u postgres pg_basebackup -h primary-ip-addr -p 5432 -U username -D /var/lib/postgresql/12/main/ -Fp -Xs -R
    ```

    - `-U test` is the replication role configured on the **primary** to accept communications. If you do not remember the role name, you can run `\du+` on the **primary** database.
    - `-D /var/lib/postgresql/12/main/` this tells the command where to output the backup.
    - `-h primary-ip-addr` The IP address of the primary database.

3. Restart Postgres on the read replica.

    ```bash
    sudo systemctl restart postgresql
    ```

    If PostgreSQL fails to start, use `sudo tail -n 300 /var/log/postgresql/*` to check what issues occurred during startup.
