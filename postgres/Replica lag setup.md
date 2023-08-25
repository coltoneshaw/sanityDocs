## Enable Replica Lag logging in Mattermost

1. Add to the Mattermost config.

    Within the Mattermost config.json, add the below to the `SqlSettings.ReplicaLagSettings`. The query will pull the diff between the current WAL and the replica. The `DataSource` string should point to your primary database.

    - `username` - Is the PostgreSQL username with permissions to the `mattermost` database. Usually, this is `mmuser`.
    - `password` - Password assigned to the `username` from above.
    - `connectionString` - IP address or URL of the PostgresSQL instance. It doesn't require `:5432`.

    ```JSON
    {
    "SqlSettings": {
        "ReplicaLagSettings": [
        {
            "DataSource": "postgres://username:password@connectionString/mattermost?sslmode=disable\u0026connect_timeout=10",
            "QueryAbsoluteLag": "select usename, pg_wal_lsn_diff(pg_current_wal_lsn(),replay_lsn) as metric from pg_stat_replication;",
            "QueryTimeLag": null
        }
        ]
    }
    }
    ```

2. Give permissions

    Before restarting, give the `mmuser` access to the PostgreSQL monitoring role. Updating permissions should be done on the PostgreSQL primary database, not the replica.

    [PostgreSQL Role documentation](https://www.postgresql.org/docs/10/default-roles.html)

    ```
    sudo -u postgres psql
    postgres=# GRANT pg_monitor TO mmuser;
    ```

3. Update chart and restart Mattermost.

    In the `Mattermost Performance Monitoring v2`, update the `Replica Lag` chart to use the value `mattermost_db_replica_lag_abs` instead of the current time value.

    The query should be the below:

    ```
    mattermost_db_replica_lag_abs{instance=~"$server"}
    ```

4. Restart Mattermost

    You should see this go into effect. Check the logs for any query errors.

    ```
    sudo systemctl restart mattermost
    ```