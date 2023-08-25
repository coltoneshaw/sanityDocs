
## Perform manual database migrations

> ⚠️ Warning that this is not an officially supported process with Mattermost and done at your own risk. Backup the database before any migrations are done.


### Channels

> Note: Technically, you _can_ only perform some of the migrations and have Mattermost fill in the remainder. This is however untested and can have side effects if a later migration requires a change in a prior migration to have occured. 

1. Before you upgrade check the `db_migrations` table to see what the last migration that completed was.
2. Navigate to the [mattermost server github](https://github.com/mattermost/mattermost-server/tree/master/db/migrations/postgres) and change the branch to the one you’re upgrading to. This will look like `release-7.2` for example.
3. Identify what migrations have not been completed based on their number.
    - Mattermost will attempt to run any migration in this folder that does not appear in the `db_migrations` table.
4. Fully stop the Mattermost server
5. Manually run each migration ending with `.up.sql`.
    - Some migrations can take longer, or have multiple steps involved. Make sure each step within a migration is run successfully. 
6. When the migrations are complete you’ll need to manually insert the migrations into the database `db_migrations` table.
    - The migration name is the name of the sql file in step 4 without the `.up.sql`. Example: Migration 90 is labeled `000090_create_enums.up.sql`. The migration number is `90` and the name is `create_enums`.
    - Insert Command `INSERT INTO db_migrations (version, name) VALUES (90, 'create_enums');`
7. Upgrade the Mattermost server and turn it back on.

### Playbooks

> ⚠️ Still testing, do not use yet.

1. Before you upgrade check the `ir_db_migrations` table to see what the last migration was. 
2. Identify what the upgraded version of Playbooks is.
    - This is usually found within the Self Hosted changelog under `Improvements`. [Example for v7.1](https://docs.mattermost.com/install/self-managed-changelog.html#id14).
    - If you want further verification of this you can download the mattermost tar [here](https://mattermost.com/deploy/) and check within `prepackaged_plugins` for the version included.
3. Navigate to the database migrations for playbooks [here](https://github.com/mattermost/mattermost-plugin-playbooks/tree/master/server/sqlstore/migrations/postgres) and filter for the correct release you'll be upgrading to.
    - Example: Mattermost v7.3.0 includes Playbooks `1.32.x` so you'll filter for `release-1.32`
Identify what migrations have not been completed based on their number.
    - Mattermost will attempt to run any migration in this folder that does not appear in the `ir_db_migrations` table.
4. Fully stop the Mattermost server or disable the Playbooks plugin
5. Manually run each migration ending with `.up.sql`.
    - Some migrations can take longer, or have multiple steps involved. Make sure each step within a migration is run successfully. 
6. When the migrations are complete you’ll need to manually insert the migrations into the database `ir_db_migrations` table.
    - The migration name is the name of the sql file in step 4 without the `.up.sql`. Example: Migration 27 is labeled `000023_add_default_commander_enabled_to_ir_playbook.up.sql`. The migration number is `23` and the name is `add_default_commander_enabled_to_ir_playbook`.
    - Insert Command `INSERT INTO ir_db_migrations (version, name) VALUES (23, 'add_default_commander_enabled_to_ir_playbook');`
7. Upgrade the Mattermost server and turn it back on.

### Boards

1. Before you upgrade check the `focalboard_schema_migrations` table to see what the last migration was. 
   - If you see the columns as `version` and `dirty` this will require Mattermost to upgrade the database before manual migrations can happen. 
2. Identify what the upgraded version of Boards is.
    - This is usually found within the Self Hosted changelog under `Improvements`. [Example for v7.1](https://docs.mattermost.com/install/self-managed-changelog.html#id14).
    - If you want further verification of this you can download the mattermost tar [here](https://mattermost.com/deploy/) and check within `prepackaged_plugins` for the version included.
3. Navigate to the database migrations for boards [here](https://github.com/mattermost/focalboard/tree/release-7.2/server/services/store/sqlstore/migrations) and filter for the correct release you'll be upgrading to.
    - Boards follows a similar version structure as Mattermost Server, so you'll search for `release-7.2` for the Mattermost 7.2 release.
4. Identify what migrations have not been completed based on their number.
    - Mattermost will attempt to run any migration in this folder that does not appear in the `focalboard_schema_migrations` table.
4. Fully stop the Mattermost server or disable the Boards plugin
5. Manually run each migration ending with `.up.sql`.
    - Some migrations can take longer, or have multiple steps involved. Make sure each step within a migration is run successfully. 
    - Anytime you see `{{prefix}}` you'll append `focalboard_` to this query.
6. When the migrations are complete you’ll need to manually insert the migrations into the database `focalboard_schema_migrations` table.
    - The migration name is the name of the sql file in step 4 without the `.up.sql`. Example: Migration 25 is labeled `000025_indexes_update.up.sql`. The migration number is `25` and the name is `indexes_update`.
    - Insert Command `INSERT INTO focalboard_schema_migrations (version, name) VALUES (25, 'indexes_update');`
7. Upgrade the Boards plugin through the Marketplace, manually uploading the new version, or upgrading Mattermost.