Title: Backup and Restore
# Backup and Restore

## Overview

On-premises Landscape consists (at a high level) of several stateful components, all of which need to be roughly in sync to guarantee correct functioning of the system as a whole. Specifically:

* The Landscape server, at least 6 PostgreSQL databases, a cache of hash databases, and a variety of configuration files.
* The Landscape clients, comprising several SQLite databases for tracking package states.

Given the wide variety of clients (from physical hardware, to VMs, to containers, some of which may be permanent and others merely temporary), backup of clients (if required at all) is beyond the scope of this document.

Backup of the Landscape server must be performed in such a fashion as to permit restoration to its latest possible state. Hence, the only supported backup option is one that permits point-in-time recovery (PITR hereafter).

We strongly recommend that administrators of an on-premises Landscape instance first familiarize themselves with PostgreSQL's archived logging and PITR facilities. Some syntactic configuration changes have occurred across PostgreSQL versions, hence you should select the documentation for your particular PostgreSQL server version:

* [PostgreSQL 9.3](https://www.postgresql.org/docs/9.3/static/continuous-archiving.html) (Ubuntu 14.04 LTS)
* [PostgreSQL 9.5](https://www.postgresql.org/docs/9.5/static/continuous-archiving.html) (Ubuntu 16.04 LTS)
* [PostgreSQL 10](https://www.postgresql.org/docs/10/static/continuous-archiving.html) (Ubuntu 18.04 LTS)


## Backup & Retention Policy

Before configuring your PostgreSQL instance for continuous archiving and PITR, it is important to decide on a backup policy:

1. When should base backups be taken? (recommendation: daily or weekly depending on volume of WAL logs, and desired recovery speed)
1. How many base backups should be retained at any given time? (this will dictate the earliest point in time to which you can initially restore)
1. Where should WAL logs be archived to? (recommendation: a separate machine or some form of networked, but secure storage)
1. Where should base backups be stored? (recommendation: the same machine as the WAL archive so that all materials necessary for restoration are available in one place)

Although it is possible to backup and archive on the same machine as the PostgreSQL server, we recommend a separate machine is used for base backup and archived log storage to permit restoration in the case the server becomes inaccessible for whatever reason. We further recommend that other files required to restore the Landscape application server (configuration files which will be listed below) are also copied to this location to permit recovery of the entire service from one place.


## PostgreSQL Configuration

In the `postgresql.conf` configuration file, set `wal_level`, `archive_mode`, and `archive_command` according to the PostgreSQL documentation for your server's version. As recommended by the PostgreSQL documentation, **test** your `archive_command` operates correctly in all circumstances, including returning the correct exit codes.

Once you are confident the configuration is correct, restart the PostgreSQL service to activate archived logging. Monitor the archive destination to ensure logs begin to appear there (if your server is very low traffic, you may wish to use the `archive_timeout` setting to force archiving of partial logs after a timeout).

When WAL logs are being archived successfully, construct a script that executes `pg_basebackup` and stores the result in your base backup storage destination. Test that this operates correctly as the cluster owner (typically `postgres`), then add a cronjob to periodically execute this script (as the cluster owner).

Note that there is no need to take Landscape offline to perform these backups. In fact, `pg_basebackup` can only execute when the cluster is up anyway. There is no need to worry about inconsistency between Landscape's various databases either: a base backup represents the state of the cluster (across all databases within it) at the instant the backup starts.


## Server Configuration Files

The following files should also be copied from your Landscape server(s) to your backup destination to ensure restoration of the Landscape application server is also possible:

* `/etc/landscape/*` - Landscape configuration files, and the on-premises Landscape license
* `/etc/default/landscape-server` - Specifies which Landscape application services start on a given machine
* `/etc/apache2/sites-available/<server-name>` - the Landscape Apache vhost configuration file, usually named after the FQDN of the server
* `/etc/ssl/certs/<landscape-cert>` - the Landscape server X.509 certificate
* `/etc/ssl/private/<landscape-cert>` - the Landscape server X.509 key file
* `/etc/postgresql/<pg-version>/main/*` - various PostgreSQL configuration files

Optionally, you may also wish to backup the following files. They are not required for normal operation of Landscape server, but may provide additional information in the case of service outages:

* `/var/log/landscape-server/*` - the Landscape server log files

If any of these files change periodically (e.g. the SSL certificates), you may also wish to set up a cronjob to handle backing-up these files regularly too.


## Restoration

We recommend that after configuring their Landscape server(s) for archived logging and PITR, administrators of on-premises Landscape test their recovery procedures, partially to ensure that backups are valid and restorable, and partially to ensure familiarity with such procedures.

1. Provision a spare server (or servers) and install Landscape server as you have on your production machine(s)
1. Stop the Landscape application server, and the PostgreSQL cluster on the spare
1. Copy configuration files (see prior section) to the spare; you may wish to keep a script handy to perform this task in your backup location
1. If your spare isn't installed from scratch (e.g. if it is installed from an image), remove everything under `/var/lib/landscape/hash-id-databases`
1. Restore a recent PostgreSQL base-backup onto the spare; this usually involves (re)moving the existing PostgreSQL cluster's data directory (e.g. /var/lib/postgresql/9.5/main) and replacing it with the contents of the base-backup (or un-tarring the base-backup into it, if tar format was chosen); ensure file ownership and modes are preserved!
1. Construct an appropriate `recovery.conf` file in the new PostgreSQL cluster's data directory; a template for this can be found in `/usr/share/postgresql/<pg-version>/main/recovery.conf.sample`
1. Start the PostgreSQL cluster on the spare and watch the PostgreSQL logs to ensure recovery proceeds by retrieving and replaying WAL logs
1. Once recovery is complete, run `/opt/canonical/landscape/scripts/hash-id-databases.sh` to regenerate the hash databases cache
1. Finally, start the Landscape application server on the spare and test it to verify correct operation
