# Quickstart deployment
The so called quickstart mode of deploying Landscape consists of installing all the necessary software on a single machine. This is very handy for quickly checking out a new version of Landscape when you don't have Juju, but should not be used for production deployments because it can't be scaled.

Advantages:

 * quick installation suitable for demo purposes or a small number of registered computers
 * simple layout: everything on one machine

Disadvantages;

 * single process per application
 * no HA
 * no horizontal scaling
 * not recommended when using more than 500 clients

To install Landscape 19.01 using quickstart on Ubuntu 18.04 LTS ("bionic"), follow these simple steps:
```
    sudo add-apt-repository ppa:landscape/19.01
    sudo apt-get update
    sudo apt-get install landscape-server-quickstart
```
If you have a valid LDS license, copy it over to `/etc/landscape/license.txt` and restart the services. Otherwise, a free license with 10+50 seats (bare metal plus LXC containers) will be used:
```
    sudo cp license.txt /etc/landscape/license.txt
    sudo lsctl restart
```

## Setup first user
The first user that is created in LDS automatically becomes the administrator of the "standalone" account. To create it, please go to https://<servername> and fill in the requested information.
== Registering clients ==
In order to register a computer with LDS, you need to install the `landscape-client` package:
```
    sudo apt-get update
    sudo apt-get install landscape-client
```

The quickstart package generates and installs a self-signed SSL certificate in `/etc/ssl/certs/landscape_server_ca.crt` using the FQDN of the host for the `commonName` field of the certificate. A copy of this file will be needed on each computer that you register with LDS.

On each computer, copy that certificate over to, say, `/etc/landscape/server.pem` and add this line to the configuration file `/etc/landscape/client.conf`:
```
    ssl_public_key = /etc/landscape/server.pem
```
Then proceed with the registration request. Replace `<server>` with the FQDN of the quickstart host:
```
    sudo landscape-config --account-name standalone --url https://<server>/message-system --ping-url http://<server>/ping
```

If you get registration errors on the client, the reason why it failed will most likely be in the `/var/log/landscape/broker.log` log file. If it's SSL related, double check:

 * `ssl_public_key` in `/etc/landscape/client.conf` should be pointing at a copy of the server self-signed certificate
 * `<server>` in the URL of the `landscape-config` command-line must match the server hostname as used in the certificate. Check the outputs of `hostname -f` and `hostname` on the server
 * check the `commonName` field of the certificate with `openssl x509 -in <certificate-file> -noout -subject`
