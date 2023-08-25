

## Check Connection

`pg_isready` is a utility designed to check if your box can access the PostgreSQL install.

1. Install `pg_isready`.

    Replace `12` with the version string of your PostgreSQL instance.

    **Example for Postgres 12**

    ```bash
    sudo apt-get install postgresql-client-common postgresql-client-12
    ```

2. Test the connection

    This will reach out to the PostgreSQL instance running at the below and make sure it's accepting connections properly.

    You can find all this data in the mattermost `config.json` file. If you run the below you can use most of the data from the `DataSource` string to fill this in.

    ```bash
    sudo grep "SqlSettings" -A 10 /opt/mattermost/config/config.json
    ```

    ```bash
    pg_isready -d <db_name> -h <host_name> -p <port_number> -U <db_user>
    ```

    Expected Response:

    ```
    accepting connections
    ```