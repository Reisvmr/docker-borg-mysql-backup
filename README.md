# borg-mysql-backup

This image is mainly intended for backing up mysql dumps to local and/or remote [borg](https://github.com/borgbackup/borg) repos.
Other files may be included in the backup, and database dumps can be excluded altogether.

The container features `backup`, `restore` and `list` scripts that can either be ran
directly with docker, or by cron, latter being the preferred method.

For cron and/or remote borg usage, you also need to mount container configuration
at `/config`, containing ssh key (named `id_rsa`) and/or crontab file (named `crontab`).

Local borg repo will be located under the dir you mount container `/backup` at.
**Remote borg repo needs to be initialised manually beforehand.**

In case some containers need to be stopped for the backup (eg to ensure there won't be any
mismatch between data and database), you can specify those container names to
the `backup` script (see below). Note that this requires mounting docker socket with `-v /var/run/docker.sock:/var/run/docker.sock`,
but keep in mind it has security implications - borg-mysql-backup will have essentially
root permissions on the host.

To synchronize container tz with that of host's, then also add following mount:
`-v /etc/localtime:/etc/localtime:ro`. You'll likely want to do this for cron times
to match your local time.

It's possible to get notified of _any_ errors that occur during backups.
Currently supported notification methods are

- sending mail via SMTP
- sending [pushover](https://pushover.net/) notifications

If you wish to provide your own msmtprc config file instead of defining `SMTP_*` env
vars, create it at the `/config` mount, named `msmtprc`.

Try to avoid using the `latest` version of this image, as you'd want to be tied to a 
certain version of borg - different borg versions can be non-compatible.

Every time any config is changed in `/config`, container needs to be restarted.


## Container Parameters

    MYSQL_HOST              the host/ip of your mysql database
    MYSQL_PORT              the port number of your mysql database
    MYSQL_USER              the username of your mysql database
    MYSQL_PASS              the password of your mysql database
    MYSQL_FAIL_FATAL        whether unsuccessful db dump should abort backup,
                            defaults to 'true';
    MYSQL_EXTRA_OPTS        the extra options to pass to 'mysqldump' command; optional
      mysql env variables are only required if you intend to back up databases


    HOST_NAME               hostname to include in the borg archive name
    REMOTE                  remote connection - user & host; eg for rsync.net
                            it'd be something like '12345@ch-s010.rsync.net'
                            optional - can be omitted when only backing up to local
                            borg repo, or if providing value via script
    REMOTE_REPO             path to repo on remote host, eg '/backup/repo'
                            optional - can be omitted when only backing up to local
                            borg repo, or if providing value via script
    BORG_LOCAL_REPO         path to local borg repo; optional - can be omitted
                            when only backing up to remote borg repo, or if
                            providing value via script
    BORG_EXTRA_OPTS         additional borg params (for both local & remote borg commands); optional
    BORG_LOCAL_EXTRA_OPTS   additional borg params for local borg command; optional
    BORG_REMOTE_EXTRA_OPTS  additional borg params for remote borg command; optional
    BORG_REMOTE_PATH        remote borg executable path; eg with rsync.net
                            you'll likely want to use value 'borg1'; optional
    BORG_PASSPHRASE         borg repo password
    BORG_PRUNE_OPTS         options for borg prune (both local and remote); not required when
                            restoring or if it's defined by backup script -P param
                            (which overrides this container env var)

    ERR_NOTIF               space separated error notification methods; supported values
                            are {mail,pushover}; optional
    NOTIF_SUBJECT           notifications' subject/title; defaults to '{p}: backup error on {h}'

      following params {MAIL,SMTP}_* are only used if ERR_NOTIF value contains 'mail';
      also note all SMTP_* env vars besides SMTP_ACCOUNT are ignored if you've
      provided smtp config at /config/msmtprc
    MAIL_TO                 address to send notifications to
    MAIL_FROM               name of the notification sender; defaults to '{h} backup reporter'
    SMTP_HOST               smtp server host; only required if MSMTPRC file not provided
    SMTP_USER               login user to the smtp account; only required if MSMTPRC file not provided
    SMTP_PASS               login password to the smtp account; only required if MSMTPRC file not provided
    SMTP_PORT               smtp server port; defaults to 587
    SMTP_AUTH               defaults to 'on'
    SMTP_TLS                defaults to 'on'
    SMTP_STARTTLS           defaults to 'on'
    SMTP_ACCOUNT            smtp account to use for sending mail, defaults to 'default';
                            makes sense only if you've provided your own MSMTPRC
                            config at /config/msmtprc that defines multiple accounts

      following params are only used/required if ERR_NOTIF value contains 'pushover':
    PUSHOVER_USER_KEY       your pushover account key
    PUSHOVER_APP_TOKEN      token of a registered app to send notifications from
    PUSHOVER_PRIORITY       defaults to 1
    PUSHOVER_RETRY          only in use if priority=2; defaults to 60
    PUSHOVER_EXPIRE         only in use if priority=2; defaults to 3600

## Script usage

Container incorporates `backup`, `restore` and `list` scripts.

### backup.sh

`backup` script is mostly intended to be ran by the cron, but can also be executed
directly via docker for one off backup.

    usage: backup [-h] [-d MYSQL_DBS] [-n NODES_TO_BACKUP] [-c CONTAINERS] [-rl]
                  [-P BORG_PRUNE_OPTS] [-B|-Z BORG_EXTRA_OPTS] [-N BORG_LOCAL_REPO]
                  [-e ERR_NOTIF] [-A SMTP_ACCOUNT] [-D MYSQL_FAIL_FATAL] -p PREFIX
    
    Create new archive
    
    arguments:
      -h                      show help and exit
      -d MYSQL_DBS            space separated database names to back up; use __all__ to back up
                              all dbs on the server
      -n NODES_TO_BACKUP      space separated files/directories to back up (in addition to db dumps);
                              path may not contain spaces, as space is the separator
      -c CONTAINERS           space separated container names to stop for the backup process;
                              requires mounting the docker socket (-v /var/run/docker.sock:/var/run/docker.sock);
                              note containers will be stopped in given order; after backup
                              completion, containers are started in reverse order;
      -r                      only back to remote borg repo (remote-only)
      -l                      only back to local borg repo (local-only)
      -P BORG_PRUNE_OPTS      overrides container env variable BORG_PRUNE_OPTS; only required when
                              container var is not defined or needs to be overridden;
      -B BORG_EXTRA_OPTS      additional borg params; note it doesn't overwrite
                              the BORG_EXTRA_OPTS env var, but extends it;
      -Z BORG_EXTRA_OPTS      additional borg params; note it _overrides_
                              the BORG_EXTRA_OPTS env var;
      -N BORG_LOCAL_REPO      overrides container env variable of same name;
      -e ERR_NOTIF            space separated error notification methods; overrides
                              env var of same name;
      -A SMTP_ACCOUNT         msmtp account to use; defaults to 'default'; overrides
                              env var of same name;
      -D MYSQL_FAIL_FATAL     whether unsuccessful db dump should abort backup; overrides
                              env var of same name; true|false
      -p PREFIX               borg archive name prefix. note that the full archive name already
                              contains HOST_NAME and timestamp, so omit those.

#### Usage examples

##### Back up App1 & App2 databases and app1's data directory /app1-data daily at 05:15 to both local and remote borg repos

    docker run -d \
        -e MYSQL_HOST=mysql.host \
        -e MYSQL_PORT=27017 \
        -e MYSQL_USER=admin \
        -e MYSQL_PASS=password \
        -e HOST_NAME=hostname-to-use-in-archive-prefix \
        -e REMOTE=remoteuser@server.com \
        -e REMOTE_REPO=repo/location \
        -e BORG_EXTRA_OPTS='--compression zlib,5 --lock-wait 60' \
        -e BORG_PASSPHRASE=borgrepopassword \
        -e BORG_PRUNE_OPTS='--keep-daily=7 --keep-weekly=4' \
        -v /etc/localtime:/etc/localtime:ro \
        -v /backup:/backup \
        -v /borg-mysql-backup/config:/config:ro \
        -v /app1-data-on-host:/app1-data:ro \
           layr/borg-mysql-backup

`/config/crontab` contents:

    15 05 * * *   /backup.sh -d "App1 App2" -n /app1-data -p app1-app2

##### Back up all databases daily at 04:10 and 16:10 to local&remote borg repos, stopping containers myapp1 & myapp2 for the process

    docker run -d \
        -e MYSQL_HOST=mysql.host \
        -e MYSQL_PORT=27017 \
        -e MYSQL_USER=admin \
        -e MYSQL_PASS=password \
        -e HOST_NAME=hostname-to-use-in-archive-prefix \
        -e REMOTE=remoteuser@server.com \
        -e REMOTE_REPO=repo/location \
        -e BORG_PASSPHRASE=borgrepopassword \
        -e BORG_PRUNE_OPTS='--keep-daily=7 --keep-weekly=4' \
        -v /var/run/docker.sock:/var/run/docker.sock \
        -v /etc/localtime:/etc/localtime:ro \
        -v /backup:/backup \
        -v /borg-mysql-backup/config:/config:ro \
           layr/borg-mysql-backup

`/config/crontab` contents:

    10 04,16 * * *   /backup.sh -d __all__ -p myapp-prefix -c "myapp1 myapp2"

##### Back up directores /app1 & /app2 every 6 hours to local borg repo (ie remote is excluded)

    docker run -d \
        -e HOST_NAME=hostname-to-use-in-archive-prefix \
        -e BORG_PASSPHRASE=borgrepopassword \
        -e BORG_PRUNE_OPTS='--keep-daily=7 --keep-weekly=4' \
        -v /backup:/backup \
        -v /borg-mysql-backup/config:/config:ro \
        -v /app1:/app1:ro \
        -v /app2:/app2:ro \
           layr/borg-mysql-backup

`/config/crontab` contents:

    0 */6 * * *   /backup.sh -l -n "/app1 /app2" -p my_app_prefix

Note we didn't need to define mysql- or remote borg repo related docker env vars.
Also there's no need to have ssh key in `/config`, as we're not connecting to a remote server.
Additionally, there was no need to mount `/etc/localtime`, as cron doesn't
define absolute time, but simply an interval.

##### Same as above, but report errors via mail

    docker run -d \
        -e HOST_NAME=hostname-to-use-in-archive-prefix \
        -e BORG_PASSPHRASE=borgrepopassword \
        -e BORG_PRUNE_OPTS='--keep-daily=7 --keep-weekly=4' \
        -v /backup:/backup \
        -v /borg-mysql-backup/config:/config:ro \
        -v /app1:/app1:ro \
        -v /app2:/app2:ro \
        -e ERR_NOTIF=mail \
        -e MAIL_TO=receiver@example.com \
        -e NOTIF_SUBJECT='{i} backup error' \
        -e SMTP_HOST='smtp.gmail.com' \
        -e SMTP_USER='your.google.username' \
        -e SMTP_PASS='your-google-app-password' \
           layr/borg-mysql-backup

Same as the example before, but we've also opted to get notified of any backup
errors via email.

##### Back up directory /emby once to remote borg repo (ie local is excluded)

    docker run -it --rm \
        -e HOST_NAME=hostname-to-use-in-archive-prefix \
        -e REMOTE=remoteuser@server.com \
        -e REMOTE_REPO=repo/location \
        -e BORG_PASSPHRASE=borgrepopassword \
        -e BORG_PRUNE_OPTS='--keep-daily=7 --keep-weekly=4' \
        -v /borg-mysql-backup/config:/config:ro \
        -v /emby/dir/on/host:/emby:ro \
           layr/borg-mysql-backup backup.sh -r -n /emby -p emby

Note there's no need to have a crontab file in `/config`, as we're executing this
command just once, after which container exits and is removed (ie we're not using
scheduled backups). Also note there's no `/backup` mount as we're operating only
against the remote borg repo.

### restore.sh

`restore` script should be executed directly with docker in interactive mode. All data
will be extracted into `/$RESTORE_DIR/restored-{archive_name}`.

Note none of the data is
copied/moved automatically - user is expected to carry this operation out on their own.
Only db will be restored from a dump, given the option is provided to the script.

    usage: restore [-h] [-d] [-c CONTAINERS] [-r] [-l]
                   [-N BORG_LOCAL_REPO] -a ARCHIVE_NAME
    
    Restore data from borg archive
    
    arguments:
      -h                      show help and exit
      -d                      automatically restore mysql database from dumped file; if this
                              option is given and archive contains no sql dumps, it's an error;
                              be careful, this is destructive operation!
      -c CONTAINERS           space separated container names to stop before the restore begins;
                              note they won't be started afterwards, as there might be need
                              to restore other data (only sql dumps are restored automatically);
                              requires mounting the docker socket (-v /var/run/docker.sock:/var/run/docker.sock)
      -r                      restore from remote borg repo
      -l                      restore from local borg repo
      -N BORG_LOCAL_REPO      overrides container env variable of same name; optional;
      -a ARCHIVE_NAME         name of the borg archive to restore/extract data from

#### Usage examples

##### Restore archive from remote borg repo & restore mysql with restored dumpfile

    docker run -it --rm \
        -e MYSQL_HOST=mysql.host \
        -e MYSQL_PORT=27017 \
        -e MYSQL_USER=admin \
        -e MYSQL_PASS=password \
        -e REMOTE=remoteuser@server.com \
        -e REMOTE_REPO=repo/location \
        -e BORG_PASSPHRASE=borgrepopassword \
        -v /tmp:/backup \
        -v /borg-mysql-backup/config:/config:ro \
           layr/borg-mysql-backup restore.sh -r -d -a my_prefix-HOSTNAME-2017-02-27-160159

##### Restore archive from local borg repo & stop container app1 beforehand

    docker run -it --rm \
        -e BORG_PASSPHRASE=borgrepopassword \
        -v /tmp:/backup \
           layr/borg-mysql-backup restore.sh -l -c app1 -a my_prefix-HOSTNAME-2017-02-27-160159

Note there's no need to mount `/config`, as we're not using cron nor connecting to remote borg.
Also we're not providing mysql env vars, as script isn't invoked with `-d` option, meaning
db won't be automatically restored with the included .sql dumpfile (if there was one).

##### Restore archive from non-default local borg repo

    docker run -it --rm \
        -v /tmp:/backup \
           layr/borg-mysql-backup restore.sh -l -N otherrepo -a my_prefix-HOSTNAME-2017-02-27-160159

Data will be restored from non-default local borg repo `otherrepo`. Also note missing
env variable `BORG_PASSPHRASE`, which will be required to be typed in manually.

Note the `BORG_EXTRA_OPTS`, `BORG_LOCAL_EXTRA_OPTS`, `BORG_REMOTE_EXTRA_OPTS` env
variables are still usable with `restore`.

### list.sh

`list` script is for listing archives in a borg repo.

    usage: list [-h] [-r] [-l] [-N BORG_LOCAL_REPO]
    
    List archives in a borg repository
    
    arguments:
      -h                      show help and exit
      -r                      list remote borg repo
      -l                      list local borg repo
      -N BORG_LOCAL_REPO      overrides container env variable of same name; optional;

#### Usage examples

##### List the local repository contents

    docker run -it --rm \
        -e BORG_PASSPHRASE=borgrepopassword \
        -v /tmp:/backup \
           layr/borg-mysql-backup list.sh -l

##### List the remote repository contents

    docker run -it --rm \
        -e BORG_EXTRA_OPTS="-P my-prefix" \
        -e REMOTE=remoteuser@server.com \
        -e REMOTE_REPO=repo/location \
        -e BORG_PASSPHRASE=borgrepopassword \
        -v /borg-mysql-backup/config:/config:ro \
           layr/borg-mysql-backup list.sh -r

Note the `BORG_EXTRA_OPTS`, `BORG_LOCAL_EXTRA_OPTS`, `BORG_REMOTE_EXTRA_OPTS` env
variables are still usable with `list`.

### notif-test.sh

Test your configured notifications. See the script for usage.

    docker run -it --rm \
        -e PUSHOVER_USER_KEY='key' \
        -e PUSHOVER_APP_TOKEN='token' \
        -e MAIL_TO='your@mail.com' \
        -e SMTP_HOST='smtp.gmail.com' \
        -e SMTP_USER='your.google.username' \
        -e SMTP_PASS='your-google-app-password' \
        -e ERR_NOTIF='mail pushover' \
        -e HOST_NAME='our-hostname' \
           layr/borg-mysql-backup notif-test.sh -p 'my-prefix' [-f]


## See also/recommended
- [restic](https://github.com/restic/restic) - note: no compression
- [duplicacy](https://github.com/gilbertchen/duplicacy) - alternatives to borg. lock-free!
- [docker-db-backup](https://github.com/tiredofit/docker-db-backup) - similar service; supports multiple dbs
- [this blog](https://ifnull.org/articles/borgbackup_rsyncnet/) for borg setup
- [borgmatic](https://github.com/witten/borgmatic) - declarative borg config
- [this dockerised borgmatic](https://hub.docker.com/r/b3vis/borgmatic/) - provides same as this service, and more
- main offsite hostings: [rsync.net](https://www.rsync.net) & [BorgBase](https://www.borgbase.com/)
- for backups from k8s:
  - [velero](https://github.com/vmware-tanzu/velero)
  - [k8up](https://github.com/vshn/k8up)
  - [stash](https://github.com/stashed/stash)

