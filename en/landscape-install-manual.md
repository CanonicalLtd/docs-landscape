Title: Landscape 19.01 Manual Installation


# Manual Installation

This is the baseline deployment recommendation we have for the LDS product when Juju is not used. At a minimum, we have two machines:

 * a database server, running Ubuntu 18.04 LTS ("bionic"), with Postgresql 10.
 * an application server, running the same version of Ubuntu as the Database server, hosting the Landscape services
 
# Network access

 * the APP server needs http access to `usn.ubuntu.com` in order to download the USN database and detect security updates. Without this, the available updates won't be distinguished between security related and regular updates
 * the APP server also needs http access to the public Ubuntu archives and `changelogs.ubuntu.com`, in order to update the hash-id-database files and detect new distribution releases. Without this, the release upgrade feature won't work
 * the APP server also needs https access to `landscape.canonical.com` in order to query for available LDS releases. If this access is not given, the only drawback is that LDS won't display a note about the available releases in the account page.
 
This is a long document. Don't be intimidated. If you want a quick installation that just works, but doesn't scale to a large number of machines, then install the `landscape-server-quickstart` package.

# Preparing for the installation

What you will need:

 * Ubuntu 18.04 LTS ("bionic") server install media.
 * Landscape Dedicated Server license file. If you don't have one, a free demo license with a small number of seats will be used instead.
 * Server X509 certificate and key, signed by a publicly known Certificate Authority, and issued for the FQDN hostname of the application server.
 * Custom CAs can be used, but this is not documented here as it's considered an advanced topic. Administrators deploying custom CAs most likely know what needs to be done. In any case, this quick how-to may help:[LDS/SSL-creating a simple CA and issuing a certificate](https://help.landscape.canonical.com/LDS/SSL). You MUST use the same version of Ubuntu on both the Application server and the Database server.

# Installing the Database Server
After having installed the basic server profile of Ubuntu Server, we need to install the postgresql database and configure it for use by Landscape. Please follow these steps:

## Install postgresql and required libraries
In the database server, run this command to install the database software.

```
sudo apt-get install postgresql-10 python-apt postgresql-plpython-10 postgresql-contrib-10 postgresql-10-debversion
```

## Create a superuser Landscape can use
Landscape needs a database superuser in order to create the lower privilege users it needs to perform routine tasks and access the data, as well as alter the database schema whenever needed:
```
sudo -u postgres createuser --createdb --createrole --superuser --pwprompt landscape_superuser
```

Use a strong password! (But don't put a @ in it)

If this database is to be shared with other services, it is recommended that another cluster is created instead for those services (or for Landscape). Please refer to the Postgresql documentation in that case.

## Configure PostgreSQL
We now need to allow the application server to access this database server. Landscape uses several users for this access, so we need to allow them all. Edit the `/etc/postgresql/10/main/pg_hba.conf` file and add the following to the end:
```
host all landscape,landscape_maintenance,landscape_superuser <IP-OF-APP> md5
```
Replace `<IP-OF-APP>` with the IP address of the application server, followed by `/32`. Alternatively, you can specify the network address using the CIDR notation. Some examples of valid values:
 * `192.168.122.199/32`: the IP address of the APP server
 * `192.168.122.0/24`: a network address
Now come changes to the main postgresql configuration file. Edit `/etc/postgresql/10/main/postgresql.conf` and:
 * find the `listen_addresses` parameter, which is probably commented, and change it to:
```
listen_addresses = '*'
```
 * Set `max_prepared_transactions` to the same value as `max_connections`. For example:
```
max_connections = 400
...
max_prepared_transactions = 400
```
Finally, restart the database service:
```
sudo systemctl restart postgresql
```

## Tune PostgreSQL

It is strongly recommended to fine tune this PostgreSQL installation according to the hardware of the server. Keeping the default settings (especially of max_connections) is known to be problematic.  Please see the following resources for more information.

[Landscape Specific Postgresql Tuning Tips](https://help.landscape.canonical.com/LDS/PostgresqlTuning)

[General PostgreSQL Tips](http://wiki.postgresql.org/wiki/Tuning_Your_PostgreSQL_Server)

If this tuning changed the value of `max_connections`, make sure you also change `max_prepared_transactions` to the same value!

# Installing the application server
The application server will host the following Landscape services:
 * application server
 * message server
 * ping server
 * job handler
 * async-frontend
 * combo loader
 * api server
 * package upload service
 * package search
Additionally, other services needed by Landscape will also be running on this machine, such as:
 * apache
 * rabbitmq-server
Let's begin.

## Adding the Landscape package archive and installing the package
Landscape is distributed in a public PPA. You can add it to the system with these commands:
```
    sudo add-apt-repository ppa:landscape/19.01
    sudo apt-get update
```

Now install the server package and its dependencies:
```
sudo apt-get install landscape-server rabbitmq-server apache2
```

## Install the license file
If you were given a license file, copy it to `/etc/landscape`:
```
sudo cp license.txt /etc/landscape
```
Make sure it's readable by the `landscape` user and root.

If you have no such file, the package will use a limited free license.

## Configure rabbitmq
Just run the following commands, replacing `<password>` with a password of your choice. It will be needed later.
```
sudo rabbitmqctl add_user landscape <password>
sudo rabbitmqctl add_vhost landscape
sudo rabbitmqctl set_permissions -p landscape landscape ".*" ".*" ".*"
```

To make rabbitmq listen only on the loopback interface (127.0.0.1), please edit the file `/etc/rabbitmq/rabbitmq-env.conf` with the following content:
```
NODE_IP_ADDRESS=127.0.0.1
```

Then restart it:
```
sudo systemctl restart rabbitmq-server
```

## Configure database and broker access
We now need to make some configuration changes to the `/etc/landscape/service.conf` file to tell Landscape how to use some other services:
Please make the following changes:
 * section `[stores]`:
  * `host`: the IP or hostname of the database server.  if not the default Postgresql port (5432), add a :NNNN port definition after the hostname (e.g., 10.0.1.5:3232)
 * section `[broker]`:
  * replace the `password` value with the password chosen above when configuring rabbitmq
 * section `[schema]`:
  * change the value of `store_user` to the landscape super user we created above during the DB installation
  * add an entry for `store_password` with the password that was chosen in that same step

## Run the Landscape setup script
This script will bootstrap the databases Landscape needs to work and setup the remaining of the configuration:
```
sudo setup-landscape-server
```

 {i} Depending on the hardware, this may take several minutes to complete

## Configure Landscape services and schema upgrades
We need to enable the Landscape services now. Please edit `/etc/default/landscape-server` and change the `RUN_ALL` line to `yes`:
```
# To run all Landscape services set this to "yes"
RUN_ALL="yes"
```
 {i} If more performance and availability are needed out of LDS, it's possible to spread out the services amongst several machines. In that case, for example, one could run message servers in one machine, application servers in another one, etc.
The message, application and ping services can be configured to run multiple instances. If your hardware has several cores and enough memory (4Gb or more), running two or more of each will improve performance. To run multiple instances of a service, just set the value in the respective `RUN_` line to the number of instances. For example, if you want to run two message servers, just set:
```
RUN_MSGSERVER="2"
```
 {i} In order to take advantage of this multiple-instances setting, you need to configure some sort of load balancer or proxy. See the `README.multiple-services` file in the `landscape-server` package documentation directory for an example using Apache's `proxy_loadbalancer` module.

In that same file, the `UPGRADE_SCHEMA` option needs to be reviewed. If set to `yes`, whenever the package `landscape-server` is updated it will attempt to update the database schema too. It is a very convenient setting, but please think about the following before enabling it:
 * schema updates can take several minutes
 * if the package is updated while the database is offline, or unreachable, the update will fail
 * you should have a backup of the database before updating the package
Without this setting enabled, a package update might result in services that won't start anymore because of a needed schema change. In that case:
 * stop all the Landscape services
 * backup your database
 * run `sudo setup-landscape-server` on the application server. This will update the schema
 * start all Landscape services again

## Webserver configuration
Landscape uses Apache to, among other things, redirect requests to each service and provide SSL support. The usual way to do this in Ubuntu is to create a Virtual Host for Landscape.

Below is a suggested configuration file that does just that. Install it as `/etc/apache2/sites-available/landscape.conf` and change the following values:
 * `@hostname@`: the FQDN of the hostname the clients (browser and machines) will use to connect to LDS. This is what will be in the URL, and it needs to be resolvable via DNS. For example, `lds.example.com`
 * `@certfile@`: the full filesystem path to where the SSL certificate for this server is installed. For example, `/etc/ssl/certs/landscape_server.pem`
 * `@keyfile@`: the full filesystem path to where the corresponding private key of that certificate is installed. For example, `/etc/ssl/private/landscape_server.key`


 If you are using a custom certificate authority for your SSL certificate, then you '''MUST''' put the CA public certificate in the file `/etc/ssl/certs/landscape_server_ca.crt` and uncomment the `SSLCertificateChainFile /etc/ssl/certs/landscape_server_ca.crt` line.

 Make sure the user apache runs as can read those files! Also, make sure the private key can only be read by root and that same apache user.
 
```
<VirtualHost *:80>

    # This Hostname is the HTTP/1.1 hostname that users and Landscape clients will access
    # It must be the same as your SSL Certificate's CommonName
    # And the DNS Hostname for this machine
    # It is not recommended that you use an IP address here...
    ServerName @hostname@
    ServerAdmin webmaster@@hostname@
    ErrorLog /var/log/apache2/landscape.error-log
    CustomLog /var/log/apache2/landscape.access-log combined
    DocumentRoot /opt/canonical/landscape/canonical/landscape

    # Set a Via header in outbound requests to the proxy, so proxied apps can
    # know who the actual client is
    ProxyVia on
    ProxyTimeout 10

    <Directory "/">
      Options +Indexes
      Order deny,allow
      Allow from all
      Require all granted
      Satisfy Any
      ErrorDocument 403 /offline/unauthorized.html
      ErrorDocument 404 /offline/notfound.html
    </Directory>

    Alias /offline /opt/canonical/landscape/canonical/landscape/offline
    Alias /static /opt/canonical/landscape/canonical/static
    Alias /repository /var/lib/landscape/landscape-repository


    <Location "/repository">
      Order deny,allow
      Deny from all
      ErrorDocument 403 default
      ErrorDocument 404 default
    </Location>
   <LocationMatch "/repository/[^/]+/[^/]+/(dists|pool)/.*">
     Allow from all
   </LocationMatch>
   <Location "/icons">
        Order allow,deny
        Allow from all
   </Location>
   <Location "/ping">
        Order allow,deny
        Allow from all
    </Location>

    <Location "/message-system">
        Order allow,deny
        Allow from all 
    </Location>

   <Location "/static">
      Header always append X-Frame-Options SAMEORIGIN
   </Location>

   <Location "/r">
      FileETag none
      ExpiresActive on
      ExpiresDefault "access plus 10 years"
      Header append Cache-Control "public"
   </Location>

    RewriteEngine On

    RewriteRule ^/r/([^/]+)/(.*) /$2 

    RewriteRule ^/ping$ http://localhost:8070/ping [P]

    RewriteCond %{REQUEST_URI} !^/server-status
    RewriteCond %{REQUEST_URI} !^/icons
    RewriteCond %{REQUEST_URI} !^/static/
    RewriteCond %{REQUEST_URI} !^/offline/
    RewriteCond %{REQUEST_URI} !^/repository/
    RewriteCond %{REQUEST_URI} !^/message-system

    # Replace the @hostname@ with the DNS hostname for this machine.
    # If you change the port number that Apache is providing SSL on, you must change the 
    # port number 443 here.
    RewriteRule ^/(.*) https://@hostname@:443/$1 [R=permanent]
</VirtualHost>

<VirtualHost *:443>
    ServerName @hostname@
    ServerAdmin webmaster@@hostname@

    ErrorLog /var/log/apache2/landscape.error-log
    CustomLog /var/log/apache2/landscape.access-log combined

    DocumentRoot /opt/canonical/landscape/canonical/landscape

    SSLEngine On
    SSLCertificateFile @certfile@
    SSLCertificateKeyFile @keyfile@
    # If you have either an SSLCertificateChainFile or, a self-signed CA signed certificate
    # uncomment the line below.
    # Note: Some versions of Apache will not accept the SSLCertificateChainFile directive.
    # Try using SSLCACertificateFile instead in that case.
    # SSLCertificateChainFile /etc/ssl/certs/landscape_server_ca.crt
    # Disable to avoid POODLE attack
    SSLProtocol all -SSLv3 -SSLv2 -TLSv1
    SSLHonorCipherOrder On
    SSLCompression Off
    SSLCipherSuite EECDH+AESGCM+AES128:EDH+AESGCM+AES128:EECDH+AES128:EDH+AES128:ECDH+AESGCM+AES128:aRSA+AESGCM+AES128:ECDH+AES128:DH+AES128:aRSA+AES128:EECDH+AESGCM:EDH+AESGCM:EECDH:EDH:ECDH+AESGCM:aRSA+AESGCM:ECDH:DH:aRSA:HIGH:!MEDIUM:!aNULL:!NULL:!LOW:!3DES:!DSS:!EXP:!PSK:!SRP:!CAMELLIA:!DHE-RSA-AES128-SHA:!DHE-RSA-AES256-SHA:!aECDH

    # Try to keep this close to the storm timeout. Not less, maybe slightly
    # more
    ProxyTimeout 305

    <Directory "/">
      Options -Indexes
      Order deny,allow
      Allow from all
      Require all granted
      Satisfy Any
      ErrorDocument 403 /offline/unauthorized.html
      ErrorDocument 404 /offline/notfound.html
    </Directory>

    <Location "/ajax">
      Order allow,deny
      Allow from all
    </Location>

    Alias /offline /opt/canonical/landscape/canonical/landscape/offline
    Alias /config /opt/canonical/landscape/apacheroot
    Alias /hash-id-databases /var/lib/landscape/hash-id-databases

    ProxyRequests off
    <Proxy *>
       Order deny,allow
       Allow from all
       ErrorDocument 403 /offline/unauthorized.html
       ErrorDocument 500 /offline/exception.html
       ErrorDocument 502 /offline/unplanned-offline.html
       ErrorDocument 503 /offline/unplanned-offline.html
    </Proxy>

    ProxyPass /robots.txt !
    ProxyPass /favicon.ico !
    ProxyPass /offline !
    ProxyPass /static !

    ProxyPreserveHost on


   <Location "/r">
      FileETag none
      ExpiresActive on
      ExpiresDefault "access plus 10 years"
      Header append Cache-Control "public"
   </Location>

   <Location "/static">
      Header always append X-Frame-Options SAMEORIGIN
   </Location>

    RewriteEngine On

    RewriteRule ^/.*\+\+.* / [F]
    RewriteRule ^/r/([^/]+)/(.*) /$2

    # See /etc/landscape/service.conf for a description of all the
    # Landscape services and the ports they run on.
    # Replace the @hostname@ with the DNS hostname for this machine.
    # If you change the port number that Apache is providing SSL on, you must change the 
    # port number 443 here.
    RewriteRule ^/message-system http://localhost:8090/++vh++https:@hostname@:443/++/ [P,L]

    RewriteRule ^/ajax http://localhost:9090/ [P,L]
    RewriteRule ^/combo(.*) http://localhost:8080/combo$1 [P,L]
    RewriteRule ^/api http://localhost:9080/ [P,L]
    RewriteRule ^/attachment/(.*) http://localhost:8090/attachment/$1 [P,L]
    RewriteRule ^/upload/(.*) http://localhost:9100/$1 [P,L]

    RewriteCond %{REQUEST_URI} !^/robots.txt$
    RewriteCond %{REQUEST_URI} !^/favicon.ico$
    RewriteCond %{REQUEST_URI} !^/offline/
    RewriteCond %{REQUEST_URI} !^/(r/[^/]+/)?static/
    RewriteCond %{REQUEST_URI} !^/config/
    RewriteCond %{REQUEST_URI} !^/hash-id-databases/

    # Replace the @hostname@ with the DNS hostname for this machine.
    # If you change the port number that Apache is providing SSL on, you must change the 
    # port number 443 here.
    RewriteRule ^/(.*) http://localhost:8080/++vh++https:@hostname@:443/++/$1 [P]

    <Location /message-system>
      Order allow,deny
      Allow from all
    </Location>

    <Location />
        # Insert filter
        SetOutputFilter DEFLATE

        # Don't compress images or .debs
        SetEnvIfNoCase Request_URI \
        \.(?:gif|jpe?g|png|deb)$ no-gzip dont-vary

        # Make sure proxies don't deliver the wrong content
        Header append Vary User-Agent env=!dont-vary
    </Location>

</VirtualHost>
```
We now need to enable some modules:
```
for module in rewrite proxy_http ssl headers expires; do sudo a2enmod $module; done
```
Disable the default http vhost:
```
sudo a2dissite 000-default
```
Finally we can enable the new site:
```
sudo a2ensite landscape.conf
sudo service apache2 restart
```

## Start Landscape services
Just run the helper script `lsctl`:
```
sudo lsctl restart
```


## Setup first user
The first user that is created in LDS automatically becomes the administrator of the "standalone" account. To create it, please go to https://<servername> and fill in the requested information.

## Configuring the first client
On the client machine, after installing the `landscape-client` package, please run this command:
```
$ sudo landscape-config --computer-title "My First Computer" --account-name standalone --url https://<servername>/message-system --ping-url http://<servername>/ping
```

If you used a custom CA, you will need to pass the `--ssl-public-key` parameter pointing to the CA file so that the client can recognize the issuer of the server certificat.

You can now accept your client in the Landscape UI, and it begins to upload data.
 {i} If you configure an account password, the client will be automatically accepted when using that password.

## Email alias
We recommend adding an alias for user landscape on your local environment, to ensure that important system emails get attention.

```
$ sudo vim /etc/aliases
```

Add a line ```landscape: <insert recipient's email address>``` to this file and rebuild your aliases

```
$ sudo /usr/bin/newaliases
```
