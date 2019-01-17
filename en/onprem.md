## Landscape On-Premises

Landscape On-Premises, is the standalone version of Landscape that you can install on your own network.

Each major Landscape version is supported for a period of one year after release. Here are the current supported releases:

| **major version**                | **Release date** | **Supported until** | **Version of Ubuntu**  |
| ----------------------           | ---------------- | ------------------- | ---------------------  |
| [19.01](./ReleaseNotes19.01.md)  | 2019-Jan         | **2020-Jan**        | 18.04 LTS              |
| [18.03](./ReleaseNotes18.03.md)  | 2018-Jun         | **2019-Jun**        | 16.04 LTS or 18.04 LTS |
| [17.03](./ReleaseNotes17.03.md)  | 2017-Mar         | **2019-Mar**        | 16.04 LTS              |


### Installing On-Prem

Here is how you can get started:

 * **[Quickstart](./landscape-install-quickstart.md)**, for when you don't have Juju but quickly want to check out On-Prem. Not recommended for production environments when having more than 500 clients.

``` 
sudo add-apt-repository -u ppa:landscape/18.03
sudo apt-get install landscape-server-quickstart
```

 * **[Juju deployed](./landscape-install-juju.md)** for a truly scalable deployment. Select the bundle that best serves your environment:

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

* **[Manual installation](./landscape-install-manual.md)**: for when you don't have a suitable Juju environment but need a scalable deployment.
