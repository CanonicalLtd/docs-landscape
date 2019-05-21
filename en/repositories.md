Title: Repositories

# Repositories

Repository management requires the use of the Landscape [API.](./api.md) Set it up and have it ready for the next steps. Linux distributions like Ubuntu use repositories to hold packages you can install on managed computers. While Ubuntu has several repositories that anyone can access, you can also maintain your own repositories on your network. This can be useful when you want to maintain packages with different versions from those in the community repositories, or if you've packages in-house software for installation. Once you add your machines a Landscape repository profile, it will take over the sources files. From now on, you will have to manage all the apt sources you need using Landscape. 

## Terminology

|Term               | Meaning	                             | Example
|-------------------|----------------------------------------|---------------------------------------------------
|Distribution       | A flavor of Linux                      | Ubuntu
|Series             | A distribution release nickname        | Trusty, Xenial, Bionic
|Pocket             | Where packages are stored              | release(*), updates, security, proposed, backports
|Suite              | A combination of a series and a pocket | bionic-updates
|Components         | An apt sources.list line               | main, restricted, universe, multiverse
|Architecture       | A computer's CPU/Hardware	             | amd64, i386, armhf, source (for source packages)

In the following instructions we will use:

* **distribution**: ubuntu
* **series**: bionic
* **pockets**: release, updates, security
* **components**: main, restricted, universe, multiverse
* **tag**: example-tag
* **repository profile**: example-profile
* **mirror-key**: the name of the gpg key used by Landscape to sign your repository

Make sure you have set up the [API](./api.md) client already, then follow the steps below. The following instructions were performed on a Ubuntu 18.04 LTS with Landscape server 19.01 installed.


*Note: The special pocket release never gets mentioned in a suite.

In a `sources.list` line, you would see:

```no-format
deb http://archive.ubuntu.com/DISTRIBUTION/ SERIES-POCKET COMPONENT [COMPONENT ...]
```

## Disk space requirements
Packages will be downloaded under the `/var/lib/landscape/landscape-repository/standalone/` directory. As of February 2019, here is the breakdown of the total size of the release,updates and security pockets for the main,restricted,universe and multiverse components of the amd64 and i386 architectures.

|           |      | 18.04|       | |        |16.04|        | |       |14.04  |
|-----------|------|------|-------|-|--------|-----|--------|-|-------|-------|-------
|           | amd64|i386  |total  |-|   amd64|i386 | total  |-|amd64  |i386   |total
|Release    |73GB  |72GB  |145GB  |-|    73GB|70GB | 143GB  |-|59GB   |59GB   |118GB
|Updates    |22GB  |18GB  |40GB   |-|    42GB|34GB | 76GB   |-|46GB   |43GB   |89GB
|Security   |1.4GB |1.4GB |2.8GB  |-|   1.2GB|1.2GB|2.4GB   |-|1.4GB  |1.3GB  |2.7GB
|All Pockets|96.4GB|91.4GB|187.8GB|-|116.2GB|105.2GB|221.4GB|-|106.4GB|103.3GB|209.7GB

## Create the gpg key

Create a secret key GPG key and import it into Landscape This will be used by Landscape On-Premises to sign your repository. 
For this guide, we'll create a key with the 'real name' of `Mirror Key`.

First step is to install and run rngd to speed-up the creation of the gpg key.
```
sudo apt-get install rng-tools && sudo rngd -r /dev/urandom
```
Create the key that will be used to sign your repository
```bash
gpg --gen-key
```

You will be prompted twice for:
```
Please enter the passphrase to
protect your new key
```

Do not set any password. Choose `<OK>` to continue. 

Next, at `"Please confirm that you do not want to have any protection on your key"` choose:

```
<Yes, protection is not needed>
```

Note: the secret key must NOT have a passphrase. To remove the passphrase from a key, before exporting it, use:
```
gpg --edit-key A1234B5678C9101112D12141516E17181920FGH0
```

See the [gpg](http://manpages.ubuntu.com/cgi-bin/search.py?q=gpg) man page for more details.


List the keys:
```
gpg -K
sec   rsa3072 2019-02-05 [SC] [expires: 2021-02-04]
      A1234B5678C9101112D12141516E17181920FGH0
uid           [ultimate] Mirror Key
ssb   rsa3072 2019-02-05 [E] [expires: 2021-02-04]
```

Copy the new key numeric ID and export the key to a file:

```
gpg -a --export-secret-keys A1234B5678C9101112D12141516E17181920FGH0 > mirror-key.asc
```

Import the key file to Landscape:

```
landscape-api import-gpg-key mirror-key mirror-key.asc
{u'fingerprint': u'e144:5a89:bc8d:4fbc:49bc:5947:5eb9:93be:549a:8d7a',
 u'has_secret': True,
 u'id': 1,
 u'key_id': u'1BE771BB147D6E6D',
 u'name': u'mirror-key'}
```

## Create the distribution, series and pockets

Create the distribution first:
```
landscape-api create-distribution ubuntu
```

Create the series and the pockets which are what hold the actual packages. This will create a `bionic` series, with pockets for `release`, `security` and `updates`.
For components, we select `main`, `restricted`, `universe` and `multiverse`. It won't download any packages yet, just create the logical infrastrcuture for them.

```bash
landscape-api create-series \
  --pockets release,updates,security \
  --components main,restricted,universe,multiverse \
  --architectures amd64,i386 \
  --gpg-key mirror-key \
  --mirror-uri http://archive.ubuntu.com/ubuntu/ \
  --mirror-series bionic bionic ubuntu
```

## Sync pockets

We can sync only one pocket at a time. Once one pocket sync is done, we can start the next one. This command will start the actual mirroring process for the `release` pocket:


```bash
landscape-api sync-mirror-pocket release bionic ubuntu
{u'children': [],
 u'computer_id': None,
 u'creation_time': u'2018-10-14T19:45:51Z',
 u'creator': {u'email': u'stan@example.com',
              u'id': 1,
              u'name': u'Stan Peters'},
 u'id': 101,
 u'parent_id': None,
(...)
```
This will create an activity called `"Sync pocket 'release' of series 'bionic' in distribution 'ubuntu'"` that will be visible at `https://your-server.com/account/standalone/activities`.

Depending on your connection speed to the archive, the above may take a few hours. The output of the sync activity you just initiated will show an `id`. To monitor its progress, you can run a query on the activity `id` plus one. If the `id` in the output was `101`, we'll query for `102` and inspect the `progress` percent result, which in this example is 75.

```bash
landscape-api get-activities --query id:102
(...)
  u'id': 102,
  u'parent_id': None,
  u'pocket_id': 5,
  u'pocket_name': u'release',
  u'progress': 75,
(...)
```

It's almost done. Once it's finished the `'progress'` will be 100 and the activity `Status` in the WebUI will show `Succeeded`. Now, we can issue another call, this time to sync the `updates` pocket. As soon as this one is completed, we can finally sync the `security` pocket.
```
landscape-api sync-mirror-pocket updates bionic ubuntu
landscape-api sync-mirror-pocket security bionic ubuntu
```
 
 
The repositories are also visible via a web browser at:

`http://your-server.com/repository/standalone/ubuntu/pool/` 

and

`http://your-server.com/repository/standalone/ubuntu/dists/`
 
 
## Create a repository profile
We will create a repository profile named `example-profile` that later will be associated with a tag named `example-tag`. This profile will be applied to all computers that have that tag.
```
landscape-api create-repository-profile --description "This profile is for Landscape On-Premises servers." example-profile
{u'all_computers': False,
 u'description': u'This profile is for Landscape On-Premises servers.',
 u'id': 5,
 u'name': u'example-profile',
 u'pockets': [],
 u'tags': []}
```

## Associate computers with repository profile
This will associate all the computers with the tag `example-tag` to the `example-profile` repository profile:
```
landscape-api associate-repository-profile --tags example-tag example-profile
{u'all_computers': False,
 u'description': u'This-profile-is-for-Landscape-On-Premises-servers.',
 u'id': 5,
 u'name': u'example-profile',
 u'pockets': [],
 u'tags': [u'example-tag']}
```

## Add pockets to the repository profile
```
landscape-api add-pockets-to-repository-profile example-profile release,updates,security bionic ubuntu
```

At the end of this activity, all computers that have the `example-tag` will get an entry in `/etc/apt/sources.list.d/` pointing to the newly created repository for Bionic. This will create the `/etc/apt/sources.list.d/landscape-example-profile.list` sources file containing the following apt lines:
```
deb http://your-server.com/repository/standalone/ubuntu xenial-security main restricted universe multiverse
deb http://your-server.com/repository/standalone/ubuntu xenial main restricted universe multiverse
deb http://your-server.com/repository/standalone/ubuntu xenial-updates main restricted universe multiverse
```

Landscape will take over the client's apt sources. The original `sources.list` file will be moved aside and only the ones enabled in Landscape will work.

```
cat /etc/apt/sources.list
# Landscape manages repositories for this computer
# Original content of sources.list can be found in sources.list.save
```
  
To revert the changes, disassociate the tag from the repository profile:

```
landscape-api disassociate-repository-profile --tags example-tag example-profile
```

## Edit pockets

If needed, pockets can be modified using  `edit-pocket`. For example, to change the archive mirror address:
```
landscape-api edit-pocket release --mirror-uri https://mirror.math.princeton.edu/pub/ubuntu/ --mirror-suite bionic bionic ubuntu
```

This will change the `mirror_uri` for the `release` pocket of the `bionic` series in the `ubuntu` distribution from:

`http://archive.ubuntu.com/ubuntu/`

to:

`https://mirror.math.princeton.edu/pub/ubuntu/` 
  
Using an `https` source could be useful in situation where a contetnt filtering interfeers with the pocket sync when using an `http` source. Some of the [Official Archive Mirrors for Ubuntu](https://launchpad.net/ubuntu/+archivemirrors) are available over `https` although they are not advertised in the mirror list. For more options have a loot at `landscape-api edit-pocket --help`.
 
## Upload pockets

Removing a package from a pocket is only supported in upload mode. Landscape lets you create and manage repositories that hold packages uploaded by authorized users. You could, for example, create a staging area to which certain users could upload packages. Here is a quick howto for creating and uploading packages to such a repository.
 
 
Create another gpg key with the 'real name' of `Upload Key`. Export the key, but this time the secret key part of the upload key is not required. This was only required for the mirror-key as it will be signing your repository files.
```
gpg -a --export $KEYID > upload-key.asc
landscape-api import-gpg-key upload-key upload-key.asc
landscape-api get-gpg-keys
```
If the above was successful, you will have two gpg keys in Landscape, one with `has_secret` 'True' and the other 'False'.
 
 
Assuming this pocket is for bionic and that the ubuntu distribution is created already, this will create the upload type pocket:
```
landscape-api create-pocket staging bionic ubuntu main amd64 upload upload-key
```

where:

 * **staging**: the name of our upload pocket
 * **ubuntu**: the distribution (`create-distribution ubuntu`)
 * **bionic**: the series (`create-series bionic ubuntu`)
 * **main**: the component
 * **amd64**: the architecture
 * **upload**: the pocket type
 * **upload-key**: a private passphrase-less GPG key


Such a repository will be accessible via this sources.list entry:
```
deb http://your-server.com/repository/standalone/ubuntu bionic-staging main
```
You can choose who is allowed to upload packages to this pocket. Since the option `--upload-allow-unsigned` was not used when creating the pocket, only uploads signed by any of the `uploader gpg keys` will be allowed. Unsigned uploads, or signed by a key not in that list, will be rejected. To add or remove a key from that list, use `add-uploader-gpg-keys-to-pocket` and `remove-uploader-gpg-keys-from-pocket` respectively.
 
 
To upload packages to this pocket we use the tool `dput` with this configuration section in `~/.dput.cf`:
```
[lds]
fqdn = your-server.com
method = https
incoming = /upload/standalone/%(lds)s
```
The `%(lds)s` bit will be replaced by whatever follows the `lds:` prefix in the dput command shown below. The package debian/changelog file must contain the `bionic-staging` target, like this example:
```
my-package (1.0-0ubuntu1) bionic-staging; urgency=low

  * Released 1.0

  -- Package Builder <builder@example.com>  Tue, 12 Feb 2019 14:57:05 -0300
(...)
```
Since Landscape doesn't build packages from source, you will have to build the package locally for the right architecture and then upload the binary changes file, like this:
```
dput lds:ubuntu/bionic/staging my-package_1.0-0ubuntu1_amd64.changes
```
Note the path components: `lds:distro/series/pocket`.

If there are errors, they will be logged in the server's `/var/log/landscape/package-upload-1.log` log file. An email will also be sent to the uploader detailing the error.

For example, let's say you forgot to add the public GPG key of a developer to this pocket. If he tries to upload a package, he will get an error like this back in an email:
```
Subject: Package import failed for 'storm_0.18.1-0landscape4_amd64.changes'

The following error(s) occurred in package import:

* Uploaded data is not signed, but the destination pocket requires it
* The signature on following file(s) could not be verified.  Please make sure to import the GPG key(s) used to sign the files or use a different key to sign them.
  - storm_0.18.1-0landscape4_amd64.changes (key id 784259B3F3DDC290)
```

To fix this, we authorize uploads signed by that GPG key. Exported to a file:
```
gpg -a --export 784259B3F3DDC290 > 784259B3F3DDC290.pem
```

Import it into Landscape:
```
landscape-api import-gpg-key key-f3ddc290 ./784259B3F3DDC290.pem
{u'fingerprint': u'6f96:333f:ae2c:0ce5:254a:8742:7842:59b3:f3dd:c290',
 u'has_secret': False,
 u'id': 7,
 u'name': u'key-f3ddc290'}
```
Authorize it:
```
landscape-api add-uploader-gpg-keys-to-pocket staging bionic ubuntu key-f3ddc290
(...)
 u'upload_allow_unsigned': False,
 u'upload_gpg_keys': [{u'fingerprint': u'6f96:333f:ae2c:0ce5:254a:8742:7842:59b3:f3dd:c290',
                       u'has_secret': False,
                       u'id': 7,
                       u'name': u'key-f3ddc290'}]}
```

If you don't want to use gpg keys, change the pocket to allow unsigned packages:
```
landscape-api edit-pocket staging bionic ubuntu --upload-allow-unsigned=true

```


## Pull pockets
Pull mode pockets are meant to "stage" packages coming from another pocket, for example if you want to exclude some packages from that pocket, you can use filters (blacklist/whitelist) and then pull packages in the pocket again. Note that since pull operations don't really fetch packages, but just builds repository indices, you can also remove and recreate pull pockets very quickly if you want to start from scratch with filters. When creating the pull type pocket, you have to specify if you want a whitelist or blacklist for filtering.
```
landscape-api create-pocket --pull-pocket release --filter-type=whitelist release-staging bionic
```
The blacklist only works only for pull type repositories, but those don't support syncing from upstream archives, only from local ones.


## Repository management mirroring tips

Here are some simple tips on how to create standard repositories using Landscape. They all use the `create-pocket` API call, so to use them, you must have created a distribution (for example ubuntu, using a command like `create-distribution ubuntu`) and a series (for instance bionic, with a command like `create-series bionic ubuntu`). For complete create-pocket syntax, run the command `landscape-api create-pocket -h`. Suppose you want to mirror an upstream repository. Basic usage looks like this:

```bash
landscape-api create-pocket [--mirror-suite MIRROR-SUITE] \
[--mirror-uri <MIRROR-URI>] <POCKETNAME> <SERIES> <DISTRIBUTION> \
<COMPONENT> <ARCHITECTURE> <MODE> <GPGKEY>
```

In this command, landscape-api and create-pocket are constants; the rest are variables. The values in [brackets] are optional.

For `MIRROR-SUITE`, use the release designation; you can add an optional suffix to the name if you want only certain packages, such as all updates or only security updates.

* **pocketname** should match the suffix of the **mirror-suite**, or if you're mirroring an entire release, use release.
* **series** is a release nickname, such as xenial or bionic.
* The **distribution** is ubuntu.
* The **component** name may be one or more components (main, restricted, universe, multiverse) separated by commas. 
* The **architecture** may be amd64, i386 or armhf separated by commas.
* Pocket **mode** may be pull, mirror, or upload.
* The **gpgkey** is the private passphrase-less GPG key that you created above.

Here's an example with some values filled in:

```bash
landscape-api create-pocket --mirror-suite bionic \
--mirror-uri http://archive.ubuntu.com/ubuntu/ \
release bionic ubuntu main,universe \
amd64 mirror mirror-sign-key
```

In this command, `--mirror-suite` indicates you want to create a pocket to mirror a series (`bionic`) as it was released, without any updates. To create a pocket to mirror `updates` for a series, add the `-updates` suffix on the series name:

```bash
landscape-api create-pocket --mirror-suite bionic-updates \
--mirror-uri http://archive.ubuntu.com/ubuntu/ \
updates bionic ubuntu main,universe \
amd64 mirror mirror-sign-key
```

To create a pocket to mirror only `security` updates for a series, use the `-security` suffix:

```bash
landscape-api create-pocket --mirror-suite bionic-security \
--mirror-uri http://archive.ubuntu.com/ubuntu/ \
security bionic ubuntu main,restricted,universe,multiverse \
amd64 mirror mirror-key
```

The specific suffix is not significant. You could theoretically choose a different convention for pocket names, but we suggest you stick to this usage. Once you've created the pocket or pockets you want to use, call `sync-mirror-pocket` to start the mirroring process:

```
landscape-api sync-mirror-pocket release bionic ubuntu
```


## Mirror smaller pocket for testing
For testing purposes, instead of mirroring the whole `bionic` series, you can just mirror the `restricted` component of the `updates` pocket:

```bash
landscape-api create-series --pockets updates --components restricted --architectures amd64 --gpg-key mirror-key --mirror-uri http://archive.ubuntu.com/ubuntu/ --mirror-series bionic bionic ubuntu
```

