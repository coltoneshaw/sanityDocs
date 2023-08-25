## Config Suggestions

If the database has less than 12GB RAM, reduce the below settings.

Note that you can set `shared_buffers` and `effective_cache_size` higher on the primary database because they generally receive less traffic.

### Primary Database

```bash
# This value should match the "SqlSettings.MaxConnections" value within your config.json for Mattermost
# This is a suggestion and can be set lower / higher based on the size of your server.
max_connections = 1020
tcp_keepalives_idle = 5
tcp_keepalives_interval = 1
tcp_keepalives_count = 5

# Set both of the below settings to 65% of total memory. For a 32 GB instance, it should be 21 GB.
# If on a smaller server, set this to 20% or less total RAM.
# ex: 512MB would work for a 4GB RAM server
shared_buffers = 512MB
effective_cache_size = 512MB

# Set it to 16 MB for readers and 32 MB for writers. If it's a single instance, 16 MB should be sufficient. If the instance is of a lower capacity than r5.xlarge, then set it to a lower number.
work_mem = 16MB

# 1GB (reduce this to 512MB if your server has less than 32GB of RAM)
autovacuum_work_mem = 512MB
autovacuum_max_workers = 4
autovacuum_vacuum_cost_limit = 500

#Set it to 1.1 unless the DB is using spinning disks.
random_page_cost = 1.1

restart_after_crash = on
```

### Replica Database

**In addition to the above settings**

```bash
# connection string to sending server
# This was created when you added a replication role to the primary database.
# username - replace "test" with the role you made
# password - replace "testpassword" with the role password
# host - replace "x.x.x.x" with your IP or URL string.
primary_conninfo = 'user=test password=testpassword host=x.x.x.x port=5432 sslmode=prefer sslcompression=0 gssencmode=prefer krbsrvname=postgres target_session_attrs=any'

# replication slot on sending server
# This needs to be configured using the docs below for "Keeping the replica and primary in sync."
primary_slot_name = 'replica1'

hot_standby = on
# Allows query communication between reader and primary. Suggested to prevent any timeouts.
hot_standby_feedback = on
```
