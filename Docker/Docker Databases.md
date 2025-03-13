up:: [[Docker]]
tags:: #linux #docker #note/info/misc #on/readme #on/docker 
description:: A quick run-down on upgrading PostgreSQL major versions running in Docker on Linux
# Upgrading Postgres on Docker
Dump the existing database to an SQL file thus: 
```docker compose exec <<db-container>> pg_dump -U <<pg_user>> -d <<pg_database>> -cC > upgrade_backup_12.sql```

- db-container is the container name of the postgres instance (usually db or postgresql depending on the stack)
- <<pg_user>> is the username used by the stack (i.e. 'authentik' or 'paperless')
- <<pg_database>> is the database name
- upgrade_backup_12.sql is the SQL dump with the numbers indicating *the original Postgres version*

This outputs a full SQL file to the local host. Make sure to verify the contents of this file!

Stop the Postgres container - ```docker compose down``` (this prevents any connections to the db)

Create a backup volume (just in case):
- docker volume create *volume_name* && docker run --rm -v *original_volume*:/from -v *backup_volume*:/to alpine sh -c 'cd /from && cp -a . /to'

++++ Command Breakdown:
- docker volume create
	- Creates a docker volume named *volume_name* (it is useful to use the stack name - i.e. paperless) with backup as a suffix to show it is a backup volume
- && docker run --rm
	- immediately run a new container to be removed once the action is complete
- -v *original_volume*:/from
	- mount the original volume (i.e. paperless_pgdata) to /from in the temporary container
- -v *backup_volume*:/to
	- mount the backup volume (i.e. paperless_pgdata_backup) to /to in the temporary container
- alpine
	- run an Alpine linux container
- sh -c 'cd /from && cp -a . /to'
	- Initiate a shell and change directory to the *original_volume* then CoPy __everything__ to the /to directory (which is the *backup_volume*)

Next, __delete__ the *original_volume* with the command ```docker volume rm -f original_volume```

Recreate the database container (i.e. Postgres 16) with ```docker compose up -d db_container```

Apply the backup SQL to the newly created container:
- ```cat upgrade_backup_12.sql | docker compose exec -T db_container psql -U db_user```

This reads the backup SQL dump to re-create the database in the new container and applies any schema changes. Once complete, the rest of the stack can be started.