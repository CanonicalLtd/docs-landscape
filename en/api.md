Title: The Landscape API

# The Landscape API

Landscape's API lets you perform many Landscape tasks from the command line or
a shell script, or a Python module. You can also use the API in HTTPS calls.

You can find API Getting Started information in the release notes, and help is
available from the API DOCUMENTATION link at the bottom of any Landscape page.
The instructions for installing the landscape-api command are available from
the API DOCUMENTATION link, but in brief, you can set up the API client by
running the following commands:

```bash
sudo add-apt-repository ppa:landscape/landscape-api
sudo apt-get update
sudo apt-get install landscape-api
```

In addition to the help information available from the Landscape GUI, the API
is documented in Landscape's online help. Once the landscape-api command is
installed you can get online help on all landscape-api commands using the
syntax landscape-api COMMAND -h.

Before you use the API, you must generate API credentials - that is, an API
access key and API secret key. To do so, click on your account name in the
upper right corner. On the settings screen that appears, click on Generate API
Credentials. You will need to use the key values thus generated with certain
commands. You specify them as command-line options, but it's easier to export
them as shell variable with commands like:

```bash
export LANDSCAPE_API_KEY="<API access key>"
export LANDSCAPE_API_SECRET="<API secret key>"
export LANDSCAPE_API_URI="https://<lds-hostname>/api/"
```

If you use a custom Certificate Authority (CA), you also need to export the
path to your certificate:

```bash
export LANDSCAPE_API_SSL_CA_FILE="/path/to/ca/file"
```

[Landscape API Reference](https://landscape.canonical.com/static/doc/api/)
