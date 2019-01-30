Title: Repositories

# Repositories

This requires using the Landscape [API](./api.md)

Linux distributions like Ubuntu use repositories to hold packages you can
install on managed computers. While Ubuntu has several repositories that anyone
can access, you can also maintain your own repositories on your network. This
can be useful when you want to maintain packages with different versions from
those in the community repositories, or if you've packages in-house software
for installation.

Some terminology:

|Term               | Meaning	                             | Example
|-------------------|----------------------------------------|--------------------------------------
|Distribution       | A flavor of Linux                      | Ubuntu
|Series             | A distribution release                 | Natty, Oneiric, Precise
|Pocket             | Where packages are stored              | updates, security, release
|Suite              | A combination of a series and a pocket | precise-updates
|Components         | an apt sources.list line               | main, extra, universe
|Architecture       | A computer's CPU/hardware	             | i386, amd64, source (for source packages)

!!! Note:
    The special pocket release never gets mentioned in a suite.

In a `sources.list` line, you would see:

```no-format
deb http://archive.ubuntu.com/DISTRIBUTION/ SERIES-POCKET COMPONENT [COMPONENT ...]
```

## Repository management - getting started

You can mirror an upstream repository with the following script. For the sake
of brevity, instead of mirroring the whole of, say, Precise, you can just
mirror the restricted component, which is smaller. You must have set up the API
client already, then run the following script:

```bash
# import a secret GPG key. This will be used by Landscape On-Premises to sign the repository.
# export a GPG secret key using gpg --export-secret-keys -a KEYID > secret-key.pem
# Note: the secret key must NOT have a passphrase. To remove the passphrase from a key,
# use gpg --edit-key KEYID before exporting it. See gpg(1) for details.
$ landscape-api import-gpg-key secret-key secret-key.pem
{u'fingerprint': u'5c49:8483:3dbf:5aaf:382e:000e:bc0e:d36a:1703:785b',
 u'has_secret': True,
 u'id': 7,
 u'name': u'secret-key'}

# create a distribution
$ landscape-api create-distribution ubuntu
{u'creation_time': u'2011-10-14T19:44:13Z', u'name': u'ubuntu', u'series': []}

# now create a series and some pockets, which are what hold the actual packages.
# This will create a "precise" series, with pockets for "release" and "main"
# of the "restricted" component, and for the i386 arch. It won't mirror any packages
# yet.
$ landscape-api create-series --pockets release,updates --components restricted --architectures i386 \
  --gpg-key secret-key --mirror-uri http://archive.ubuntu.com/ubuntu/ --mirror-series lucid lucid ubuntu
{u'creation_time': u'2011-10-14T19:44:57Z',
 u'name': u'precise',
 u'pockets': [{u'apt_source_line': u'deb http://biriba.canonical.com/repository/standalone/ubuntu precise restricted',
(...)

# now, let's sync the pockets. This will start the actual mirroring process:
$ landscape-api sync-mirror-pocket release precise ubuntu
{u'children': [],
 u'computer_id': None,
 u'creation_time': u'2011-10-14T19:45:51Z',
 u'creator': {u'email': u'stan@example.com',
              u'id': 1,
              u'name': u'Stan Peters'},
 u'id': 101,
 u'parent_id': None,
 u'pocket_id': 5,
 u'pocket_name': u'release',
 u'progress': 0,
 u'summary': u"Sync pocket 'release' of series 'precise' in distribution 'ubuntu'",
 u'type': u'SyncPocketRequest'}


# the result of the above command is an activity, and it has an id.
# We can query its progress by using get-activities with the activity id, and inspect the "progress" result,
# which is a percentage:
$ landscape-api get-activities --query id:101
(...)
  u'id': 101,
  u'parent_id': None,
  u'pocket_id': 5,
  u'pocket_name': u'release',
  u'progress': 35,
(...)

# it's almost done. We can only issue another sync-mirror-pocket call once the above is finished.
# Once it's finished, we can issue another call, this time to sync the updates pocket:
$ landscape-api sync-mirror-pocket updates precise ubuntu
(...)
 u'id': 102,
(...)

# while the sync happens, we can create a repository profile which we will later apply to computers:
$ landscape-api create-repository-profile --description "This profile is for Landscape On-Premises servers." lds-profile
{u'all_computers': False,
 u'description': u'This profile is for Landscape On-Premises servers.',
 u'id': 5,
 u'name': u'lds-profile',
 u'pockets': [],
 u'tags': []}

# now we associate computers with the tag "lds" to this repository profile:
$ landscape-api associate-repository-profile --tags lds lds-profile
{u'all_computers': False,
 u'description': u'This-profile-is-for-Landscape-On-Premises-servers.',
 u'id': 5,
 u'name': u'lds-profile',
 u'pockets': [],
 u'tags': [u'lds']}

# finally, we say which pockets are part of this repository profile:
$ landscape-api add-pockets-to-repository-profile lds-profile release,updates precise ubuntu
True
```

At the end of the script, computers with the "lds" tag will get an entry in
`/etc/apt/sources.list.d/` pointing to the newly created release and updates
repository for Precise restricted component.

The repositories are also visible via a web browser at
http://<lds-server>/repository/standalone/ubuntu/pool/ and
http://<lds-server>/repository/standalone/ubuntu/dists/.

### Repository management mirroring tips

Here are some simple tips on how to create standard repositories using
Landscape. They all use the `create-pocket` API call, so to use them, you must
have created a distribution (for example ubuntu, using a command like
`create-distribution ubuntu`) and a series (for instance precise, with a command
like `create-series precise ubuntu`).

For complete create-pocket syntax, run the command `landscape-api create-pocket -h`.

Suppose you want to mirror an upstream repository. Basic usage looks like this:

```bash
landscape-api create-pocket [--mirror-suite MIRROR-SUITE] \
[--mirror-uri <MIRROR-URI>] <POCKETNAME> <SERIES> <DISTRIBUTION> \
<COMPONENT> <ARCHITECTURE> <MODE> <GPGKEY>
```

In this command, landscape-api and create-pocket are constants; the rest are
variables. The values in [brackets] are optional.

For MIRROR-SUITE, use the release designation; you can add an optional suffix
to the name if you want only certain packages, such as all updates or only
security updates.

- **POCKETNAME** should match the suffix of the **MIRROR-SUITE**, or if you're mirroring
  an entire release, use release.
- **SERIES** is a release nickname, such as oneiric or precise.
- The **DISTRIBUTION** is ubuntu.
- The **COMPONENT** name may be one or more components (main, restricted, universe,
  multiverse) separated by commas. The **ARCHITECTURE** may be i386, amd64, or
  both, separated by commas.
- **MODE** may be pull, mirror, or upload.
- The **GPGKEY** is the private passphrase-less GPG key that you created in section 1 above.

Here's an example with some values filled in:

```bash
landscape-api create-pocket --mirror-suite precise \
--mirror-uri http://archive.ubuntu.com/ubuntu/ \
release precise ubuntu main,universe \
i386 mirror mirror-sign-key
```

In this command, --mirror-suite indicates you want to create a pocket to mirror
a series (precise) as it was released, without any updates. To create a pocket
to mirror updates for a series, add the -updates suffix on the series name:

```bash
landscape-api create-pocket --mirror-suite precise-updates \
--mirror-uri http://archive.ubuntu.com/ubuntu/ \
updates precise ubuntu main,universe \
i386 mirror mirror-sign-key
```

To create a pocket to mirror only security updates for a series, use the
-security suffix:

```bash
landscape-api create-pocket --mirror-suite precise-security \
--mirror-uri http://archive.ubuntu.com/ubuntu/ \
security precise ubuntu main,restricted,universe,multiverse \
i386,amd64 mirror mirror-sign-key
```

The specific suffix is not significant. You could theoretically choose a
different convention for pocket names, but we suggest you stick to this usage.

Once you've created the pocket or pockets you want to use, call
sync-mirror-pocket to start the mirroring process:

```bash
landscape-api sync-mirror-pocket release precise ubuntu
```

## Upload pockets

Landscape also lets you create and manage repositories that hold packages
uploaded by authorized users. You could, for example, create a staging area to
which certain users could upload packages. Assuming both the series and the
distribution are created already, you would use a command like:

```bash
landscape-api create-pocket staging precise ubuntu main i386 upload upload-sign-key
```

where:

* **staging** is the name of the upload pocket to be created
* **precise** is the series
* **ubuntu** is the distribution
* **upload** is the pocket type
* and the rest of the parameters are the same as those for create-pocket.
