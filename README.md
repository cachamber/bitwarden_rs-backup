# bitwarden_rs Backup
Docker Containers for [bitwarden_rs](https://github.com/dani-garcia/bitwarden_rs) Backup.

## Usage
The default tag `latest` should be used for a x86-64 system. If you try to run the container on a raspberry pi 3 you should use the tag `rpi3`. Also make sure that your **bitwarden_rs container is named `bitwarden`** otherwise you have to replace the container name in the `--volumes-from` section of the `docker run` call.

### Automatic Backups 
A cron daemon is running inside the container and the container keeps running in background.

Start backup container with default settings (automatic backup at 5 am)
```sh
docker run -d --restart=always --name bitwarden_backup --volumes-from=bitwarden bruceforce/bw_backup
```

Example for hourly backups
```sh
docker run -d --restart=always --name bitwarden_backup --volumes-from=bitwarden -e CRON_TIME="0 * * * *" bruceforce/bw_backup
```

Example for backups that delete after 30 days
```sh
docker run -d --restart=always --name bitwarden_backup --volumes-from=bitwarden -e DELETE_AFTER=30 bruceforce/bw_backup
```

### Manual Backups
You can use the crontab of your host to schedule the backup and the container will only be running during the backup process.

```sh
docker run --rm --volumes-from=bitwarden --entrypoint sqlite3 bruceforce/bw_backup $DB_FILE ".backup $BACKUP_FILE"
```

Keep in mind that the above command will be executed inside the container. So
- `$DB_FILE` is the path to the bitwarden database which is normally locatated at `/data/db.sqlite3`
- `$BACKUP_FILE` can be any place inside the container. Easiest would be to set it to `/data/backup.sqlite3` which will create the backup near the original database file.
If you want the backed up file to be stored outside the container you have to mount
a directory by adding `-v <PATH_ON_YOUR_HOST>:<PATH_INSIDE_CONTAINER>`. The complete command could look like this

```sh
docker run --rm --volumes-from=bitwarden -v /tmp/myBackup:/myBackup --entrypoint sqlite3 bruceforce/bw_backup /data/db.sqlite3 ".backup /myBackup/backup.sqlite3"
```

## Environment variables
| ENV | Description |
| ----- | ----- |
| DB_FILE | Path to the Bitwarden sqlite3 database *inside* the container |
| BACKUP_FILE | Path to the desired backup location *inside* the container |
| CRON_TIME | Cronjob format "Minute Hour Day_of_month Month_of_year Day_of_week Year" |
| TIMESTAMP | Set to `true` to append timestamp to the `BACKUP_FILE` |
| UID | User ID to run the cron job with |
| GID | Group ID to run the cron job with |
| LOGFILE | Path to the logfile *inside* the container |
| CRONFILE | Path to the cron file *inside* the container |
| DELETE_AFTER | Delete old backups after X many days |
| COMPRESS | Compress database backups |

## Common erros
### Wrong permissions
`Error: unable to open database file` is most likely caused by permission errors.
Note that sqlite3 creates a lock file in the source directory while running the backup.
So source *AND* destination have to be +rw for the user. You can set the user and group ID
via the `UID` and `GID` environment variables like described above.

### Wrong timestamp
If you need timestamps in your local timezone you should mount `/etc/timezone:/etc/timezone:ro` and `/etc/localtime:/etc/localtime:ro`
like it's done in the [docker-compose.yml](docker-compose.yml).
