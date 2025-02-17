---
title: "Checks"
description: "Reference documentation for Sensu Checks."
product: "Sensu Core"
version: "1.8"
weight: 3
menu:
  sensu-core-1.8:
    parent: reference
---

## Reference documentation

- [What is a Sensu check?](#what-is-a-sensu-check)
  - [Sensu check specification](#sensu-check-specification)
- [Check commands](#check-commands)
  - [What is a check command?](#what-is-a-check-command)
  - [Check command arguments](#check-command-arguments)
  - [How and where are check commands executed?](#how-and-where-are-check-commands-executed)
- [Check execution platform](#check-execution-platform)
  - [How are checks scheduled?](#how-are-checks-scheduled)
    - [Subscription checks](#subscription-checks)
    - [Standalone checks](#standalone-checks)
- [Check results](#check-results)
  - [What is a check result?](#what-is-a-check-result)
  - [Example check result output](#example-check-result-output)
- [Check token substitution](#check-token-substitution)
  - [What is check token substitution?](#what-is-check-token-substitution)
  - [Example check tokens](#example-check-tokens)
  - [Check token specification](#check-token-specification)
- [Check hooks](#check-hooks)
  - [What are check hooks?](#what-are-check-hooks)
  - [Example check hooks](#example-check-hooks)
- [Managing checks](#managing-checks)
- [Check configuration](#check-configuration)
  - [Example check definition](#example-check-definition)
  - [Check definition specification](#check-definition-specification)
    - [Check naming](#check-names)
    - [`CHECK` attributes](#check-attributes)
    - [`subdue` attributes](#subdue-attributes)
    - [`influxdb` attributes](#influxdb-attributes)
    - [`opsgenie` attributes](#opsgenie-attributes)
    - [`hooks` attributes](#hooks-attributes)
    - [Custom attributes](#custom-attributes)
  - [Check result specification](#check-result-specification)
    - [`check` attributes](#check-result-check-attributes)
    - [`client` attributes](#check-result-client-attributes)

## What is a Sensu check?

Sensu checks are commands executed by the [Sensu client][1] which monitor a
condition (e.g. is Nginx running?) or collect measurements (e.g. how much disk
space do I have left?). Although the Sensu client will attempt to execute any
command defined for a check, successful processing of check results requires
adherence to a simple specification.

### Sensu check specification

* Result data is output to [STDOUT or STDERR][2]
  * For standard checks this output is typically a human-readable message
  * For metrics checks this output contains the measurements gathered by the
    check
* Exit status code indicates state
  * `0` indicates "OK"
  * `1` indicates "WARNING"
  * `2` indicates "CRITICAL"
  * exit status codes other than `0`, `1`, or `2` indicate an "UNKNOWN" or
    custom status

_PRO TIP: Those familiar with the [Nagios][3] monitoring system may recognize
this specification, as it is the same one used by Nagios plugins. As a result,
Nagios plugins can be used with Sensu without any modification._

At every execution of a check command &ndash; regardless of success or failure
&ndash; the Sensu client publishes the check's [result][4] for eventual handling
by the [event processor][5] (i.e. the [Sensu server][6].

## Check commands

### What is a check command?

Each [Sensu check definition][7] defines a `command` and the interval at which
it  should be executed. Check commands are literally executable commands which
will be executed on the [Sensu client][1], run as the `sensu` user. Most check
commands are provided by [Sensu check plugins][8].

### Check command arguments

Sensu check `command` attributes may include command line arguments for
controlling the behavior of the command executable. Most [Sensu check
plugins][8] provide support for command line arguments for reusability.

### How and where are check commands executed?

As mentioned above, all check commands are executed by [Sensu clients][1] as the
`sensu` user. Commands must be executable files that are discoverable on the
Sensu client system (i.e. installed in a system [`$PATH` directory][9]).

_NOTE: By default, the Sensu installer packages will modify the system `$PATH`
for the Sensu processes to include `/etc/sensu/plugins`. As a result, executable
scripts (e.g. plugins) located in `/etc/sensu/plugins` will be valid commands.
This allows `command` attributes to use "relative paths" for Sensu plugin
commands; <br><br>e.g.: `"command": "check-http.rb -u https://sensuapp.org"`_

## Check execution platform

### How are checks scheduled?

Sensu offers two distinct check execution schedulers: the [Sensu
server](../server), and the [Sensu client][10] (monitoring agent).
The Sensu server schedules and publishes check execution requests to client
subscriptions via a [Publish/Subscribe model][11] (i.e. [subscription
checks](#subscription-checks)). The Sensu client (monitoring agent) schedules and
executes [standalone checks][12] (on the local system only).
Because Sensu’s execution schedulers are not <abbr title="in other words, you
don't have to choose one or the other - you can use both">mutually
exclusive</abbr>, any Sensu client may be configured to both schedule and
execute it's own standalone checks as well as execute subscription checks
scheduled by the Sensu server.

#### Subscription checks

Sensu checks which are centrally defined and scheduled by the [Sensu server][6]
are called "subscription checks". Sensu’s use of a [message bus (transport)][13]
for communication enables [topic-based communication][14] &mdash; where messages
are published to a specific "topic", and consumers _subscribe_ to one or more
specific topics. This form of communication is commonly referred to as the
["publish-subscribe pattern"][11], or "pubsub" for short.

Subscription checks have a defined set of [subscribers][15], a list of
[transport][13] [topics][14] that check requests will be published to. Sensu
clients become subscribers to these topics (i.e. subscriptions) via their
individual [client definition][16] `subscriptions` attribute. In practice,
subscriptions will typically correspond to a specific role and/or responsibility
(e.g. a webserver, database, etc).

Subscriptions are a powerful primitives in the monitoring context because they
allow you to effectively monitor for specific behaviors or characteristics
corresponding to the function being provided by a particular system. For
example, disk capacity thresholds might be more important (or at least
different) on a database server as opposed to a webserver; conversely, CPU
and/or memory usage thresholds might be more important on a caching system than
on a file server. Subscriptions also allow you to configure check requests for
an entire group or subgroup of systems rather than require a traditional 1:1
mapping.

#### Standalone checks

Sensu checks which are defined on a [Sensu client][1] with the [check definition
attribute][17] `standalone` set to `true` are called "standalone checks". The
Sensu client provides its own [scheduler][10] for scheduling standalone checks
which ensures <abbr title='typically within 500ms'>scheduling
consistency</abbr> between Sensu clients with identical check definitions
(assuming that system clocks are synchronized via [NTP][18]).

Standalone checks are an important complement to [subscription checks][19]
because they provide a de-centralized management alternative for Sensu.

## Check results

### What is a check result?

A check result is a [JSON][20] document published as a message on the [Sensu
transport][13] by the Sensu client upon completion of a check execution. Sensu
check results include the [check definition attributes][17] (e.g. `command`,
`subscribers`, `interval`, `name`, etc; including [custom attributes][21]), the
client name the result was submitted from, and the `output` of the check.

### Example check result output

{{< highlight json >}}
{
  "check": {
    "status": 0,
    "command": "check-http.rb -u https://sensuapp.org",
    "subscribers": [
      "demo"
    ],
    "interval": 60,
    "name": "sensu-website",
    "issued": 1458934742,
    "executed": 1458934742,
    "duration": 0.637,
    "output": "CheckHttp OK: 200, 78572 bytes\n"
  },
  "client": "sensu-docs"
}
{{< /highlight >}}

_NOTE: please refer to the [check result specification][38] (below) for more
information about check results._

## Check token substitution

### What is check token substitution?

Sensu [check definitions][17] may include attributes that you may wish to
override on a client-by-client basis. For example, [check commands][23] &ndash;
which may include [command line arguments][23] for controlling the behavior of
the check command &ndash; may benefit from client-specific thresholds, etc.
Sensu check tokens are check definition placeholders that will be replaced by
the Sensu client with the corresponding [client definition attribute][16] values
(including [custom attributes][24]).

_NOTE: Sensu check tokens were formerly known as "check command tokens"
(which limited token substitution to the check `command` attribute); command
tokens were also sometimes referred to as **"Sensu client overrides"**; a
reference to the fact that command tokens allowed client attributes to
"override" [check command arguments][23]._

_NOTE: Check tokens are processed before check execution, therefore token
substitution will not apply to check data delivered via the local [client
socket input][46]._

### Example check tokens

The following is an example Sensu [check definition][17] using three check
tokens for [check command arguments][23]. In this example, the
`check-disk-usage.rb` command accepts `-w` (warning) and `-c` (critical)
arguments to indicate the thresholds (as percentages) for creating warning or
critical events. As configured, this check will create a warning event at 80%
disk capacity, unless a different threshold is provided by the client definition
(i.e. `:::disk.warning|80:::`); and a critical event will be created if disk
capacity reaches 90%, unless a different threshold is provided by the client
definition (i.e. `:::disk.critical|90:::`). This example also creates a custom
check definition attribute called `environment`, which will default to a value
of `production`, unless a different value is provided by the client definition
(i.e. `:::environment|production:::`).

{{< highlight json >}}
{
  "checks": {
    "check_disk_usage": {
      "command": "check-disk-usage.rb -w :::disk.warning|80::: -c :::disk.critical|90:::",
      "subscribers": [
        "production"
      ],
      "interval": 60,
      "environment": ":::environment|production:::"
    }
  }
}
{{< /highlight >}}

The following example [Sensu client definition][16] would provide the necessary
attributes to override the `disk.warning`, `disk.critical`, and `environment`
tokens declared above.

{{< highlight json >}}
{
  "client": {
    "name": "i-424242",
    "address": "10.0.2.100",
    "subscriptions": [
      "production",
      "webserver",
      "mysql"
    ],
    "disk": {
      "warning": 75,
      "critical": 85
    },
    "environment": "development"
  }
}
{{< /highlight >}}

### Check token specification

#### Token substitution syntax

Check tokens are invoked by wrapping references to client attributes with
"triple colons" (i.e. three colon characters, i.e. `:::`). Nested Sensu [client
definition attributes][16] can be accessed via "dot notation" (e.g.
`disk.warning`).

- `:::address:::` would be replaced with the [client `address` attribute][26]
- `:::url:::` would be replaced with a [custom attribute][24] called `url`
- `:::disk.warning:::` would be replaced with a [custom attribute][24] called
  `warning` nested inside of a JSON hash called `disk`

#### Token substitution default values {#check-token-default-values}

Check token default values can be used as a fallback in the event that an
attribute is not provided by the [client definition][16]. Check token default
values are separated by a pipe character (`|`), and can be used to provide a
"fallback value" for clients that are missing the declared token attribute.

- `:::url|https://sensuapp.org:::` would be replaced with a [custom
  attribute][24] called `url`. If no such attribute called `url` is included in
  the client definition, the default (or fallback) value of
  `https://sensuapp.org` will be used.

#### Unmatched tokens

If a [token substitution default value][25] is not provided (i.e. as a fallback
value), _and_ the Sensu client definition does not have a matching definition
attribute, a [check result][4] indicating "unmatched tokens" will be published
for the check execution (e.g.: `"Unmatched check token(s): disk.warning"`).

#### Token data type limitations

As part of the substitution process, Sensu converts all tokens to strings.
This means that tokens cannot be used for bare integer values or to access individual list items.

For example, token substitution **cannot** be used for specifying a check interval because the interval attribute requires an _integer_ value. But token substitution **can** be used for alerting thresholds since those values are included within the command _string_.

**Invalid use of token substitution:**

The resulting interval value is a string (`"60"`) instead of an integer (`60`), causing an invalid check request.

{{< highlight json >}}
{
  "checks": {
    "check_disk_usage": {
      "command": "check-disk-usage.rb -w 80 -c 90",
      "interval": ":::interval|60:::"
    }
  }
}
{{< /highlight >}}

**Valid use of token substitution:**

The resulting tokens are included within the command string, producing a valid check request.

{{< highlight json >}}
{
  "checks": {
    "check_disk_usage": {
      "command": "check-disk-usage.rb -w :::disk.warning|80::: -c :::disk.critical|90:::",
      "interval": 60
    }
  }
}
{{< /highlight >}}

When accessing a list using token substitution, Sensu returns the list in a fixed format.
For example, for the following check definition executed on a client with the subscriptions `["sensu", "rhel", "all"]`:

{{< highlight shell >}}
{
  "checks": {
    "token_test": {
      "command": "echo ':::subscriptions:::'",
      "standalone": true,
      "interval": 15
    }
  }
}
{{< /highlight >}}

The resulting command output would be:

{{< highlight shell >}}
[\"sensu\", \"rhel\", \"all\"]
{{< /highlight >}}

## Check hooks

### What are check hooks? {#what-are-check-hooks}

Check hooks are commands run by the Sensu client in response to the result of check command execution. The Sensu client will execute the appropriate configured hook command, depending on the check execution status (e.g. 1). Valid hook names include (in order of precedence): “1”-“255”, “ok”, “warning”, “critical”, “unknown”, and “non-zero”. The check hook command output, status, executed timestamp, and duration are captured and published in the check result. Check hook commands can optionally receive JSON serialized Sensu client and check definition data via STDIN.

### Example check hooks

Check hooks can be used for automated data gathering for incident triage, for example, a check hook could be used to capture the process tree when a process has been determined to be not running etc.

{{< highlight json >}}
{
  "checks": {
    "nginx_process": {
      "command": "check-process.rb -p nginx",
      "subscribers": [
        "proxy"
      ],
      "interval": 60,
      "hooks": {
        "non-zero": {"command": "ps aux"}
      }
    }
  }
}
{{< /highlight >}}

Check hooks can also be used to add context to a check result, for example,
outputting the last 100 lines of an error log.

{{< highlight json >}}
{
  "checks": {
    "nginx_process": {
      "command": "check-process.rb -p nginx",
      "subscribers": [
        "proxy"
      ],
      "interval": 60,
      "hooks": {
        "critical": {"command": "tail -n 100 /var/log/nginx/error.log"}
      }
    }
  }
}
{{< /highlight >}}

## Managing checks

### Delete a check

To delete a check from Sensu, remove the configuration file for the check, then restart the Sensu server and API.
The following example deletes a check named `check-sensu-website`:

{{< highlight shell >}}
# Delete the configuration file
sudo rm /etc/sensu/conf.d/check-sensu-website.json
# Restart the Sensu server and API
sudo systemctl restart sensu-{server,api}
{{< /highlight >}}

_NOTE: On Ubuntu 14.04, CentOS 6, and RHEL 6, use `sudo service sensu-{server,api} restart` to restart the Sensu server and API._

While removing the configuration file removes the check from the registry, it doesn't affect the history of the check or any existing check results.
To remove a deleted check's results and history from Sensu, use the [Checks API DELETE endpoint][66].
The following example deletes all check results and check history for a check named `check-sensu-website`:

{{< highlight shell >}}
# Delete check results and check history
curl -s -i -X DELETE http://127.0.0.1:4567/checks/check-sensu-website
{{< /highlight >}}

### Rename a check

To rename a check, modify the [check name](#check-names) in the check configuration, then restart the Sensu server and API and use the [Checks API DELETE endpoint][66] to remove the history and check results tied to the former name.

## Check configuration

### Example check definition

The following is an example Sensu check definition, a JSON configuration file
located at `/etc/sensu/conf.d/check-sensu-website.json`. This check definition
uses a [Sensu plugin][27] named [`check-http.rb`][28] to ensure that the Sensu
website is still available. The check is named `sensu-website` and it runs on
Sensu clients with the `production` [subscription][15], at an `interval` of 60
seconds.

{{< highlight json >}}
{
  "checks": {
    "sensu-website": {
      "command": "check-http.rb -u https://sensuapp.org",
      "subscribers": [
        "production"
      ],
      "interval": 60,
      "contact": "ops"
    }
  }
}
{{< /highlight >}}

### Check definition specification

#### Check naming {#check-names}

Each check definition has a unique check name, used for the definition key.
Every check definition is within the `"checks": {}` [configuration scope][29].

- A unique string used to name/identify the check
- Cannot contain special characters or spaces
- Validated with [Ruby regex][30] `/^[\w\.-]+$/.match("check-name")`

#### `CHECK` attributes

The following attributes are configured within the `{"checks": { "CHECK": {} }
}` [configuration scope][29] (where `CHECK` is a valid [check name][41]).

type            | 
----------------|------
description     | The check type, either `standard` or `metric`. Setting type to `metric` will cause OK (exit 0) check results to create events.
required        | false
type            | String
allowed values  | `standard`, `metric`
default         | `standard`
example         | {{< highlight shell >}}"type": "metric"{{< /highlight >}}

command      | 
-------------|------
description  | The check command to be executed.
required     | true (unless `extension` is configured)
type         | String
example      | {{< highlight shell >}}"command": "/etc/sensu/plugins/check-chef-client.rb"{{< /highlight >}}

extension    | 
-------------|------
description  | The name of a Sensu check extension to run instead of a command. This is an _advanced feature_ and is not commonly used.
required     | true (unless `command` is configured)
type         | String
example      | {{< highlight shell >}}"extension": "system_profile"{{< /highlight >}}

standalone   | 
-------------|------
description  | If the check is scheduled by the local Sensu client instead of the Sensu server (standalone mode).
required     | false
type         | Boolean
default      | false
example      | {{< highlight shell >}}"standalone": true{{< /highlight >}}

subscribers  | 
-------------|------
description  | An array of Sensu client subscriptions that check requests will be sent to. The array cannot be empty and its items must each be a string.
required     | true (unless `standalone` is `true`)
type         | Array
example      | {{< highlight shell >}}"subscribers": ["production"]{{< /highlight >}}

publish      | 
-------------|------
description  | If check requests are published for the check. If `standalone` is `true`, setting `publish` to `false` prevents the Sensu client from scheduling the check automatically.
required     | false
type         | Boolean
default      | true
example      | {{< highlight shell >}}"publish": false{{< /highlight >}}

interval     | 
-------------|------
description  | The frequency in seconds the check is executed.
required     | true (unless `publish` is `false` or `cron` is configured)
type         | Integer
example      | {{< highlight shell >}}"interval": 60{{< /highlight >}}

cron         | 
-------------|------
description  | When the check should be executed, using the [Cron syntax][47].
required     | true (unless `publish` is `false` or `interval` is configured)
type         | String
example      | {{< highlight shell >}}"cron": "0 0 * * *"{{< /highlight >}}

timeout      | 
-------------|------
description  | The check execution duration timeout in seconds (hard stop).
required     | false
type         | Integer
example      | {{< highlight shell >}}"timeout": 30{{< /highlight >}}

stdin        | 
-------------|------
description  | If the Sensu client writes JSON serialized Sensu client and check data to the command process’ STDIN. The command must expect the JSON data via STDIN, read it, and close STDIN. This attribute cannot be used with existing Sensu check plugins, nor Nagios plugins etc, as the Sensu client will wait indefinitely for the check process to read and close STDIN.
required     | false
type         | boolean
example      | {{< highlight shell >}}"stdin": true{{< /highlight >}}

ttl          | 
-------------|------
description  | The time to live (TTL) in seconds until check results are considered stale. If a client stops publishing results for the check, and the TTL expires, an event will be created for the client. The check `ttl` must be greater than the check `interval`, and should accommodate time for the check execution and result processing to complete. For example, if a check has an `interval` of `60` (seconds) and a `timeout` of `30` (seconds), an appropriate `ttl` would be a minimum of `90` (seconds).
required     | false
type         | Integer
example      | {{< highlight shell >}}"ttl": 100{{< /highlight >}}

ttl_status   | 
-------------|------
description  | The exit code that a check with the `ttl` attribute should return.
required     | false
type         | Integer
default      | 1
example      | {{< highlight shell >}}"ttl_status": 2{{< /highlight >}}

auto_resolve | 
-------------|------
description  | When a check in a `WARNING` or `CRITICAL` state returns to an `OK` state, the event generated by the `WARNING` or `CRITICAL` state will be automatically resolved. Setting `auto_resolve` to `false` will prevent this automatic event resolution from occurring. This is useful in situations where you want to explicitly require manual resolution of an event, e.g. via the API or a dashboard.
required     | false
type         | Boolean
default      | true
example      | {{< highlight shell >}}"auto_resolve": false{{< /highlight >}}

force_resolve | 
--------------|------
description   | Setting `force_resolve` to `true` on a check result ensures that the event is resolved and removed from the registry, regardless of the current event action. This attribute is used internally by [Sensu's `/resolve` API][45].
required      | false
type          | Boolean
default       | false
example       | {{< highlight shell >}}"force_resolve": true{{< /highlight >}}

handle       | 
-------------|------
description  | If events created by the check should be handled.
required     | false
type         | Boolean
default      | true
example      | {{< highlight shell >}}"handle": false{{< /highlight >}}

handler      | 
-------------|------
description  | The Sensu event handler (name) to use for events created by the check.
required     | false
type         | String
example      | {{< highlight shell >}}"handler": "pagerduty"{{< /highlight >}}

handlers     | 
-------------|------
description  | An array of Sensu event handlers (names) to use for events created by the check. Each array item must be a string.
required     | false
type         | Array
example      | {{< highlight shell >}}"handlers": ["pagerduty", "email"]{{< /highlight >}}

low_flap_threshold | 
-------------------|------
description        | The flap detection low threshold (% state change) for the check. Sensu uses the same [flap detection algorithm as Nagios][31].
required           | false
type               | Integer
example            | {{< highlight shell >}}"low_flap_threshold": 20{{< /highlight >}}

high_flap_threshold | 
--------------------|------
description         | The flap detection high threshold (% state change) for the check. Sensu uses the same [flap detection algorithm as Nagios][31].
required            | true (if `low_flap_threshold` is configured)
type                | Integer
example             | {{< highlight shell >}}"high_flap_threshold": 60{{< /highlight >}}

source       | 
-------------|------
description  | The check source, used to create a [proxy client][32] for an external resource (e.g. a network switch).
required     | false
type         | String
validated    | `/^[\w\.-]+$/`
example      | {{< highlight shell >}}"source": "switch-dc-01"{{< /highlight >}}

truncate_output | 
----------------|------
description     | If check output will be truncated for storage. Output truncation is automatically set to `true` for checks with type set to `"metric"` unless configured to `false`.
required        | false
type            | Boolean
default         | false
example         | {{< highlight shell >}}"truncate_output": true{{< /highlight >}}

truncate_output_length | 
-----------------------|------
description            |The output truncation length, the maximum number of characters.
required               | false
type                   | Integer
default                | 255
example                | {{< highlight shell >}}"truncate_output_length": 1024{{< /highlight >}}

aggregate    | 
-------------|------
description  | Create a named aggregate for the check. Check result data will be aggregated and exposed via the [Sensu Aggregates API][33]. _NOTE: named aggregates are new to [Sensu version 0.24][43], now being defined with a String data type rather than a Boolean (i.e. `true` or `false`). Legacy check definitions with `"aggregate": true` attributes will default to using the check name as the aggregate name._
required     | false
type         | String
example      | {{< highlight shell >}}"aggregate": "proxy_servers"{{< /highlight >}}

aggregates   | 
-------------|------
description  | An array of strings defining one or more named aggregates (described above).
required     | false
type         | Array
example      | {{< highlight shell >}}"aggregates": [ "webservers", "production" ]{{< /highlight >}}

subdue       | 
-------------|------
description  | The [`subdue` definition scope][42], used to determine when a check is subdued.
required     | false
type         | Hash
example      | {{< highlight shell >}}"subdue": {}{{< /highlight >}}

hooks        | 
-------------|------
description  | The `hooks` definition scope, commands run by the Sensu client in response to the result of the check command execution
required     | false
type         |  Hash
example      | {{< highlight shell >}}"hooks": {}{{< /highlight >}}

contact      | 
-------------|------
description  | A contact name to use for the check. <br>**ENTERPRISE: This configuration is provided for using [Contact Routing][44].**
required     | false
type         | String
example      | {{< highlight shell >}}"contact": "ops"{{< /highlight >}}

contacts     | 
-------------|------
description  | An array of contact names to use for the check. Each array item (name) must be a string. <br>**ENTERPRISE: This configuration is provided for using [Contact Routing][44].**
required     | false
type         | Array
example      | {{< highlight shell >}}"contacts": ["ops"]{{< /highlight >}}

proxy_requests | 
---------------|------
description    | The [`proxy_requests` definition scope][48], used to create proxy check requests.
required       | false
type           | Hash
example        | {{< highlight shell >}}"proxy_requests": {}{{< /highlight >}}

occurrences  | 
-------------|------
description  | The number of event occurrences that must occur before an event is handled for the check. _NOTE: For this attribute to take effect, the `occurrences` filter must be explicitly configured in your handler definition._
required     | false
type         | Integer
default      | `1`
example      | {{< highlight shell >}}"occurrences": 3{{< /highlight >}}

refresh      | 
-------------|------
description  | Time in seconds until the event occurrence count is considered reset for the purpose of counting `occurrences`, to allow an event for the check to be handled again. For example, a check with a refresh of `1800` will have its events (recurrences) handled every 30 minutes, to remind users of the issue. _NOTE: For this attribute to take effect, the `occurrences` filter must be explicitly configured in your handler definition._
required     | false
type         | Integer
default      | `1800`
example      | {{< highlight shell >}}"refresh": 3600{{< /highlight >}}

dependencies | 
-------------|------
description  | An array of check dependencies. Events for the check will not be handled if events exist for one or more of the check dependencies. A check dependency can be a check executed by the same Sensu client (eg. `check_app`), or a client/check pair (eg.`db-01/check_mysql`). _NOTE: For this attribute to take effect, the `check_dependencies` filter must be explicitly configured in your handler definition._
required     | false
type         | Array
example      | {{< highlight shell >}}"dependencies": [
  "check_app",
  "db-01/check_mysql"
]
{{< /highlight >}}

notification | 
-------------|------
description  | The notification message used for events created by the check, instead of the commonly used check output. This attribute is used by most notification event handlers that use the sensu-plugin library.
required     | false
type         | String
example      | {{< highlight shell >}}"notification": "the shopping cart application is not responding to requests"{{< /highlight >}}

influxdb     | 
-------------|------
description  | The [`influxdb` definition scope][61], used to configure the [Sensu Enterprise InfluxDB integration][62] ([Sensu Enterprise][60] users only).
required     | false
type         | Hash
example      | {{< highlight shell >}}"influxdb": {}{{< /highlight >}}

opsgenie     | 
-------------|------
description  | The [`opsgenie` definition scope][63], used to configure the [Sensu Enterprise OpsGenie integration][64] ([Sensu Enterprise][60] users only).
required     | false
type         | Hash
example      | {{< highlight shell >}}"opsgenie": {}{{< /highlight >}}

#### `subdue` attributes

The following attributes are configured within the `{"checks": { "CHECK": {
"subdue": {} } } }` [configuration scope][29] (where `CHECK` is a valid [check
name][41]).

##### EXAMPLE {#subdue-attributes-example}

{{< highlight json >}}
{
  "checks": {
    "check-printer": {
      "...": "...",
      "subdue": {
        "days": {
          "all": [
            {
              "begin": "5:00 PM",
              "end": "8:00 AM"
            }
          ]
        }
      }
    }
  }
}
{{< /highlight >}}

##### ATTRIBUTES {#subdue-attributes-specification}

days         | 
-------------|------
description  | A hash of days of the week or 'all', each day specified must define one or more time windows in which the check is not scheduled to be executed.
required     | false (unless `subdue` is configured)
type         | Hash
example      | {{< highlight shell >}}"days": {
  "all": [
    {
      "begin": "5:00 PM",
      "end": "8:00 AM"
    }
  ],
  "friday": [
    {
      "begin": "12:00 PM",
      "end": "5:00 PM"
    }
  ]
}
{{< /highlight >}}

#### `influxdb` attributes

The following attributes are configured within the `{ "check": { "influxdb": {} }
}` [configuration scope][29].

**ENTERPRISE: This configuration is provided for using the built-in [Sensu
Enterprise InfluxDB integration][62].**

##### EXAMPLE {#influxdb-attributes-example}

{{< highlight json >}}
{
  "check": {
    "name": "nginx_process",
    "...": "...",
    "influxdb": {
      "tags": {
        "dc": "us-central-1"
      }
    }
  }
}
{{< /highlight >}}

##### ATTRIBUTES {#influxdb-attributes-specification}

tags           | 
---------------|------
description    | Custom tags (key/value pairs) to add to every InfluxDB measurement. Check tags will override any [InfluxDB integration tags][62] with the same key.
required       | false
type           | Hash
default        | {{< highlight shell >}}{}{{< /highlight >}}
example        | {{< highlight shell >}}
"tags": {
  "dc": "us-central-1"
}
{{< /highlight >}}

#### `opsgenie` attributes

The following attributes are configured within the `{ "check": { "opsgenie": {} }
}` [configuration scope][29].

**ENTERPRISE: This configuration is provided for using the built-in [Sensu
Enterprise OpsGenie integration][64].**

##### EXAMPLE {#opsgenie-attributes-example}

{{< highlight json >}}
{
  "check": {
    "name": "nginx_process",
    "...": "...",
    "opsgenie": {
      "tags": ["production"]
    }
  }
}
{{< /highlight >}}

##### ATTRIBUTES {#opsgenie-attributes-specification}

tags         | 
-------------|------
description  | An array of OpsGenie alert tags that will be added to created alerts.
required     | false
type         | Array
default      | `[]`
example      | {{< highlight shell >}}"tags": ["production"]{{< /highlight >}}

#### `hooks` attributes {#hooks-attributes}

The following attributes are configured within the `{"checks": { "CHECK": { "hooks": {} } } }` [configuration scope][29] (where `CHECK` is a valid [check name][41]).

##### ATTRIBUTES {#hooks-attributes-specification}

command      | 
-------------|------
description  | The hook command to be executed.
required     | true
type         | String
example      | {{< highlight shell >}}"command": "ps aux"{{< /highlight >}}

timeout      | 
-------------|------
description  | The hook command execution duration timeout in seconds (hard stop).
required     | false
type         | Integer
default      | 60
example      | {{< highlight shell >}}"timeout": 30{{< /highlight >}}

stdin        | 
-------------|------
description  | If the Sensu client writes JSON serialized Sensu client and check data to the hook command process' STDIN. The hook command must expect the JSON data via STDIN, read it and close STDIN.
required     | false
type         | Boolean
default      | false
example      | {{< highlight shell >}}"stdin": true{{< /highlight >}}

##### Hook naming {#hook-names}

Each check hook has a unique hook name. Valid hook names include (in order of precedence): “1”-“255”, “ok”, “warning”, “critical”, “unknown”, and “non-zero”. The check hook name is used to determine the appropriate hook command to execute, depending on the check execution status (e.g. 1).


##### `HOOK` attributes {#hook-attributes}

The following attributes are configured within the {"checks": { "CHECK": { "hooks": { "HOOK": {}} } } } [configuration scope][29] (where CHECK is a valid [check name][41] and HOOK is a valid [hook name](#hook-names)).


##### EXAMPLE {#hooks-attributes-example}

{{< highlight json >}}
{
  "checks": {
    "nginx_process": {
      "command": "check-process.rb -p nginx",
      "subscribers": [
        "proxy"
      ],
      "interval": 60,
      "hooks": {
        "non-zero": {
          "command": "ps aux",
          "timeout": 10
        }
      }
    }
  }
}
{{< /highlight >}}

#### `proxy_requests` attributes {#proxy-requests-attributes}

The following attributes are configured within the `{"checks": {
"CHECK": { "proxy_requests": {} } } }` [configuration scope][29]
(where `CHECK` is a valid [check name][41]).

##### EXAMPLE {#proxy-requests-attributes-example}

Publish a check request to the configured `subscribers` (e.g.
`["roundrobin:snmp_pollers"]`) for every Sensu client in the registry
that matches the configured client attributes in `client_attributes`
on the configured `interval` (e.g. `60`). Client tokens in the check
definition (e.g. `"check-snmp-if.rb -h :::address::: -i eth0"`) are
substituted prior to publishing the check request. The check request
check `source` is set to the client `name`.

{{< highlight json >}}
{
  "checks": {
    "check_arista_eth0": {
      "command": "check-snmp-if.rb -h :::address::: -i eth0",
      "subscribers": [
        "roundrobin:snmp_pollers"
      ],
      "interval": 60,
      "proxy_requests": {
        "client_attributes": {
          "keepalives": false,
          "device_type": "router",
          "device_manufacturer": "arista"
        }
      }
    }
  }
}
{{< /highlight >}}

##### ATTRIBUTES {#proxy-requests-attributes-specification}

client_attributes | 
------------------|------
description       | Sensu client attributes to match clients in the registry.
required          | true
type              | Hash
example           | {{< highlight shell >}}"client_attributes": {
  "keepalive": false,
  "device_type": "router",
  "device_manufacturer": "arista",
  "subscriptions": "eval: value.include?('dc-01')"
}
{{< /highlight >}}

splay       | 
------------|------
description | If proxy check requests should be splayed, published evenly over a window of time, determined by the check interval and a configurable splay coverage percentage. For example, if a check has an interval of 60s and a configured splay coverage of 90%, its proxy check requests would be splayed evenly over a time window of 60s * 90%, 54s, leaving 6s for the last proxy check execution before the the next round of proxy check requests for the same check.
required    | false
type        | Boolean
default     | false
example     | {{< highlight shell >}}"splay": true{{< /highlight >}}

splay_coverage | 
---------------|------
description    | If proxy check requests should be splayed, published evenly over a window of time, determined by the check interval and a configurable splay coverage percentage. For example, if a check has an interval of 60s and a configured splay coverage of 90%, its proxy check requests would be splayed evenly over a time window of 60s * 90%, 54s, leaving 6s for the last proxy check execution before the the next round of proxy check requests for the same check.
required       | false
type           | Integer
default        | 90
example        | {{< highlight shell >}}"splay_coverage": 65{{< /highlight >}}

You can, as above, also use `eval` to perform more complicated filtering with Ruby on the available `value`, such as finding clients with particular subscriptions.

For a more general introduction to using this style of check, check out the [guide on using proxy checks][49] or the [guide on adding a proxy client][50].

#### Custom attributes

Because Sensu configuration is simply JSON data, it is possible to define
configuration attributes that are not part of the Sensu [check definition
specification][17]. These custom check attributes are included in
[check results][4] and [event data][40], providing additional context
about the check. Some great example use cases for custom check
definition attributes are links to playbook documentation
(i.e. "here's a link to some instructions for how to fix this if it's
broken"), [contact routing][36], and metric graph image URLs.

##### EXAMPLE

The following is an example Sensu check definition that a custom definition
attribute, `"playbook"`, a URL for documentation to aid in the resolution of
events for the check. The playbook URL will be available in [event data][34] and
thus able to be included in event notifications (e.g. email).

{{< highlight json >}}
{
  "checks": {
    "check_mysql_replication": {
      "command": "check-mysql-replication-status.rb --user sensu --password secret",
      "subscribers": [
        "mysql"
      ],
      "interval": 30,
      "playbook": "http://docs.example.com/wiki/mysql-replication-playbook"
    }
  }
}
{{< /highlight >}}

### Check result specification

See [check results][4] (above) for more information about check results,
including an [example check result][37].

Required attributes below are the minimum for check results submitted
via the [client socket input][46]. Additional attributes are
automatically added by the client to build a complete check result.

#### `check` attributes {#check-result-check-attributes}

status       | 
-------------|------
description  | The check execution exit status code. An exit status code of `0` (zero) indicates `OK`, `1` indicates `WARNING`, and `2` indicates `CRITICAL`; exit status codes other than `0`, `1`, or `2` indicate an `UNKNOWN` or custom status.
required     | true
type         | Integer
example      | {{< highlight shell >}}"status": 0{{< /highlight >}}

command      | 
-------------|------
description  | The command as [provided by the check definition][17].
required     | false
type         | String
example      | {{< highlight shell >}}"command": "check-http.rb -u https://sensuapp.org"{{< /highlight >}}

subscribers  | 
-------------|------
description  | The check subscribers as [provided by the check definition][17].
required     | false
type         | Array
example      | {{< highlight shell >}}"subscribers": ["database_servers"]{{< /highlight >}}

interval     | 
-------------|------
description  | The check interval in seconds, as [provided by the check definition][17]
required     | false
type         | Integer
example      | {{< highlight shell >}}"interval": 60{{< /highlight >}}

name         | 
-------------|------
description  | The check name, as [provided by the check definition][17]
required     | true
type         | String
example      | {{< highlight shell >}}"name": "sensu-website"{{< /highlight >}}

issued       | 
-------------|------
description  | The time the check request was issued (by the [Sensu server][6] or [client][1]), stored as an integer (i.e. `Time.now.to_i`)
required     | false
type         | Integer
example      | {{< highlight shell >}}"issued": 1458934742{{< /highlight >}}

executed     | 
-------------|------
description  | The time the check request was executed by the [Sensu client][1], stored as and integer (i.e. `Time.now.to_i`).
required     | false
type         | Integer
example      | {{< highlight shell >}}"executed": 1458934742{{< /highlight >}}

duration     | 
-------------|------
description  | The amount of time (in seconds) it took for the [Sensu client][1] to execute the check.
required     | false
type         | Float
example      | {{< highlight shell >}}"duration": 0.637{{< /highlight >}}

output       | 
-------------|------
description  | The output produced by the check `command`.
required     | true
type         | String
example      | {{< highlight shell >}}"output": "CheckHttp OK: 200, 78572 bytes\n"{{< /highlight >}}


#### `client` attributes {#check-result-client-attributes}

name         | 
-------------|------
description  | The name of the [Sensu client][1] that generated the check result. The Sensu server will use the client `name` value to fetch the corresponding client attributes from the [Clients API][39] and add them to the resulting [Sensu event][34] for context.
type         | String
example      | {{< highlight shell >}}"client": { "name": "i-424242" }{{< /highlight >}}


[?]:  #
[1]:  ../clients/
[2]:  https://en.wikipedia.org/wiki/Standard_streams
[3]:  https://www.nagios.org/
[4]:  #check-results
[5]:  ../../overview/architecture/#event-processor
[6]:  ../server/
[7]:  #check-configuration
[8]:  ../plugins/#check-plugins
[9]:  https://en.wikipedia.org/wiki/PATH_(variable)
[10]: ../clients/#standalone-check-execution-scheduler
[11]: https://en.wikipedia.org/wiki/Publish%E2%80%93subscribe_pattern
[12]: #standalone-checks
[13]: ../transport/
[14]: https://en.wikipedia.org/wiki/Publish%E2%80%93subscribe_pattern#Message_filtering
[15]: ../clients/#client-subscriptions
[16]: ../clients/#client-definition-specification
[17]: #check-definition-specification
[18]: http://www.ntp.org/
[19]: #subscription-checks
[20]: http://www.json.org/
[21]: #custom-attributes
[22]: #check-commands
[23]: #check-command-arguments
[24]: ../clients#custom-attributes
[25]: #check-token-default-values
[26]: ../clients#client-attributes
[27]: ../plugins
[28]: https://github.com/sensu-plugins/sensu-plugins-http
[29]: ../configuration/#configuration-scopes
[30]: http://ruby-doc.org/core-2.2.0/Regexp.html
[31]: https://assets.nagios.com/downloads/nagioscore/docs/nagioscore/3/en/flapping.html
[32]: ../clients#proxy-clients
[33]: ../../api/aggregates/
[34]: ../events/
[35]: ../handlers/
[36]: /sensu-enterprise/latest/contact-routing
[37]: #example-check-result-output
[38]: #check-result-specification
[39]: ../../api/clients/
[40]: ../events#event-data
[41]: #check-names
[42]: #subdue-attributes
[43]: ../../changelog/
[44]: /sensu-enterprise/latest/contact-routing
[45]: ../../api/events#the-resolve-api-endpoint
[46]: ../clients#client-socket-input
[47]: https://en.wikipedia.org/wiki/Cron#CRON_expression
[48]: #proxy-requests-attributes
[49]: ../../guides/intro-to-checks/#proxy-clients
[50]: ../../guides/adding-a-client/#proxy-clients
[60]: /sensu-enterprise/latest
[61]: #influxdb-attributes
[62]: /sensu-enterprise/latest/integrations/influxdb
[63]: #opsgenie-attributes
[64]: /sensu-enterprise/latest/integrations/opsgenie
[65]: ../configuration#configuration-scopes
[66]: ../../api/checks#checkscheck-delete
