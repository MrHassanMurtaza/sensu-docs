---
title: "Solaris"
description: "User documentation for installing and operating Sensu on Solaris
  systems."
weight: 5
version: "1.8"
product: "Sensu Core"
platformContent: true
menu:
  sensu-core-1.8:
    parent: platforms
---

# Sensu on Solaris

## Reference documentation

- [Installing Sensu Core](#sensu-core)
  - [Download and install Sensu on Solaris 10](#download-and-install-sensu-core-on-solaris-10)
  - [Download and install Sensu on Solaris 11](#download-and-install-sensu-core-on-solaris-11)
- [Configure Sensu](#configure-sensu)
  - [Create the Sensu configuration directory](#create-the-sensu-configuration-directory)
  - [Example client configuration](#example-client-configuration)
  - [Example transport configuration](#example-transport-configuration)
- [Operating Sensu](#operating-sensu)
  - [Managing the Sensu client process](#service-management)

## Install Sensu {#sensu-core}

Sensu Core is installed on Solaris systems via native system installer packages
(i.e. .pkg or [IPS][13] .p5p files), which are available for download from
the [Sensu Downloads][1] page, and from these repositories: [Solaris 10
(.pkg)][2], and [Solaris 11 (IPS .p5p)][3].

### Download and install Sensu on Solaris 10 {#download-and-install-sensu-core-on-solaris-10}

_NOTE: As of Sensu version 1.0, package repository URLs have changed.
To install or upgrade to the latest version of Sensu, please ensure
you have updated existing configurations to follow the repository URL
format specified below._

1. Download Sensu from the [Sensu Downloads][1] page

2. Install the `sensu-1.4.1-1.i386.pkg` package using the `pkgadd` utility:
   {{< highlight shell >}}
$ su
$ pkgadd -d sensu-1.4.1-1.i386.pkg{{< /highlight >}}

3. Install the Sensu service init script(s) using the `svccfg` utility:
   {{< highlight shell >}}
svccfg import /lib/svc/manifest/site/sensu-client.xml{{< /highlight >}}

4. Configure the Sensu client. **No "default" configuration is provided with
   Sensu**, so the Sensu client will not run without the corresponding
   configuration. Please refer to the ["Configure Sensu" section][9] (below),
   for more information on configuring Sensu. **At minimum, the Sensu client
   will need a working [transport definition][10] and [client definition][11]**.

### Download and install Sensu on Solaris 11 {#download-and-install-sensu-core-on-solaris-11}

1. Download Sensu from the [Sensu Downloads][1] page, or by using the `wget`
   utility:
   {{< highlight shell >}}
wget https://sensu.global.ssl.fastly.net/solaris/ips/5.11/sensu-1.4.1-1.i386.p5p{{< /highlight >}}

2. Install the `sensu-1.4.1-1.i386.p5p` package using the `pkg` utility:
   {{< highlight shell >}}
$ sudo pkg install -g sensu-1.4.1-1.i386.p5p developer/versioning/sensu{{< /highlight >}}

3. Download and run the Sensu [post-install script][12]:
   {{< highlight shell >}}
$ wget https://sensuapp.org/docs/1.4/files/postinst.sh
$ chmod +x postinst.sh
$ sudo ./postinst.sh{{< /highlight >}}
   _NOTE: all native system installer packages for Sensu contain this
   post-install script, which is used for setting up the `sensu` system user and
   group, creating various configuration directories, setting configuration
   directory permissions, and creating service init scripts. This step <span
   class='strike'>may</span> will be included in future Solaris 11 packages,
   however at this time it is necessary to perform these steps manually._

4. Install the Sensu service init script(s) using the `svcadm` utility:
   {{< highlight shell >}}
$ sudo svcadm restart manifest-import{{< /highlight >}}

5. Configure the Sensu client. **No "default" configuration is provided with
   Sensu**, so the Sensu client will not run without the corresponding
   configuration. Please refer to the ["Configure Sensu" section][9] (below),
   for more information on configuring Sensu. **At minimum, the Sensu client
   will need a working [transport definition][10] and [client definition][11]**.

## Configure Sensu

By default, all of the Sensu services on Solaris systems will load configuration
from the following locations:

- `/etc/sensu/config.json`
- `/etc/sensu/conf.d/`

_NOTE: additional or alternative configuration file and directory locations may
be used by modifying Sensu's service configuration and/or by starting the Sensu
services with the corresponding CLI arguments. For more information, please
consult the [Sensu Configuration][5] reference documentation._

### Create the Sensu configuration directory

In some cases, the default Sensu configuration directory (i.e.
`/etc/sensu/conf.d/`) is not created by the Sensu installer, in which case it is
necessary to create this directory manually.

{{< highlight shell >}}
mkdir /etc/sensu/conf.d{{< /highlight >}}

### Example client configuration

1. Copy the following contents to a configuration file located at
   `/etc/sensu/conf.d/client.json`:
   {{< highlight json >}}
{
  "client": {
    "name": "solaris-client",
    "address": "127.0.0.1",
    "environment": "development",
    "subscriptions": [
      "dev",
      "solaris-hosts"
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
   for your transport configuration. For more information, please visit the
   [transport definition specification][10]._

2. If the transport being used is running on a different host, additional configuration is required to tell the sensu client how to connect to the transport.
Please see [Redis][7] or [RabbitMQ][8] reference documentation for examples.

## Operating Sensu

### Managing the Sensu client process {#service-management}

Manually start, stop, and restart the Sensu services using the `svcadm` utility:

{{< highlight shell >}}
$ svcadm enable sensu-client
$ svcadm disable sensu-client
$ svcadm restart sensu-client{{< /highlight >}}

[?]:  #
[1]:  https://sensuapp.org/download
[2]:  https://sensu.global.ssl.fastly.net/solaris/pkg/
[3]:  https://sensu.global.ssl.fastly.net/solaris/ips/
[4]:  https://sensuapp.org/mit-license
[5]:  ../../reference/configuration/
[6]:  ../../reference/transport/
[7]:  ../../reference/redis/#configure-sensu
[8]:  ../../reference/rabbitmq/#sensu-rabbitmq-configuration
[9]:  #configure-sensu
[10]: #example-transport-configuration
[11]: #example-client-configuration
[12]: ../../files/postinst.sh
[13]: http://www.oracle.com/technetwork/server-storage/solaris11/technologies/ips-323421.html
