Title: Landscape

# Welcome to Landscape Help

Landscape is a tool for remotely managing computers that run Ubuntu.

[User Guide](https://landscape.canonical.com/static/doc/user-guide/)

[API Documentation](https://landscape.canonical.com/static/doc/api/)

## Landscape On-Premises

Landscape On-Premises, is the standalone version of Landscape that you can install on your own network.

Each major Landscape version is supported for a period of one year after release. Here are the current supported releases:

| **major version**                | **Release date** | **Supported until** | **Version of Ubuntu** | 
| ----------------------           | ---------------- | ------------------- | --------------------- |
| [18.03](ReleaseNotes18.03.html)   | 2018-Jun         | **2019-Jun**            | 16.04 LTS ("xenial") or 18.04 LTS ("bionic") |
| 17.03                            | 2017-Mar         | **2019-Mar**            | 16.04 LTS ("xenial") |


### Installing On-Prem

Here is how you can get started:

 * **[Quickstart](LDS/QuickstartDeployment18.03)**, for when you don't have Juju but quickly want to check out On-Prem:

``` 
sudo add-apt-repository -u ppa:landscape/18.03
sudo apt-get install landscape-server-quickstart
```

 * **[Juju deployed](LDS/JujuDeployment18.03)** for a truly scalable deployment. Select the bundle that best serves your environment:

**landscape-dense-maas** if you have the MAAS provider, you can deploy all the services using containers:

``` 
juju deploy cs:bundle/landscape-dense-maas
```

**landscape-scalable** each service gets its own machine. Currently that means you will need 4 machines for Landscape, and one for the controller node:

```
juju deploy cs:bundle/landscape-scalable
```

Once the deployment has finished, grab the address of the first `haproxy` unit and access it with your browser:

```
juju status haproxy
```

* **[Manual installation](landscape-manual-install.html)**: for when you don't have a suitable Juju environment but need a scalable deployment.
