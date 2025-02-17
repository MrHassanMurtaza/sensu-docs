---
title: "Microsoft Windows"
description: "User documentation for installing and operating Sensu on Microsoft
  Windows."
weight: 4
version: "1.8"
product: "Sensu Core"
platformContent: true
menu:
  sensu-core-1.8:
    parent: platforms
---

# Sensu on Microsoft Windows

## Reference documentation

- [Installing Sensu Core](#sensu-core)
  - [Download and install Sensu using the Sensu MSI](#download-and-install-sensu-core)
- [Configure Sensu](#configure-sensu)
  - [Create the Sensu configuration directory](#create-the-sensu-configuration-directory)
  - [Example client configuration](#example-client-configuration)
  - [Example transport configuration](#example-transport-configuration)
  - [Configure the Sensu client Windows service wrapper](#configure-the-sensu-client-windows-service-wrapper)
  - [Install the Sensu client Windows service](#install-the-sensu-client-windows-service)
- [Operating Sensu](#operating-sensu)
  - [Managing the Sensu client Windows service](#service-management)

## Install Sensu Core {#sensu-core}

Sensu Core is installed on Microsoft Windows systems via a native system
installer package (i.e. a .msi file), which is available for download from the
[Sensu Downloads][1] page, and from [this repository][2].

### Download and install Sensu using the Sensu MSI {#download-and-install-sensu-core}

_NOTE: As of Sensu version 1.0, repository URLs have changed.
To install or upgrade to the latest version of Sensu, please ensure you have updated existing configurations to follow the repository URL format specified below._

1. Download Sensu from the [Sensu Downloads][1] page.

2. Double-click the `sensu-1.4.1-1-x64.msi` installer package to launch the
   installer, accept the Sensu Core [MIT License][4] and install Sensu using the
   default settings (e.g. install location, etc).
   _WARNING: changing the default installation path from `C:\opt` is strongly
   discouraged._

3. Configure the Sensu client. **No "default" configuration is provided with
   Sensu**, so the Sensu Client will not start without the corresponding
   configuration. Please refer to the ["Configure Sensu" section][12] (below)
   for more information on configuring Sensu. **At minimum, the Sensu client
   will need a working [transport definition][13] and [client definition][14]**.

## Configure Sensu

_WARNING: Many text editors on Windows, including Notepad, save text in a format that is not suitable for Sensu configuration.
While we require UTF-8, there are similar-looking character sets that are not actually 'UTF-8', such as 'UTF-8 BOM'.
A more modern text editor, such as Atom or Notepad++, will allow you to do this easily, fortunately.
For more about editor encodings, see [this discussion on StackOverflow][16].
If you're automating JSON creation with e.g. PowerShell, make sure that you're speaking UTF-8 in your shell, too! See [here][17] for more details._


By default, all of the Sensu services on Microsoft Windows systems will load
configuration from the following locations:

- `C:\opt\sensu\config.json`
- `C:\opt\sensu\conf.d\`

_WARNING: Ensure that the configure files you create have a `.json` file extension and not, for example, a `.json.txt` file extension.
Sensu will only load files that have a `.json` file extension._

_NOTE: in general, where references to configuration file locations found
elsewhere in the Sensu documentation suggest paths beginning with `/etc/sensu`,
these will correspond to `C:\opt\sensu` on Microsoft Windows systems.
Additional or alternative configuration file and directory locations may be used by
modifying Sensu's service configuration XML and/or by starting the Sensu services with the corresponding CLI arguments.
For more information, please consult the [Sensu Configuration][5] reference documentation._

The following Sensu configuration files are provided as examples. Please review
the [Sensu configuration reference documentation][5] for additional information
on how Sensu is configured.

### Create the Sensu configuration directory

In some cases, the default Sensu configuration directory (i.e.
`C:\opt\sensu\conf.d\`) is not created by the Sensu MSI installer, in which case
it is necessary to create this directory manually.

{{< highlight shell>}}
mkdir C:\opt\sensu\conf.d\{{< /highlight >}}

### Example client configuration

1. Copy the following contents to a configuration file located at
   `C:\opt\sensu\conf.d\client.json`:
   {{< highlight json >}}
{
  "client": {
    "name": "windows-client",
    "address": "127.0.0.1",
    "environment": "development",
    "subscriptions": [
      "dev",
      "windows-hosts"
    ],
    "socket": {
      "bind": "127.0.0.1",
      "port": 3030
    }
  }
}{{< /highlight >}}

### Example Transport Configuration

At minimum, the Sensu client process requires configuration to tell it how to
connect to the configured [Sensu Transport][6].

1. Copy the following contents to a configuration file located at
   `/etc/sensu/conf.d/transport.json`:
   {{< highlight json >}}
{
  "transport": {
    "name": "rabbitmq",
    "reconnect_on_error": true
  }
}{{< /highlight >}}
   _NOTE: if you are using Redis as your transport, please use `"name": "redis"`
   for your transport configuration.
   For more information, please visit the [transport definition specification][15]._

2. If the transport being used is running on a different host, additional configuration is required to tell the sensu client how to connect to the transport.
Please see [Redis][7] or [RabbitMQ][8] reference documentation for examples.

### Configure the Sensu client Windows service wrapper

The Sensu Core MSI package includes a Sensu client service wrapper, allowing
Sensu to be registered as a Windows service. The Sensu client service wrapper
uses an XML configuration file to configure the `sensu-client` run arguments
(e.g. `--log C:\opt\sensu\sensu-client.log`).

To configure the Sensu client service wrapper, edit the service definition file
at `C:\opt\sensu\bin\sensu-client.xml` with your favorite text editor. This XML
configuration file allows you to set [Sensu client CLI arguments][9]. The
following example configuration file sets the Sensu client primary configuration
file path to `C:\opt\sensu\config.json`, the Sensu configuration directory to
`C:\opt\sensu\conf.d`, and the log file path to `C:\opt\sensu\sensu-client.log`.

{{< highlight xml >}}
<!--
  Windows service definition for Sensu
-->
<service>
  <id>sensu-client</id>
  <name>Sensu Client</name>
  <description>This service runs a Sensu client</description>
  <executable>C:\opt\sensu\embedded\bin\ruby</executable>
  <arguments>C:\opt\sensu\embedded\bin\sensu-client -c C:\opt\sensu\config.json -d C:\opt\sensu\conf.d -l C:\opt\sensu\sensu-client.log</arguments>
</service>{{< /highlight >}}

### Install the Sensu client Windows service

Open an Administrative Command Prompt and use the [Windows SC][10] utility to create the Windows service for the Sensu client:

{{< highlight shell>}}
sc create sensu-client start= delayed-auto binPath= c:\opt\sensu\bin\sensu-client.exe DisplayName= "Sensu Client"{{< /highlight >}}

_NOTE: the space between the equals (=) and the values is required._

## Operating Sensu

### Managing the Sensu client Windows service {#service-management}

To manually start and stop the Sensu client Windows service, use the
[Services.msc][11] utility, or via the Command Prompt.

- Start or stop the Sensu client

{{< highlight shell >}}
sc start sensu-client
sc stop sensu-client{{< /highlight >}}


[1]:  https://sensuapp.org/download
[2]:  https://sensu.global.ssl.fastly.net/msi/
[3]:  https://sensu.global.ssl.fastly.net/msi/2012r2/sensu-1.4.1-1-x64.msi
[4]:  https://sensuapp.org/mit-license
[5]:  ../../reference/configuration/
[6]:  ../../reference/transport/
[7]:  ../../reference/redis/#configure-sensu
[8]:  ../../reference/rabbitmq/#sensu-rabbitmq-configuration
[9]:  ../../reference/configuration/#sensu-service-cli-arguments
[10]: https://technet.microsoft.com/en-us/library/bb490995.aspx
[11]: https://technet.microsoft.com/en-us/library/cc755249.aspx
[12]: #configure-sensu
[13]: #example-transport-configuration
[14]: #example-client-configuration
[15]: ../../reference/transport/#transport-definition-specification
[16]: http://stackoverflow.com/questions/2223882/whats-different-between-utf-8-and-utf-8-without-bom
[17]: http://stackoverflow.com/questions/5596982/using-powershell-to-write-a-file-in-utf-8-without-the-bom
