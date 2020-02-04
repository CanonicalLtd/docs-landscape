Title: Juju deployment of Landscape On-Prem
# Juju deployment of Landscape On-Prem
With Juju you can deploy Landscape in a scalable way, and still have all services on a single machine if you want to.

## Install Juju
Let's get ready to use Juju:
```
sudo apt-get install juju
```

To learn more about Juju and to bootstrap a Juju controller, check out their [getting started](https://jaas.ai/docs/getting-started-with-juju) page.

## Deploying OPL
When deploying with Juju, you will use a Juju bundle. A bundle is an encapsulation of all of the parts needed to deploy the required services as well as associated relations and configurations that the deployment requires. When deploying OPL using Juju, there are three different methods you can use. Select the one that meets the needs for your environment.

### landscape-dense-maas bundle

If you have a [MAAS](https://maas.io) server, you can take advantage of containers and use the `landscape-dense-maas` bundle:
```
juju deploy cs:landscape-dense-maas
```
This will deploy Landscape on just one node using LXD containers for all services.
### landscape-scalable bundle
**landscape-scalable** each service gets its own machine. Currently that means you will need 4 machines for Landscape, and one for the controller node:
```
juju deploy cs:landscape-scalable
```
### landscape-dense bundle
**landscape-dense** is quite similar to the `landscape-dense-maas` deployment, but it installs the `haproxy` service directly on the machine without a container. All the other services use a container:
```
juju deploy cs:landscape-dense
```
This is useful for the cases where the LXD containers don't get externally routable IP addresses.
## Accessing Landscape On-Premises
Once the deployment has finished, grab the address of the first `haproxy` unit and access it with your browser:
```
juju status haproxy
```
