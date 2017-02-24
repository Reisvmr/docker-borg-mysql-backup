# borg-mysql-backup

This image is mainly intended for backing up mysql dumps to local and/or remote [borg](https://github.com/borgbackup/borg) repos.
Other files may be included in the backup, and database dumps can be excluded altogether.

The container features `backup` and `restore` scripts that can either be ran directly with docker,
or by cron, latter being the preferred method.

For cron and/or remote borg usage, you also need to mount container configuration
at `/config`, containing ssh key (named `id_rsa`) and/or crontab file (named `crontab`).

Local borg repo will be located under the dir you mount container `/backup` at.
Remote borg repo needs to be initialised manually beforehand.

In case some containers need to be stopped for the backup (eg to assert there won't be any
mismatch between data and database), you can specify those container names to
the `backup` script (see below). Note that this requires mounting docker socket with `-v /var/run/docker.sock:/var/run/docker.sock`,
but keep in mind it has security implications (borg-mysql-backup will have essentially
root permissions on the host).


## Container Parameters

    MYSQL_HOST              the host/ip of your mysql database
    MYSQL_PORT              the port number of your mysql database
    MYSQL_USER              the username of your mysql database
    MYSQL_PASS              the password of your mysql database
    MYSQL_EXTRA_OPTS        the extra options to pass to `mysqldump` command; optional
      mysql env variables are only required if you intend to back up databases


    HOST_HOSTNAME           hostname to include in the borg archive name;
                            defaults to container's hostname (which is not really descriptive)
    REMOTE                  remote connection, including repository, eg remoteuser@remoteserver.com:/backup/location
                            optional - can be omitted when only backing up to local borg repo.
    BORG_LOCAL_REPO_NAME    local borg repo name; optional, defaults to 'repo'
    BORG_EXTRA_OPTS         additional borg params (for both local & remote borg commands); optional
    BORG_LOCAL_EXTRA_OPTS   additional borg params for local borg command; optional
    BORG_REMOTE_EXTRA_OPTS  additional borg params for remote borg command; optional
    BORG_PASSPHRASE         borg repo password
    BORG_PRUNE_OPTS         options for borg prune (both local and remote); not required when
                            restoring or if it's defined by backup script -P param
                            (which overrides this container env var)

## Script usage

Container incorporates `backup` and `restore` scripts.

### backup

`backup` script is mostly intended to be ran by the cron, but can also be executed
directly via docker for one off backup.

    usage: backup [-h] [-d MYSQL_DBS] [-n NODES_TO_BACKUP] [-c CONTAINERS]
                  [-r] [-l] [-P BORG_PRUNE_OPTS] [-N BORG_LOCAL_REPO_NAME] -p PREFIX
    
    Create new archive
    
    arguments:
      -h                      show help and exit
      -d MYSQL_DBS            space separated database names to back up; __all__ to back up
                              all dbs on the server
      -n NODES_TO_BACKUP      space separated files/directories to back up (in addition to db dumps);
                              filenames may not contain spaces, as space is the separator
      -c CONTAINERS           space separated container names to stop for the backup process;
                              requires mounting the docker socket (-v /var/run/docker.sock:/var/run/docker.sock)
      -r                      only back to remote borg repo (remote-only)
      -l                      only back to local borg repo (local-only)
      -P BORG_PRUNE_OPTS      overrides container env variable BORG_PRUNE_OPTS; only required when
                              container var is not defined;
      -N BORG_LOCAL_REPO_NAME overrides container env variable BORG_LOCAL_REPO_NAME; optional;
      -p PREFIX               borg archive name prefix. note that the full archive name already
                              contains hostname and timestamp.

#### Usage examples

##### Back up App1 & App2 databases and app1's data directory /app1-data daily at 05:15 to both local and remote borg repos

    docker run -d \
        -e MYSQL_HOST=mysql.host \
        -e MYSQL_PORT=27017 \
        -e MYSQL_USER=admin \
        -e MYSQL_PASS=password \
        -e HOST_HOSTNAME=hostname-to-use-in-archive-prefix \
        -e REMOTE=remoteuser@server.com:repo/location \
        -e BORG_EXTRA_OPTS='--compression zlib,5 --lock-wait 60' \
        -e BORG_PASSPHRASE=borgrepopassword \
        -e BORG_PRUNE_OPTS='--keep-daily=7 --keep-weekly=4' \
        -v /backup:/backup:ro \
        -v /borg-mysql-backup/config:/config:ro \
        -v /app1-data-on-host:/app1-data:ro \
           layr/borg-mysql-backup

`/config/crontab` contents:

    15 05 * * * root  backup -d "App1 App2" -n /app1-data -p app1-app2

##### Back up all databases daily at 04:10 and 16:10 to local&remote borg repos, stopping containers myapp1 & myapp2 for the process

    docker run -d \
        -e MYSQL_HOST=mysql.host \
        -e MYSQL_PORT=27017 \
        -e MYSQL_USER=admin \
        -e MYSQL_PASS=password \
        -e HOST_HOSTNAME=hostname-to-use-in-archive-prefix \
        -e REMOTE=remoteuser@server.com:repo/location \
        -e BORG_PASSPHRASE=borgrepopassword \
        -e BORG_PRUNE_OPTS='--keep-daily=7 --keep-weekly=4' \
        -v /var/run/docker.sock:/var/run/docker.sock \
        -v /backup:/backup:ro \
        -v /borg-mysql-backup/config:/config:ro \
           layr/borg-mysql-backup

`/config/crontab` contents:

    10 04,16 * * * root  backup -d __all__ -p myapp-prefix -c "myapp1 myapp2"

##### Back up directores /app1 & /app2 every 6 hours to local borg repo (ie remote is excluded)

    docker run -d \
        -e HOST_HOSTNAME=hostname-to-use-in-archive-prefix \
        -e BORG_PASSPHRASE=borgrepopassword \
        -e BORG_PRUNE_OPTS='--keep-daily=7 --keep-weekly=4' \
        -v /backup:/backup:ro \
        -v /borg-mysql-backup/config:/config:ro \
        -v /app1:/app1:ro \
        -v /app2:/app2:ro \
           layr/borg-mysql-backup

`/config/crontab` contents:

    0 */6 * * * root  backup -l -n "/app1 /app2" -p my_app_prefix

Note we didn't need to define mysql- or remote borg repo related docker env vars.
Also there's no need to have ssh key in `/config`, as we're not connecting to a remote server.

##### Back up directory /app3 once to remote borg repo (ie local is excluded)

    docker run -it --rm \
        -e HOST_HOSTNAME=hostname-to-use-in-archive-prefix \
        -e REMOTE=remoteuser@server.com:repo/location \
        -e BORG_PASSPHRASE=borgrepopassword \
        -e BORG_PRUNE_OPTS='--keep-daily=7 --keep-weekly=4' \
        -v /backup:/backup:ro \
        -v /borg-mysql-backup/config:/config:ro \
        -v /app3/on/host:/app3:ro \
           layr/borg-mysql-backup -- backup -r -n /app3 -p my_prefix

Note there's no need to have a crontab file in `/config`, as container only lives until
the `backup` command returns.

### restore

`restore` script should be executed directly with docker in interactive mode.
Script will restore db from the restored dump, if respective param is provided. All data
will be extracted into `/backup/restored-{archive_name}`

Repository contents can be listed with the `-L` option, or simply run directly `borg list`
instead of `restore`

    usage: restore [-h] [-d] [-c CONTAINERS] [-r] [-l] [-L]
                   [-N BORG_LOCAL_REPO_NAME] -a ARCHIVE_NAME
    
    Restore data from borg archive
    
    arguments:
      -h                      show help and exit
      -d                      restore mysql database from dumped file
      -c CONTAINERS           space separated container names to stop before the restore begins;
                              note they won't be started afterwards, as there might be need
                              to restore other data (only sql dumps are restored automatically);
                              requires mounting the docker socket (-v /var/run/docker.sock:/var/run/docker.sock)
      -r                      restore from remote borg repo
      -l                      restore from local borg repo
      -L                      instead of restoring, simply list the repository contents and exit;
                              in this case, naturally, -a option is not mandatory
      -N BORG_LOCAL_REPO_NAME overrides container env variable BORG_LOCAL_REPO_NAME; optional;
      -a ARCHIVE_NAME         name of the borg archive to restore data from

#### Usage examples

##### List the local repository contents

    docker run -it --rm \
        -e BORG_PASSPHRASE=borgrepopassword \
        -v /tmp:/backup:ro \
           layr/borg-mysql-backup -- restore -lL

##### List the remote repository contents

    docker run -it --rm \
        -e BORG_EXTRA_OPTS="-P my-prefix" \
        -e REMOTE=remoteuser@server.com:repo/location \
        -e BORG_PASSPHRASE=borgrepopassword \
        -v /borg-mysql-backup/config:/config:ro \
           layr/borg-mysql-backup -- restore -rL

Note the `BORG_EXTRA_OPTS`, `BORG_LOCAL_EXTRA_OPTS`, `BORG_REMOTE_EXTRA_OPTS` env
variables are still usable with -L option.

##### Restore archive from remote borg repo & restore mysql with restored dumpfile

    docker run -it --rm \
        -e MYSQL_HOST=mysql.host \
        -e MYSQL_PORT=27017 \
        -e MYSQL_USER=admin \
        -e MYSQL_PASS=password \
        -e REMOTE=remoteuser@server.com:repo/location \
        -e BORG_PASSPHRASE=borgrepopassword \
        -v /tmp:/backup:ro \
        -v /borg-mysql-backup/config:/config:ro \
           layr/borg-mysql-backup -- restore -r -d -a my_prefix-HOSTNAME-2017-02-27-160159

##### Restore archive from local borg repo & stop container app1 beforehand

    docker run -it --rm \
        -e BORG_PASSPHRASE=borgrepopassword \
        -v /tmp:/backup:ro \
           layr/borg-mysql-backup -- restore -l -c app1 -a my_prefix-HOSTNAME-2017-02-27-160159

Note there's no need to mount `/config`, as we're not using cron nor connecting to remote borg.
Also we're not providing mysql env vars, as script isn't invoked with `-d` option, meaning
db won't be automatically restored with the included .sql dumpfile (if there was one).

##### Restore archive from non-default local borg repo

    docker run -it --rm \
        -v /tmp:/backup:ro \
           layr/borg-mysql-backup -- restore -l -N otherrepo -a my_prefix-HOSTNAME-2017-02-27-160159

Data will be restored from non-default local borg repo `otherrepo`. Also note missing `BORG_PASSPHRASE`,
which is required to be typed in manually.
