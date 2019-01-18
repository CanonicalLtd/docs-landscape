# Landscape Release 19.01
These are the release notes for Landscape 19.01.

## Highlights

 * Improvements to security updates detection.
 * [#1770416](https://bugs.launchpad.net/landscape/+bug/1770416) Create script to unblock stuck repository activities.
 * [#1782651](https://bugs.launchpad.net/landscape/+bug/1782651) Disable TLS 1.0 in apache2 config (quickstart install)
 * [#1791119](https://bugs.launchpad.net/landscape/+bug/1791119) ComputerOfflineAlert flapping when exchanges are not happening
 * [#1797503](https://bugs.launchpad.net/landscape/+bug/1797503) Option to randomise delivery of manual upgrade requests
 * [#673002](https://bugs.launchpad.net/landscape/+bug/673002) Activity scheduling widget should reject dates in the past
 * [#1616626](https://bugs.launchpad.net/landscape/+bug/1616626) Landscape appears to hang after hitting 'remove computer'
 * [#1764456](https://bugs.launchpad.net/landscape/+bug/1764456) Segmentation fault in landscape-env.sh
 * [#1774210](https://bugs.launchpad.net/landscape/+bug/1774210) Can't run a script on more than 100 clients using the API 
 * [#1790656](https://bugs.launchpad.net/landscape/+bug/1790656) last_exchange_time should be visible in a couple key places 
 * [#1797087](https://bugs.launchpad.net/landscape/+bug/1797087) Landscape should display CVEs alongside USNs
 * [#1786078](https://bugs.launchpad.net/landscape/+bug/1786078) Landscape-server-quickstart fails if there is no eth0
 * [#1810793](https://bugs.launchpad.net/landscape/+bug/1810793) Landscape should include GPG material for Bionic.
 * There are no special upgrade instructions for Landscape 19.01, regardless of the installation method.

## Upgrade notes

Landscape 19.01 supports Ubuntu 18.04 LTS ("bionic"). It can only be upgraded from Landscape 18.03 also running on the same Ubuntu 18.04 LTS release.

To upgrade a Landscape 18.03 running on Ubuntu 16.04 LTS ("xenial") to Landscape 19.01, first you need to upgrade your Ubuntu release to 18.04 LTS ("bionic"), and then upgrade Landscape to 19.01.

## Quickstart upgrade
If you used the landscape-server-quickstart package to install Landscape 18.03 then you can use this method to upgrade it.

If you are a [Landscape](https://landscape.canonical.com) customer, you can select new version of Landscape in your hosted account at [https://landscape.canonical.com](https://landscape.canonical.com) and then run:
```
    sudo apt-get update
    sudo apt-get dist-upgrade
```
Alternatively, just add the Landscape 19.01 PPA and run the same commands as above:
```
    sudo add-apt-repository -u ppa:landscape/19.01
    sudo apt-get dist-upgrade
```
When prompted, reply with `N` to any dpkg questions about configuration files so the existing files stay untouched. The quickstart package will make any needed modifications to your configuration files automatically. 

## Non-quickstart upgrade

Follow these steps to perform a non-quickstart upgrade, that is, you did not use the landscape-server-quickstart package when installing Landscape 19.01:

 * stop all landscape services on all machines that make up your non-quickstart deployment, except the database service: `sudo lsctl stop`
 * double check that `UPGRADE_SCHEMA` is set to what you want in `/etc/default/landscape-server`
 * disable all the landscape-server cron jobs from `/etc/cron.d/landscape-server` in all app servers
 * Update the Landscape apache vhost as follows:
  * Add the following SSL directives to the HTTPS vhost:
```
# Disable insecure TLSv1
  SSLProtocol all -SSLv3 -SSLv2 -TLSv1
  SSLHonorCipherOrder On
  SSLCompression Off
  # Disable old/vulnerable ciphers. Note: one very long line
  SSLCipherSuite EECDH+AESGCM+AES128:EDH+AESGCM+AES128:EECDH+AES128:EDH+AES128:ECDH+AESGCM+AES128:aRSA+AESGCM+AES128:ECDH+AES128:DH+AES128:aRSA+AES128:EECDH+AESGCM:EDH+AESGCM:EECDH:EDH:ECDH+AESGCM:aRSA+AESGCM:ECDH:DH:aRSA:HIGH:!MEDIUM:!aNULL:!NULL:!LOW:!3DES:!DSS:!EXP:!PSK:!SRP:!CAMELLIA:!DHE-RSA-AES128-SHA:!DHE-RSA-AES256-SHA:!aECDH
```
 * Restart apache using `sudo service apache2 restart`
 * add the Landscape 19.01 PPA: `sudo add-apt-repository -u ppa:landscape/19.01`
 * update and upgrade: `sudo apt-get update && apt-get dist-upgrade`
 * answer with `N` to any dpkg questions about Landscape configuration files
 * if you have `UPGRADE_SCHEMA` enabled in `/etc/default/landscape-server`, then the required schema upgrade will be performed as part of the package upgrade and all services will be running at the end. The upgrade is finished.
 * if `UPGRADE_SCHEMA` is disabled, then you will have failures when the services are restarted at the end of the upgrade. That's expected. You now have to perform the schema upgrade manually with this command: 
```
    sudo setup-landscape-server
```
  After all these steps are completed, the Landscape services can be started: 
```
    sudo lsctl restart
```
 * re-enable the landscape-server cron jobs in `/etc/cron.d/landscape-server` in all app servers

## Charm upgrade with juju 2.x

Starting with Landscape 15.10, Juju deployed Landscape can be upgraded
in place. If you have just one landscape-server unit, please follow
this procedure:

```
juju upgrade-charm landscape-server
juju config landscape-server source=ppa:landscape/19.01
juju run-action landscape-server/0 pause
juju run-action landscape-server/0 upgrade
juju run-action landscape-server/0 migrate-schema
juju run-action landscape-server/0 resume
```

For multiple landscape-server units, you should pause all of them,
upgrade one by one, run the migrate-schema command on only one, and
then resume all units.

Each action returns an identifier that should be used to check its
outcome with the fetch command before running the next action:

```
juju show-action-output --wait 0 <uuid>
```

For example:

```
$ juju run-action landscape-server/0 pause
Action queued with id: 72fd7975-3e0b-4b6d-84b9-dbd76d50f6af
$ juju show-action-output --wait 0 72fd7975-3e0b-4b6d-84b9-dbd76d50f6af
status: completed
timing:
  completed: 2019-01-11 04:27:57 +0000 UTC
  enqueued: 2019-01-11 04:27:46 +0000 UTC
  started: 2019-01-11 04:27:47 +0000 UTC
```

As an example of when it fails and what kind of output to expect, here
we are trying to upgrade a unit that hasn't been paused before, other
than command line structure, this has not changed in Juju 2.x:

```
$ juju run-action landscape-server/0 upgrade
Action queued with id: f3d2343c-33e4-4faf-8c4e-59f796124dd4
$ juju show-action-output --wait 0 f3d2343c-33e4-4faf-8c4e-59f796124dd4
message: This action can only be called on a unit in paused state.
status: failed
timing:
  completed: 2016-06-23 19:26:40 +0000 UTC
  enqueued: 2016-06-23 19:26:36 +0000 UTC
  started: 2016-06-23 19:26:38 +0000 UTC
```

## Charm upgrade with Juju 1.x

Starting with Landscape 15.10, Juju deployed Landscape can be upgraded
in place. If you have just one landscape-server unit, please follow
this procedure:

```
juju upgrade-charm landscape-server
juju set landscape-server source=ppa:landscape/19.01
juju action do landscape-server/0 pause
juju action do landscape-server/0 upgrade
juju action do landscape-server/0 migrate-schema
juju action do landscape-server/0 resume
```

For multiple landscape-server units, you should pause all of them,
upgrade one by one, run the migrate-schema command on only one, and
then resume all units.

Each action returns an identifier that should be used to check its
outcome with the fetch command before running the next action:

```
juju action fetch <uuid>
```

For example:

```
$ juju action do landscape-server/0 pause
Action queued with id: 72fd7975-3e0b-4b6d-84b9-dbd76d50f6af
$ juju action fetch 72fd7975-3e0b-4b6d-84b9-dbd76d50f6af
status: completed
timing:
  completed: 2016-06-23 19:24:39 +0000 UTC
  enqueued: 2016-06-23 19:24:32 +0000 UTC
  started: 2016-06-23 19:24:33 +0000 UTC
```

As an example of when it fails and what kind of output to expect, here
we are trying to upgrade a unit that hasn't been paused before, other
than command line structure:

```
$ juju action do landscape-server/0 upgrade
Action queued with id: f3d2343c-33e4-4faf-8c4e-59f796124dd4
$ juju action fetch f3d2343c-33e4-4faf-8c4e-59f796124dd4
message: This action can only be called on a unit in paused state.
status: failed
timing:
  completed: 2016-06-23 19:26:40 +0000 UTC
  enqueued: 2016-06-23 19:26:36 +0000 UTC
  started: 2016-06-23 19:26:38 +0000 UTC
```


## Known issues

This section describes some relevant known issues that might affect your usage of Landscape 19.01.

 * The `landscape-package-search` service ignores the `RUN_*` variable settings in `/etc/default/landscape-server` and will always try to start. To configure it not to start, run this command: `sudo systemctl disable landscape-package-search`. If it was already running, you will also have to stop it: `sudo service landscape-package-search stop`. This is only noticeable using multiple application servers [Bug #1675569](https://bugs.launchpad.net/landscape/+bug/1675569).

 * To upgrade to 19.01 from Ubuntu 16.04, you must first `do-release-upgrade` to Ubuntu 18.04.

 * When the landscape-server package is installed or upgraded, its postinst step runs a `chown landscape:landscape -R /var/lib/landscape` command. If you have the repository management files mounted via NFS in the default location `/var/lib/landscape/landscape-repository` and with the NFS `root_squash` option set, then this command will fail. There are two workarounds:
   * temporarily enable the `no_root_squash` option on the NFS server, which will allow the command to complete
   * mount the repository elsewhere, outside of the `/var/lib/landscape` tree. For example, to mount it under `/landscape-repository`, follow these steps:
```
    sudo mkdir -m 0755 /landscape-repository 
    sudo chown landscape:landscape /landscape-repository 
    sudo vi /etc/landscape/service.conf <-- change repository-path to /landscape-repository 
    sudo vi /etc/apache2/sites-enabled/<yourvhost> <-- change "Alias /repository /var/lib/landscape/landscape-repository" to "Alias /repository /landscape-repository" 
    sudo lsctl stop 
    sudo service apache2 stop 
    sudo umount /var/lib/landscape/landscape-repository # may have to kill gpg-agent processes to be allowed to umount 
    sudo mount <nfserver>:<export> -t nfs -o rw /landscape-repository 
    # check that /landscape-repository and files/directories under it are still owned by landscape:landscape 
    sudo service apache2 start 
    sudo lsctl start 
    # update /etc/fstab regarding the new mount point, to avoid surprises after a reboot 
```

 * Also due to the `chown` command run during postinst explained above, the upgrade can take a long time if the repository files are mounted somewhere `/var/lib/landscape`, depending on the size of the repository. On an experiment with two machines on the same gigabit switch and a 150Gb repository mounted via NFS, a test upgrade spent about 30min just in that `chown` command. While that happens, the service is down. This is being tracked as [bug #1725282](https://bugs.launchpad.net/landscape/+bug/1725282) and until a fix is explicitly mentioned in the release notes, we suggest the same workaround as for the previous case: mount the repository outside of the `/var/lib/landscape/` tree.
