Title: Use cases

# Use cases

You can use Landscape to perform many common system administration tasks
easily and automatically. Here are a few examples how to's...

## Group machines together to perform a task across the group

You can use [tags] to manage a group of computers. To add a tag to a group of
computers:

1. Click on the 'Computers' tab.
1. Select the computers you want to tag.
1. Click 'Info'.
1. In the 'Tags' section enter the tag you want to use.
1. Click the 'Add' button.

## Upgrade all packages on a certain group of machines

Using tags, you can perform an upgrade across a group of machines. If, for
instance, you want to upgrade all your desktop computers, you might want to use
"desktop" as a tag.

Starting the upgrade:

1. Click on the 'Computers' tab.
1. Click the desired tag from the left column. This will select only the
computers associated with the tag selected.
1. Click 'Packages'.
1. Scroll to the bottom of the page and click 'Request upgrades'. This will
create a queued 'Activity' for upgrading the computers.

![Activities - Upgrade pending approval][img_pending_approval]

!!! Note:
    While the upgrade tasks are now in the queue, they will not be executed
    until you approve them. To approve the tasks, click select 'All' and click
    the 'Approve' button at the bottom of the page.

## Keep a set of machines automatically up to date

The best way is to use [upgrade profiles], which rely on access groups. If an
access group is already setup for the group of machines you want to keep
updated automatically, simply click on its name. If not, you must create an
access group for them:

1. Click on your account name, then 'Access groups'.
1. Specify a title for your new access group and click 'Save'.

You must then add computers to the access group:

1. Click on the 'Computers' tab.
1. Select all of the machines you want to keep updated by:
    * using a tag if one exists
    * using search to find the machines
    * selecting them individually
1. Click on the 'Info' tab.
1. In the 'Access group' section, select the access group you want to move the
machines to.
1. Click 'Update access group'.

![Update access group][img_update_access_group]

Now that you have added machines to an access group, you will need to create
an upgrade profile:

1. Click on your account name, then 'Profiles'.
1. Click the 'Upgrade Profiles' link, then 'Add upgrade profile'.
1. Complete the 'Create an upgrade profile' form, defining:
    * name
    * the upgrade settings you want to use
    * an access group
    * the schedule you want to use
1. Click 'Save'

## Keep Landscape from upgrading a certain package on one of my servers

1. Click on the 'Computers' tab, then 'Packages'.
1. Use the search box at the top of the screen to find the package you want.
1. Click the triangle on the left of the listing line of the package you want
to hold, which expands the information for that package.
1. Now click on the icon to the left of the package name. A new icon with a
lock will replace the old icon, indicating that this package is to be held
during upgrades.
1. Click 'Apply Changes'.

![Locked packages][img_locked_packages]

## Set up a custom graph

Suppose you want to monitor the size of the PostgreSQL database on your
database servers, you may use tags to group these machines together. Now you
can setup a graph to provide information from all of these servers:

1. Click on your account name, then 'Graphs'.
1. Click the 'Add graph' link.
1. Complete the 'Create graph' form. In our example we would do something like:
    * Title: **PostgreSQL database size**
    * Provide a 'Y-axis title' and define the machines you want the graph created for.
    * Run as user: **postgres**
    * Code:

```no-highlight
#!/bin/bash
psql -tAc "select pg_database_size('postgres')"
```

Click 'Save'

![Create custom graph][img_create_custom_graph]

To view the graph, click on 'Computers', then 'Monitoring'. You can select the
monitoring period from the drop-down box at the top of the window.

## Ensure all computers with a given tag have a common list of packages installed

Manage them via a [package profile].

<!-- IMAGES -->
[img_pending_approval]: ../media/usecases1.png
[img_update_access_group]: ../media/accessgroups4.png
[img_locked_packages]: ../media/usecases2.png
[img_create_custom_graph]: ../media/usecases3.png

<!-- LINKS -->

[tags]: ./concepts.md#tags
[upgrade profiles]: ./concepts.md#upgrade-profiles
[package profile]: ./landscape-managing-packages.md#adding-a-package-profile

## Landscape-clients with configuration management tools

If want to manage landcape-client through a configuration managment tool such
as puppet, ansible, etc. you can avoid getting duplicate computers by writing
the ```/etc/landscape/client.conf``` and ```/etc/default/landscape-config```
files, and then restarting the landscape-client service.

/etc/landscape/client.conf
```no-highlight
[client]
log_level = info
url = https://landscape.canonical.com/message-system
ping_url = http://landscape.canonical.com/ping
data_path = /var/lib/landscape/client
registration_key = changeme
computer_title = my_machine
account_name = myaccount
include_manager_plugins = ScriptExecution
```

/etc/default/landscape
```no-highlight
RUN=1
```

The advantage over calling landscape-config is that this will request a
registration only if the client is not already registered against
landscape-server. Be aware, that some configuration options (namely
computer_title, tags, access_group) are only sent to landscape-server on
registration.
