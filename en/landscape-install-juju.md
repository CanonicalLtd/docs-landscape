# Juju deployment of Landscape OnPrem
With Juju you can deploy Landscape in a scalable way, and still have all services on a single machine if you want to.

Let's get ready to use Juju:
```
    sudo apt-get install juju
```

To learn more about Juju and to bootstrap a Juju controller, check out their [getting started](https://jujucharms.com/get-started) page.

## Using a bundle
A bundle is a file that contains a description of all the services that make up a deployment and their relation with each other. If you already have a suitable Juju environment configured, you can run this command to deploy Landscape:

```
    juju deploy cs:landscape-scalable
```
Landscape 18.03 will be deployed on 4 machines.

If you have a [MAAS](https://maas.io) server, you can take advantage of containers and use the `landscape-dense-maas` bundle:
```
    juju deploy cs:landscape-dense-maas
```
This will deploy Landscape 18.03 on just one node using LXD containers for all services.

Finally, the `landscape-dense` bundle is quite similar to the `dense-maas` one, but it installs the `haproxy` service directly on the machine without a container. All the other services use a container:
```
    juju deploy cs:landscape-dense
```
This is useful for the cases where the LXD containers don't get externally routable IP addresses.
