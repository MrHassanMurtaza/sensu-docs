---
title: "Authentication"
description: "In addition to built-in RBAC, Sensu includes license-activated support for authentication using a Lightweight Directory Access Protocol (LDAP) provider. Read the guide to configure a provider."
weight: 4
version: "5.14"
product: "Sensu Go"
menu:
  sensu-go-5.14:
    parent: installation
---

- [Managing authentication providers](#managing-authentication-providers)
- [Configuring authentication providers](#configuring-authentication-providers)
- [LDAP authentication](#ldap-authentication)
  - [Examples](#ldap-configuration-examples)
  - [LDAP specification](#ldap-specification)
  - [Troubleshooting](#ldap-troubleshooting)
- [Active Directory authentication](#active-directory-authentication)
  - [Examples](#active-directory-configuration-examples)
  - [Active Directory specification](#active-directory-specification)
  - [Troubleshooting](#active-directory-troubleshooting)
- [OIDC authentication](#oidc-authentication)
  - [OIDC configuration examples](#oidc-configuration-examples)
  - [OIDC specification](#oidc-specification)
  - [Okta](#okta)

Sensu requires username and password authentication to access the [Sensu dashboard][1], [API][8], and command line tool ([sensuctl][2]).
For Sensu's [default user credentials][3] and more information about configuring Sensu role based access control, see the [RBAC reference][4] and [guide to creating users][5].

In addition to built-in RBAC, Sensu includes [license-activated][6] support for authentication using external authentication providers.
Sensu currently supports Microsoft Active Directory and standards-compliant Lightweight Directory Access Protocol tools like OpenLDAP.

**LICENSED TIER**: Unlock authentication providers in Sensu Go with a Sensu license. To activate your license, see the [getting started guide][6].

## Managing authentication providers

You can view and delete authentication providers using sensuctl and the [authentication providers API](../../api/authproviders).
To set up an authentication provider for Sensu, see the section on [configuring authentication providers](#configuring-authentication-providers).

To view active authentication providers:

{{< highlight shell >}}
sensuctl auth list
{{< /highlight >}}

To view configuration details for an authentication provider named `openldap`:

{{< highlight shell >}}
sensuctl auth info openldap
{{< /highlight >}}

To delete an authentication provider named `openldap`:

{{< highlight shell >}}
sensuctl auth delete openldap
{{< /highlight >}}

## Configuring authentication providers

**1. Write an authentication provider configuration definition**

Write an authentication provider configuration definition.

For standards-compliant Lightweight Directory Access Protocol tools like OpenLDAP, see the [LDAP configuration examples](#ldap-configuration-examples) and [specification](#ldap-specification).
For Microsoft Active Directory, see the [AD configuration examples](#active-directory-configuration-examples) and [specification](#active-directory-authentication).

**2. Apply the configuration using sensuctl**

Log in to sensuctl as the [default admin user][3] and apply the configuration to Sensu.

{{< highlight shell >}}
sensuctl create --file filename.json
{{< /highlight >}}

You can verify that your provider configuration has been applied successfully using sensuctl.

{{< highlight shell >}}
sensuctl auth list

 Type     Name    
────── ────────── 
 ldap   openldap  
{{< /highlight >}}

**3. Integrate with Sensu RBAC**

Now that you've configured an authentication provider, you'll need to configure Sensu RBAC to give those users permissions within Sensu.
Sensu RBAC allows management and access of users and resources based on namespaces, groups, roles, and bindings.
See the [RBAC reference][4] for more information about configuring permissions in Sensu and [implementation examples](../../reference/rbac/#role-and-role-binding-examples).

- **Namespaces** partition resources within Sensu. Sensu entities, checks, handlers, and other [namespaced resources][17] belong to a single namespace.
- **Roles** create sets of permissions (get, delete, etc.) tied to resource types. **Cluster roles** apply permissions across namespaces and include access to [cluster-wide resources][18] like users and namespaces. 
- **Role bindings** assign a role to a set of users and groups within a namespace; **cluster role bindings** assign a cluster role to a set of users and groups cluster-wide.

To enable permissions for external users and groups within Sensu, create a set of [roles][10], [cluster roles][11], [role bindings][12], and [cluster role bindings][13] that map to the usernames and group names found in your authentication providers.
Make sure to include the [group prefix](#groups-prefix) and [username prefix](#username-prefix) when creating Sensu role bindings and cluster role bindings.
Without an assigned role or cluster role, users can sign in to the Sensu dashboard but can't access any Sensu resources.

**4. Log in to Sensu**

Once you've configured the correct roles and bindings, log in to [sensuctl](../../sensuctl/reference#first-time-setup) and the [Sensu dashboard](../../dashboard/overview) using your single-sign-on username and password (no prefix required).


## LDAP authentication

Sensu offers license-activated support for using a standards-compliant Lightweight Directory Access Protocol tool for authentication to the Sensu dashboard, API, and sensuctl.
The Sensu LDAP authentication provider is tested with [OpenLDAP][7].
Active Directory users should head over to the [Active Directory section](#active-directory-authentication).

### LDAP configuration examples

**Example LDAP configuration: Minimum required attributes**

{{< language-toggle >}}

{{< highlight yml >}}
type: ldap
api_version: authentication/v2
metadata:
  name: openldap
spec:
  servers:
  - group_search:
      base_dn: dc=acme,dc=org
    host: 127.0.0.1
    user_search:
      base_dn: dc=acme,dc=org
{{< /highlight >}}

{{< highlight json >}}
{
  "type": "ldap",
  "api_version": "authentication/v2",
  "spec": {
    "servers": [
      {
        "host": "127.0.0.1",
        "group_search": {
          "base_dn": "dc=acme,dc=org"
        },
        "user_search": {
          "base_dn": "dc=acme,dc=org"
        }
      }
    ]
  },
  "metadata": {
    "name": "openldap"
  }
}
{{< /highlight >}}

{{< /language-toggle >}}

**Example LDAP configuration: All attributes**

{{< language-toggle >}}

{{< highlight yml >}}
type: ldap
api_version: authentication/v2
metadata:
  name: openldap
spec:
  groups_prefix: ldap
  servers:
  - binding:
      password: P@ssw0rd!
      user_dn: cn=binder,dc=acme,dc=org
    client_cert_file: /path/to/ssl/cert.pem
    client_key_file: /path/to/ssl/key.pem
    group_search:
      attribute: member
      base_dn: dc=acme,dc=org
      name_attribute: cn
      object_class: groupOfNames
    host: 127.0.0.1
    insecure: false
    port: 636
    security: tls
    trusted_ca_file: /path/to/trusted-certificate-authorities.pem
    user_search:
      attribute: uid
      base_dn: dc=acme,dc=org
      name_attribute: cn
      object_class: person
  username_prefix: ldap
{{< /highlight >}}

{{< highlight json >}}
{
  "type": "ldap",
  "api_version": "authentication/v2",
  "spec": {
    "servers": [
      {
        "host": "127.0.0.1",
        "port": 636,
        "insecure": false,
        "security": "tls",
        "trusted_ca_file": "/path/to/trusted-certificate-authorities.pem",
        "client_cert_file": "/path/to/ssl/cert.pem",
        "client_key_file": "/path/to/ssl/key.pem",
        "binding": {
          "user_dn": "cn=binder,dc=acme,dc=org",
          "password": "P@ssw0rd!"
        },
        "group_search": {
          "base_dn": "dc=acme,dc=org",
          "attribute": "member",
          "name_attribute": "cn",
          "object_class": "groupOfNames"
        },
        "user_search": {
          "base_dn": "dc=acme,dc=org",
          "attribute": "uid",
          "name_attribute": "cn",
          "object_class": "person"
        }
      }
    ],
    "groups_prefix": "ldap",
    "username_prefix": "ldap"
  },
  "metadata": {
    "name": "openldap"
  }
}
{{< /highlight >}}

{{< /language-toggle >}}

## LDAP specification

### Top-level attributes

type         | 
-------------|------
description  | Top-level attribute specifying the [`sensuctl create`][sc] resource type. LDAP definitions should always be of type `ldap`.
required     | true
type         | String
example      | {{< highlight shell >}}"type": "ldap"{{< /highlight >}}

api_version  | 
-------------|------
description  | Top-level attribute specifying the Sensu API group and version. For LDAP definitions, this attribute should always be `authentication/v2`.
required     | true
type         | String
example      | {{< highlight shell >}}"api_version": "authentication/v2"{{< /highlight >}}

metadata     | 
-------------|------
description  | Top-level map containing the LDAP definition `name`. See the [metadata attributes reference][24] for details.
required     | true
type         | Map of key-value pairs
example      | {{< highlight shell >}}
"metadata": {
  "name": "openldap"
}
{{< /highlight >}}

spec         | 
-------------|------
description  | Top-level map that includes the LDAP [spec attributes][sp].
required     | true
type         | Map of key-value pairs
example      | {{< highlight shell >}}
"spec": {
  "servers": [
    {
      "host": "127.0.0.1",
      "port": 636,
      "insecure": false,
      "security": "tls",
      "trusted_ca_file": "/path/to/trusted-certificate-authorities.pem",
      "client_cert_file": "/path/to/ssl/cert.pem",
      "client_key_file": "/path/to/ssl/key.pem",
      "binding": {
        "user_dn": "cn=binder,dc=acme,dc=org",
        "password": "P@ssw0rd!"
      },
      "group_search": {
        "base_dn": "dc=acme,dc=org",
        "attribute": "member",
        "name_attribute": "cn",
        "object_class": "groupOfNames"
      },
      "user_search": {
        "base_dn": "dc=acme,dc=org",
        "attribute": "uid",
        "name_attribute": "cn",
        "object_class": "person"
      }
    }
  ],
  "groups_prefix": "ldap",
  "username_prefix": "ldap"
}
{{< /highlight >}}

### Spec attributes

| servers    |      |
-------------|------
description  | An array of [LDAP servers](#server-attributes) for your directory. During the authentication process, Sensu attempts to authenticate using each LDAP server in sequence.
required     | true
type         | Array
example      | {{< highlight shell >}}
"servers": [
  {
    "host": "127.0.0.1",
    "port": 636,
    "insecure": false,
    "security": "tls",
    "trusted_ca_file": "/path/to/trusted-certificate-authorities.pem",
    "client_cert_file": "/path/to/ssl/cert.pem",
    "client_key_file": "/path/to/ssl/key.pem",
    "binding": {
      "user_dn": "cn=binder,dc=acme,dc=org",
      "password": "P@ssw0rd!"
    },
    "group_search": {
      "base_dn": "dc=acme,dc=org",
      "attribute": "member",
      "name_attribute": "cn",
      "object_class": "groupOfNames"
    },
    "user_search": {
      "base_dn": "dc=acme,dc=org",
      "attribute": "uid",
      "name_attribute": "cn",
      "object_class": "person"
    }
  }
]
{{< /highlight >}}

<a name="groups-prefix"></a>

| groups_prefix |   |
-------------|------
description  | The prefix added to all LDAP groups. Sensu prepends prefixes with a colon. For example, for the groups prefix `ldap` and the group `dev`, the resulting group name in Sensu is `ldap:dev`. Use this prefix when integrating LDAP groups with Sensu RBAC [role bindings][12] and [cluster role bindings][13].
required     | false
type         | String
example      | {{< highlight shell >}}"groups_prefix": "ldap"{{< /highlight >}}

<a name="username-prefix"></a>

| username_prefix | |
-------------|------
description  | The prefix added to all LDAP usernames.  Sensu prepends prefixes with a colon. For example, for the username prefix `ldap` and the user `alice`, the resulting username in Sensu is `ldap:alice`. Use this prefix when integrating LDAP users with Sensu RBAC [role bindings][12] and [cluster role bindings][13]. Users _do not_ need to provide this prefix when logging in to Sensu.
required     | false
type         | String
example      | {{< highlight shell >}}"username_prefix": "ldap"{{< /highlight >}}

### Server attributes

| host       |      |
-------------|------
description  | LDAP server IP address or [FQDN](https://en.wikipedia.org/wiki/Fully_qualified_domain_name)
required     | true
type         | String
example      | {{< highlight shell >}}"host": "127.0.0.1"{{< /highlight >}}

| port       |      |
-------------|------
description  | LDAP server port
required     | true
type         | Integer
default      | `389` for insecure connections, `636` for TLS connections
example      | {{< highlight shell >}}"port": 636{{< /highlight >}}

| insecure   |      |
-------------|------
description  | Skips SSL certificate verification when set to `true`. _WARNING: Do not use an insecure connection in production environments._
required     | false
type         | Boolean
default      | `false`
example      | {{< highlight shell >}}"insecure": false{{< /highlight >}}

| security   |      |
-------------|------
description  | Determines the encryption type to be used for the connection to the LDAP server: `insecure` (unencrypted connection, not recommended for production), `tls` (secure encrypted connection), or `starttls` (unencrypted connection upgrades to a secure connection).
type         | String
default      | `"tls"`
example      | {{< highlight shell >}}"security": "tls"{{< /highlight >}}

| trusted_ca_file | |
-------------|------
description  | Path to an alternative CA bundle file in PEM format to be used instead of the system's default bundle. This CA bundle is used to verify the server's certificate.
required     | false
type         | String
example      | {{< highlight shell >}}"trusted_ca_file": "/path/to/trusted-certificate-authorities.pem"{{< /highlight >}}

| client_cert_file | |
-------------|------
description  | Path to the certificate that should be sent to the server if it requests it
required     | false
type         | String
example      | {{< highlight shell >}}"client_cert_file": "/path/to/ssl/cert.pem"{{< /highlight >}}

| client_key_file | |
-------------|------
description  | Path to the key file associated with the `client_cert_file`
required     | false
type         | String
example      | {{< highlight shell >}}"client_key_file": "/path/to/ssl/key.pem"{{< /highlight >}}

| binding    |      |
-------------|------
description  | The LDAP account that performs user and group lookups. If your sever supports anonymous binding, you can omit the `user_dn` or `password` attributes to query the directory without credentials.
required     | false
type         | Map
example      | {{< highlight shell >}}
"binding": {
  "user_dn": "cn=binder,dc=acme,dc=org",
  "password": "P@ssw0rd!"
}
{{< /highlight >}}

| group_search |    |
-------------|------
description  | Search configuration for groups. See the [group search attributes][21] for more information.
required     | true
type         | Map
example      | {{< highlight shell >}}
"group_search": {
  "base_dn": "dc=acme,dc=org",
  "attribute": "member",
  "name_attribute": "cn",
  "object_class": "groupOfNames"
}
{{< /highlight >}}

| user_search |     |
-------------|------
description  | Search configuration for users. See the [user search attributes][22] for more information.
required     | true
type         | Map
example      | {{< highlight shell >}}
"user_search": {
  "base_dn": "dc=acme,dc=org",
  "attribute": "uid",
  "name_attribute": "cn",
  "object_class": "person"
}
{{< /highlight >}}

### Binding attributes

| user_dn    |      |
-------------|------
description  | The LDAP account that performs user and group lookups. We recommend using a read-only account. Use the distinguished name (DN) format, such as `cn=binder,cn=users,dc=domain,dc=tld`. If your sever supports anonymous binding, you can omit this attribute to query the directory without credentials.
required     | false
type         | String
example      | {{< highlight shell >}}"user_dn": "cn=binder,dc=acme,dc=org"{{< /highlight >}}

| password   |      |
-------------|------
description  | Password for the `user_dn` account. If your sever supports anonymous binding, you can omit this attribute to query the directory without credentials.
required     | false
type         | String
example      | {{< highlight shell >}}"password": "P@ssw0rd!"{{< /highlight >}}

### Group search attributes

| base_dn    |      |
-------------|------
description  | Tells Sensu which part of the directory tree to search. For example, `dc=acme,dc=org` searches within the `acme.org` directory.
required     | true
type         | String
example      | {{< highlight shell >}}"base_dn": "dc=acme,dc=org"{{< /highlight >}}

| attribute  |      |
-------------|------
description  | Used for comparing result entries. This is combined with other filters as <br> `"(<Attribute>=<value>)"`.
required     | false
type         | String
default      | `"member"`
example      | {{< highlight shell >}}"attribute": "member"{{< /highlight >}}

| name_attribute |  |
-------------|------
description  | Represents the attribute to use as the entry name.
required     | false
type         | String
default      | `"cn"`
example      | {{< highlight shell >}}"name_attribute": "cn"{{< /highlight >}}

| object_class |   |
-------------|------
description  | Identifies the class of objects returned in the search result. This is combined with other filters as `"(objectClass=<ObjectClass>)"`.
required     | false
type         | String
default      | `"groupOfNames"`
example      | {{< highlight shell >}}"object_class": "groupOfNames"{{< /highlight >}}

### User search attributes

| base_dn    |      |
-------------|------
description  | Tells Sensu which part of the directory tree to search. For example, `dc=acme,dc=org` searches within the `acme.org` directory.
required     | true
type         | String
example      | {{< highlight shell >}}"base_dn": "dc=acme,dc=org"{{< /highlight >}}

| attribute  |      |
-------------|------
description  | Used for comparing result entries. This is combined with other filters as <br> `"(<Attribute>=<value>)"`.
required     | false
type         | String
default      | `"uid"`
example      | {{< highlight shell >}}"attribute": "uid"{{< /highlight >}}

| name_attribute |  |
-------------|------
description  | Represents the attribute to use as the entry name.
required     | false
type         | String
default      | `"cn"`
example      | {{< highlight shell >}}"name_attribute": "cn"{{< /highlight >}}

| object_class |   |
-------------|------
description  | Identifies the class of objects returned in the search result. This is combined with other filters as `"(objectClass=<ObjectClass>)"`.
required     | false
type         | String
default      | `"person"`
example      | {{< highlight shell >}}"object_class": "person"{{< /highlight >}}

### Metadata attributes

| name       |      |
-------------|------
description  | A unique string used to identify the LDAP configuration. Names cannot contain special characters or spaces (validated with Go regex [`\A[\w\.\-]+\z`](https://regex101.com/r/zo9mQU/2)).
required     | true
type         | String
example      | {{< highlight shell >}}"name": "openldap"{{< /highlight >}}

## LDAP troubleshooting

In order to troubleshoot any issue with LDAP authentication, the first step
should always be to [increase log verbosity][19] of sensu-backend to the debug
log level. Most authentication and authorization errors are only displayed on
the debug log level, in order to avoid flooding the log files.

_NOTE: If you can't locate any log entries referencing LDAP authentication, make
sure the LDAP provider was successfully installed using [sensuctl][20]_

### Authentication errors

Here are some common error messages and possible solutions:

**Error message**: `failed to connect: LDAP Result Code 200 "Network Error"`

The LDAP provider couldn't establish a TCP connection to the LDAP server. Verify
the `host` & `port` attributes. If you are not using LDAP over TLS/SSL , make
sure to set the value of the `security` attribute to `"insecure"` for plaintext
communication.

 **Error message**: `certificate signed by unknown authority`

If you are using a self-signed certificate, make sure to set the `insecure`
attribute to `true`. This will bypass verification of the certificate's signing
authority.

**Error message**: `failed to bind: ...`

The first step for authenticating a user with the LDAP provider is to bind to
the LDAP server using the service account specified in the [`binding`
object](#binding-attributes). Make sure the `user_dn` specifies a valid **DN**,
and its password is the right one.

**Error message**: `user <username> was not found`

The user search failed, no user account could be found with the given username.
Go look at the [`user_search` object][22] and make sure that:

- The specified `base_dn` contains the requested user entry DN
- The specified `attribute` contains the _username_ as its value in the user entry
- The `object_class` attribute corresponds to the user entry object class

**Error message**: `ldap search for user <username> returned x results, expected only 1`

The user search returned more than one user entry, therefore the provider could
not determine which of these entries should be used. The [`user_search`
object][22] needs to be tweaked so the provided *username* can be used to
uniquely identify a user entry. Here's few possible way of doing it:

- Adjust the `attribute` so its value (which corresponds to the *username*) is
  unique amongst the user entries
- Adjust the `base_dn` so it only includes one of the user entries

**Error message**: `ldap entry <DN> missing required attribute <name_attribute>`

The user entry returned (identified by `<DN>`) doesn't include the attribute
specified by [`name_attribute` object][22]. Therefore the LDAP provider could
not determine which attribute to use as the username in the user entry. The
`name_attribute` should be adjusted so it specifies a human friendly name for
the user. 

**Error message**: `ldap group entry <DN> missing <name_attribute> and cn attributes`

The group search returned a group entry (identified by `<DN>`) that doesn't have
the [`name_attribute` attribute][21] nor a `cn` attribute. Therefore the LDAP
provider could not determine which attribute to use as the group name in the
group entry. The `name_attribute` should be adjusted so it specifies a human
friendly name for the group.

### Authorization issues

Once authenticated, a user needs to be granted permissions via either a
`ClusterRoleBinding` or a `RoleBinding`.

The way in which LDAP users and LDAP groups can be referred as subjects of a
cluster role or role binding depends on the `groups_prefix` and
`username_prefix` configuration attributes values of the [LDAP provider][sp].
For example, for the groups prefix `ldap` and the group `dev`, the resulting
group name in Sensu is `ldap:dev`.

**Issue**: Permissions are not granted via the LDAP group(s)

During authentication, the LDAP provider will print in the logs all groups found
in LDAP, e.g. `found 1 group(s): [dev]`. Keep in mind that this group name does
not contain the `groups_prefix` at this point.

The Sensu backend logs each attempt made to authorize an RBAC request. This is
useful for determining why a specific binding didn't grant the request. For
example:

```
[...] the user is not a subject of the ClusterRoleBinding cluster-admin [...]
[...] could not authorize the request with the ClusterRoleBinding system:user [...]
[...] could not authorize the request with any ClusterRoleBindings [...]
```

## Active Directory authentication

Sensu offers license-activated support for using Microsoft Active Directory (AD) for authentication to the Sensu dashboard, API, and sensuctl. The AD authentication provider is based on the [LDAP authentication provider](#ldap-authentication).

### Active Directory configuration examples

**Example AD configuration: Minimum required attributes**

{{< language-toggle >}}

{{< highlight yml >}}
type: ad
api_version: authentication/v2
metadata:
  name: activedirectory
spec:
  servers:
  - group_search:
      base_dn: dc=acme,dc=org
    host: 127.0.0.1
    user_search:
      base_dn: dc=acme,dc=org
{{< /highlight >}}

{{< highlight json >}}
{
  "type": "ad",
  "api_version": "authentication/v2",
  "spec": {
    "servers": [
      {
        "host": "127.0.0.1",
        "group_search": {
          "base_dn": "dc=acme,dc=org"
        },
        "user_search": {
          "base_dn": "dc=acme,dc=org"
        }
      }
    ]
  },
  "metadata": {
    "name": "activedirectory"
  }
}
{{< /highlight >}}

{{< /language-toggle >}}

**Example AD configuration: All attributes**

{{< language-toggle >}}

{{< highlight yml >}}
type: ad
api_version: authentication/v2
metadata:
  name: activedirectory
spec:
  groups_prefix: ad
  servers:
  - binding:
      password: P@ssw0rd!
      user_dn: cn=binder,cn=users,dc=acme,dc=org
    client_cert_file: /path/to/ssl/cert.pem
    client_key_file: /path/to/ssl/key.pem
    default_upn_domain: example.org
    include_nested_groups: true
    group_search:
      attribute: member
      base_dn: dc=acme,dc=org
      name_attribute: cn
      object_class: group
    host: 127.0.0.1
    insecure: false
    port: 636
    security: tls
    trusted_ca_file: /path/to/trusted-certificate-authorities.pem
    user_search:
      attribute: sAMAccountName
      base_dn: dc=acme,dc=org
      name_attribute: displayName
      object_class: person
  username_prefix: ad
{{< /highlight >}}

{{< highlight json >}}
{
  "type": "ad",
  "api_version": "authentication/v2",
  "spec": {
    "servers": [
      {
        "host": "127.0.0.1",
        "port": 636,
        "insecure": false,
        "security": "tls",
        "trusted_ca_file": "/path/to/trusted-certificate-authorities.pem",
        "client_cert_file": "/path/to/ssl/cert.pem",
        "client_key_file": "/path/to/ssl/key.pem",
        "default_upn_domain": "example.org",
        "include_nested_groups": true,
        "binding": {
          "user_dn": "cn=binder,cn=users,dc=acme,dc=org",
          "password": "P@ssw0rd!"
        },
        "group_search": {
          "base_dn": "dc=acme,dc=org",
          "attribute": "member",
          "name_attribute": "cn",
          "object_class": "group"
        },
        "user_search": {
          "base_dn": "dc=acme,dc=org",
          "attribute": "sAMAccountName",
          "name_attribute": "displayName",
          "object_class": "person"
        }
      }
    ],
    "groups_prefix": "ad",
    "username_prefix": "ad"
  },
  "metadata": {
    "name": "activedirectory"
  }
}
{{< /highlight >}}

{{< /language-toggle >}}

## Active Directory specification

### Top-level attributes

type         | 
-------------|------
description  | Top-level attribute specifying the [`sensuctl create`][sc] resource type. AD definitions should always be of type `ad`.
required     | true
type         | String
example      | {{< highlight shell >}}"type": "ad"{{< /highlight >}}

api_version  | 
-------------|------
description  | Top-level attribute specifying the Sensu API group and version. For AD definitions, this attribute should always be `authentication/v2`.
required     | true
type         | String
example      | {{< highlight shell >}}"api_version": "authentication/v2"{{< /highlight >}}

metadata     | 
-------------|------
description  | Top-level map containing the AD definition `name`. See the [metadata attributes reference][23] for details.
required     | true
type         | Map of key-value pairs
example      | {{< highlight shell >}}
"metadata": {
  "name": "activedirectory"
}
{{< /highlight >}}

spec         | 
-------------|------
description  | Top-level map that includes the AD [spec attributes](#active-directory-spec-attributes).
required     | true
type         | Map of key-value pairs
example      | {{< highlight shell >}}
"spec": {
  "servers": [
    {
      "host": "127.0.0.1",
      "port": 636,
      "insecure": false,
      "security": "tls",
      "trusted_ca_file": "/path/to/trusted-certificate-authorities.pem",
      "client_cert_file": "/path/to/ssl/cert.pem",
      "client_key_file": "/path/to/ssl/key.pem",
      "default_upn_domain": "example.org",
      "include_nested_groups": true,
      "binding": {
        "user_dn": "cn=binder,cn=users,dc=acme,dc=org",
        "password": "P@ssw0rd!"
      },
      "group_search": {
        "base_dn": "dc=acme,dc=org",
        "attribute": "member",
        "name_attribute": "cn",
        "object_class": "group"
      },
      "user_search": {
        "base_dn": "dc=acme,dc=org",
        "attribute": "sAMAccountName",
        "name_attribute": "displayName",
        "object_class": "person"
      }
    }
  ],
  "groups_prefix": "ad",
  "username_prefix": "ad"
}
{{< /highlight >}}

### Active Directory spec attributes

| servers    |      |
-------------|------
description  | An array of [AD servers](#active-directory-server-attributes) for your directory. During the authentication process, Sensu attempts to authenticate using each AD server in sequence.
required     | true
type         | Array
example      | {{< highlight shell >}}
"servers": [
  {
    "host": "127.0.0.1",
    "port": 636,
    "insecure": false,
    "security": "tls",
    "trusted_ca_file": "/path/to/trusted-certificate-authorities.pem",
    "client_cert_file": "/path/to/ssl/cert.pem",
    "client_key_file": "/path/to/ssl/key.pem",
    "default_upn_domain": "example.org",
    "include_nested_groups": true,
    "binding": {
      "user_dn": "cn=binder,cn=users,dc=acme,dc=org",
      "password": "P@ssw0rd!"
    },
    "group_search": {
      "base_dn": "dc=acme,dc=org",
      "attribute": "member",
      "name_attribute": "cn",
      "object_class": "group"
    },
    "user_search": {
      "base_dn": "dc=acme,dc=org",
      "attribute": "sAMAccountName",
      "name_attribute": "displayName",
      "object_class": "person"
    }
  }
]
{{< /highlight >}}

<a name="ad-groups-prefix"></a>

| groups_prefix |   |
-------------|------
description  | The prefix added to all AD groups. Sensu prepends prefixes with a colon. For example, for the groups prefix `ad` and the group `dev`, the resulting group name in Sensu is `ad:dev`. Use this prefix when integrating AD groups with Sensu RBAC [role bindings][12] and [cluster role bindings][13].
required     | false
type         | String
example      | {{< highlight shell >}}"groups_prefix": "ad"{{< /highlight >}}

<a name="ad-username-prefix"></a>

| username_prefix | |
-------------|------
description  | The prefix added to all AD usernames.  Sensu prepends prefixes with a colon. For example, for the username prefix `ad` and the user `alice`, the resulting username in Sensu is `ad:alice`. Use this prefix when integrating AD users with Sensu RBAC [role bindings][12] and [cluster role bindings][13]. Users _do not_ need to provide this prefix when logging in to Sensu.
required     | false
type         | String
example      | {{< highlight shell >}}"username_prefix": "ad"{{< /highlight >}}

### Active Directory server attributes

| host       |      |
-------------|------
description  | AD server IP address or [FQDN](https://en.wikipedia.org/wiki/Fully_qualified_domain_name)
required     | true
type         | String
example      | {{< highlight shell >}}"host": "127.0.0.1"{{< /highlight >}}

| port       |      |
-------------|------
description  | AD server port
required     | true
type         | Integer
default      | `389` for insecure connections, `636` for TLS connections
example      | {{< highlight shell >}}"port": 636{{< /highlight >}}

| insecure   |      |
-------------|------
description  | Skips SSL certificate verification when set to `true`. _WARNING: Do not use an insecure connection in production environments._
required     | false
type         | Boolean
default      | `false`
example      | {{< highlight shell >}}"insecure": false{{< /highlight >}}

| security   |      |
-------------|------
description  | Determines the encryption type to be used for the connection to the AD server: `insecure` (unencrypted connection, not recommended for production), `tls` (secure encrypted connection), or `starttls` (unencrypted connection upgrades to a secure connection).
type         | String
default      | `"tls"`
example      | {{< highlight shell >}}"security": "tls"{{< /highlight >}}

| trusted_ca_file | |
-------------|------
description  | Path to an alternative CA bundle file in PEM format to be used instead of the system's default bundle. This CA bundle is used to verify the server's certificate.
required     | false
type         | String
example      | {{< highlight shell >}}"trusted_ca_file": "/path/to/trusted-certificate-authorities.pem"{{< /highlight >}}

| client_cert_file | |
-------------|------
description  | Path to the certificate that should be sent to the server if it requests it
required     | false
type         | String
example      | {{< highlight shell >}}"client_cert_file": "/path/to/ssl/cert.pem"{{< /highlight >}}

| client_key_file | |
-------------|------
description  | Path to the key file associated with the `client_cert_file`
required     | false
type         | String
example      | {{< highlight shell >}}"client_key_file": "/path/to/ssl/key.pem"{{< /highlight >}}

| binding    |      |
-------------|------
description  | The AD account that performs user and group lookups. If your sever supports anonymous binding, you can omit the `user_dn` or `password` attributes to query the directory without credentials. To use anonymous binding with AD, the `ANONYMOUS LOGON` object requires read permissions for users and groups.
required     | false
type         | Map
example      | {{< highlight shell >}}
"binding": {
  "user_dn": "cn=binder,cn=users,dc=acme,dc=org",
  "password": "P@ssw0rd!"
}
{{< /highlight >}}

| group_search |    |
-------------|------
description  | Search configuration for groups. See the [group search attributes](#active-directory-group-search-attributes) for more information.
required     | true
type         | Map
example      | {{< highlight shell >}}
"group_search": {
  "base_dn": "dc=acme,dc=org",
  "attribute": "member",
  "name_attribute": "cn",
  "object_class": "group"
}
{{< /highlight >}}

| user_search |     |
-------------|------
description  | Search configuration for users. See the [user search attributes](#active-directory-user-search-attributes) for more information.
required     | true
type         | Map
example      | {{< highlight shell >}}
"user_search": {
  "base_dn": "dc=acme,dc=org",
  "attribute": "sAMAccountName",
  "name_attribute": "displayName",
  "object_class": "person"
}
{{< /highlight >}}

| default_upn_domain |     |
-------------|------
description  | Enables UPN authentication when set. The default UPN suffix that will be appended to the username when a domain is not specified during login (for example: `user` becomes `user@defaultdomain.xyz`). _WARNING: When using UPN authentication, users must re-authenticate to apply any changes made to group membership on the Active Directory server since their last authentication. To ensure group membership updates are reflected without re-authentication, specify a binding account or enable anonymous binding._
required     | false
type         | String
example      | {{< highlight shell >}}
"default_upn_domain": "example.org"
{{< /highlight >}}

| include_nested_groups |     |
-------------|------
description  | When set to `true`,  group search includes any nested groups instead of just the top level groups that a user is a member of.
required     | false
type         | Boolean
example      | {{< highlight shell >}}
"include_nested_groups": true
{{< /highlight >}}

### Active Directory binding attributes

| user_dn    |      |
-------------|------
description  | The AD account that performs user and group lookups. We recommend using a read-only account. Use the distinguished name (DN) format, such as `cn=binder,cn=users,dc=domain,dc=tld`. If your sever supports anonymous binding, you can omit this attribute to query the directory without credentials.
required     | false
type         | String
example      | {{< highlight shell >}}"user_dn": "cn=binder,cn=users,dc=acme,dc=org"{{< /highlight >}}

| password   |      |
-------------|------
description  | Password for the `user_dn` account. If your sever supports anonymous binding, you can omit this attribute to query the directory without credentials.
required     | false
type         | String
example      | {{< highlight shell >}}"password": "P@ssw0rd!"{{< /highlight >}}

### Active Directory group search attributes

| base_dn    |      |
-------------|------
description  | Tells Sensu which part of the directory tree to search. For example, `dc=acme,dc=org` searches within the `acme.org` directory.
required     | true
type         | String
example      | {{< highlight shell >}}"base_dn": "dc=acme,dc=org"{{< /highlight >}}

| attribute  |      |
-------------|------
description  | Used for comparing result entries. This is combined with other filters as <br> `"(<Attribute>=<value>)"`.
required     | false
type         | String
default      | `"member"`
example      | {{< highlight shell >}}"attribute": "member"{{< /highlight >}}

| name_attribute |  |
-------------|------
description  | Represents the attribute to use as the entry name.
required     | false
type         | String
default      | `"cn"`
example      | {{< highlight shell >}}"name_attribute": "cn"{{< /highlight >}}

| object_class |   |
-------------|------
description  | Identifies the class of objects returned in the search result. This is combined with other filters as `"(objectClass=<ObjectClass>)"`.
required     | false
type         | String
default      | `"group"`
example      | {{< highlight shell >}}"object_class": "group"{{< /highlight >}}

### Active Directory user search attributes

| base_dn    |      |
-------------|------
description  | Tells Sensu which part of the directory tree to search. For example, `dc=acme,dc=org` searches within the `acme.org` directory.
required     | true
type         | String
example      | {{< highlight shell >}}"base_dn": "dc=acme,dc=org"{{< /highlight >}}

| attribute  |      |
-------------|------
description  | Used for comparing result entries. This is combined with other filters as <br> `"(<Attribute>=<value>)"`.
required     | false
type         | String
default      | `"sAMAccountName"`
example      | {{< highlight shell >}}"attribute": "sAMAccountName"{{< /highlight >}}

| name_attribute |  |
-------------|------
description  | Represents the attribute to use as the entry name.
required     | false
type         | String
default      | `"displayName"`
example      | {{< highlight shell >}}"name_attribute": "displayName"{{< /highlight >}}

| object_class |   |
-------------|------
description  | Identifies the class of objects returned in the search result. This is combined with other filters as `"(objectClass=<ObjectClass>)"`.
required     | false
type         | String
default      | `"person"`
example      | {{< highlight shell >}}"object_class": "person"{{< /highlight >}}

### Active Directory metadata attributes

| name       |      |
-------------|------
description  | A unique string used to identify the AD configuration. Names cannot contain special characters or spaces (validated with Go regex [`\A[\w\.\-]+\z`](https://regex101.com/r/zo9mQU/2)).
required     | true
type         | String
example      | {{< highlight shell >}}"name": "activedirectory"{{< /highlight >}}

## Active Directory troubleshooting

See the [LDAP troubleshooting](#ldap-troubleshooting) section.

## OIDC authentication

The Sensu offers license-activated support for OIDC driver for using the OpenID Connect 1.0 protocol (OIDC) on top of the OAuth 2.0 protocol for RBAC authentication.

_NOTE: OIDC authentication is currently supported only via `sensuctl`. OIDC authentication for the Web UI will be added in a future release._

### OIDC configuration examples

{{< language-toggle >}}

{{< highlight yml >}}
---
type: oidc
api_version: authentication/v2
metadata:
  name: oidc_name
spec:
  additional_scopes:
  - groups
  - email
  client_id: a8e43af034e7f2608780
  client_secret: b63968394be6ed2edb61c93847ee792f31bf6216
  redirect_uri: http://127.0.0.1:8080/api/enterprise/authentication/v2/oidc/callback
  server: https://oidc.example.com:9031
  groups_claim: groups
  groups_prefix: 'oidc:'
  username_claim: email
  username_prefix: 'oidc:'
{{< /highlight >}}

{{< highlight json >}}
{
   "type": "oidc",
   "api_version": "authentication/v2",
   "metadata": {
      "name": "oidc_name"
   },
   "spec": {
      "additional_scopes": [
         "groups",
         "email"
      ],
      "client_id": "a8e43af034e7f2608780",
      "client_secret": "b63968394be6ed2edb61c93847ee792f31bf6216",
      " redirect_uri": "http://sensu-backend.example.com:8080/api/enterprise/authentication/v2/oidc/callback",
      "server": "https://oidc.example.com:9031",
      "groups_claim": "groups",
      "groups_prefix": "oidc:",
      "username_claim": "email",
      "username_prefix": "oidc:"
   }
}
{{< /highlight >}}

{{< /language-toggle >}}

### OIDC specification

#### Top-level attributes

type         | 
-------------|------
description  | Top-level attribute specifying the [`sensuctl create`][sc] resource type. For OIDC configuration, always use type `oidc`.
required     | true
type         | String
example      | {{< highlight shell >}}"type": "oidc"{{< /highlight >}}

api_version  | 
-------------|------
description  | Top-level attribute specifying the Sensu API group and version. For OIDC configuration in Sensu Go, always use `authentication/v2`.
required     | true
type         | String
example      | {{< highlight shell >}}"api_version": "authentication/v2"{{< /highlight >}}

metadata     | 
-------------|------
description  | Top-level collection of metadata about the OIDC configuration. The `metadata` map is always at the top level of the OIDC definition. This means that in `wrapped-json` and `yaml` formats, the `metadata` scope occurs outside the `spec` scope.
required     | true
type         | Map of key-value pairs
example      | {{< highlight shell >}}"metadata": {
  "name": "oidc_name"
  }
}{{< /highlight >}}

spec         | 
-------------|------
description  | Top-level map that includes the OIDC [spec attributes][sp].
required     | true
type         | Map of key-value pairs
example      | {{< highlight shell >}}"spec": {
  "additional_scopes": [
    "groups",
    "email"
    ],
  "client_id": "a8e43af034e7f2608780",
  "client_secret": "b63968394be6ed2edb61c93847ee792f31bf6216",
  "redirect_uri": "http://sensu-backend.example.com:8080/api/enterprise/authentication/v2/oidc/callback",
  "server": "https://oidc.example.com:9031",
  "groups_claim": "groups",
  "groups_prefix": "oidc:",
  "username_claim": "email",
  "username_prefix": "oidc:"
  }
}{{< /highlight >}}

##### Metadata attribute

| name       |      |
-------------|------
description  | A unique string used to identify the OIDC configuration. The `metadata.name` cannot contain special characters or spaces (validated with Go regex [`\A[\w\.\-]+\z`](https://regex101.com/r/zo9mQU/2)).
required     | true
type         | String
example      | {{< highlight shell >}}"name": "oidc_name"{{< /highlight >}}

##### Spec attributes {#oidc-spec-attributes}

| additional_scopes |   |
-------------|------
description  | Scopes to include in the claims, in addition to the default `openid` scope. _NOTE: For most providers, you'll want to include `groups`, `email` and `username` in this list._
required     | false
type         | Array
example      | {{< highlight shell >}}"additional_scopes": ["groups", "email", "username"]{{< /highlight >}}

| client_id    |      |
-------------|------
description  | The OIDC provider application "Client ID" _NOTE: requires [registration of an application in the OIDC provider][26]._
required     | true
type         | String
example      | {{< highlight shell >}}"client_id": "1c9ae3e6f3cc79c9f1786fcb22692d1f"{{< /highlight >}}

| client_secret  |      |
-------------|------
description  | The OIDC provider application "Client Secret" _NOTE: requires [registration of an application in the OIDC provider][26]._
required     | true
type         | String
example      | {{< highlight shell >}}"client_secret": "a0f2a3c1dcd5b1cac71bf0c03f2ff1bd"{{< /highlight >}}

| redirect_uri |   |
-------------|------
description  | Redirect URL to provide to the OIDC provider. Requires `/api/enterprise/authentication/v2/oidc/callback` _NOTE: only required for certain OIDC providers, such as Okta._
required     | false
type         | String
example      | {{< highlight shell >}}"redirect_uri": "http://sensu-backend.example.com:8080/api/enterprise/authentication/v2/oidc/callback"{{< /highlight >}}

| server |  |
-------------|------
description  | The location of the OIDC server you wish to authenticate against. _NOTE: Configuring with http will cause the connection to  be insecure._
required     | true
type         | String
example      | {{< highlight shell >}}"server": "https://sensu.oidc.provider.example.com"{{< /highlight >}}

| groups_claim |   |
-------------|------
description  | The claim to use to form the associated RBAC groups. _Note: The value held by the claim must be an array of strings._
required     | false
type         | String
example      | {{< highlight shell >}} "groups_claim": "groups" {{< /highlight >}}

| groups_prefix |   |
-------------|------
description  | A prefix to use to form the final RBAC groups if required.
required     | false
type         | String
example      | {{< highlight shell >}}"groups_prefix": "okta"{{< /highlight >}}

| username_claim |   |
-------------|------
description  | The claim to use to form the final RBAC user name.
required     | false
type         | String
example      | {{< highlight shell >}}"username_claim": "person"{{< /highlight >}}

| username_prefix |   |
-------------|------
description  | A prefix to use to form the final RBAC user name.
required     | false
type         | String
example      | {{< highlight shell >}}"username_prefix": "okta"{{< /highlight >}}

## Register an OIDC application

To use OIDC for authentication, register Sensu Go as an OIDC application. Use the following instructions to register an OIDC application for Sensu Go based on your OIDC provider:

- [Okta](#okta)

### Okta

#### Requirements

- Access to the Okta Administrator Dashboard
- Sensu Go 5.12.0 or later with a valid license

#### Create an Okta application

1. From the Okta Administrator Dashboard, select `Applications > Add Application > Create New App` to start the wizard.
2. Select the `Web` platform and `OpenID Connect` sign in method.
3. In *General Settings*, enter an app name and (optionally) upload a logo.
4. In *Configure OpenID Connect*, add the following redirect URI (replace `DASHBOARD_URL` with the URL for your dashboard: `{DASHBOARD_URL}/api/enterprise/authentication/v2/oidc/callback`
5. Click **Save**.
6. Open the *Sign On* page. In the *OpenID Connect ID Token* section, click **Edit**.
7. Enter the following information for the *Groups* claim attribute:
  - First field: `groups`
  - Dropdown menu: `Regex`
  - Second field: `.*`
8. Click **Save**.
9. Assign people and groups in the *Assignments* page.

#### OIDC driver configuration

1. Add the `aadditional_scopes` configuration attribute in the [OIDC scope][25] and set the value to `[ "groups" ]`:
  - `"additional_scopes": [ "groups" ]`

2. Add the `groups` to the `groups_claim` string. For example, if you have an Okta group `groups` and you set the `groups_prefix` to `okta:`, you can set up RBAC objects to mention group `okta:groups` as needed:
  - `"groups_claim": "okta:groups" `

3. Add the `redirect_uri` configuration attribute in the [OIDC scope][25] and set
  the value to the Redirect URI configured at step 4 of
  [Create an Okta application](#create-an-okta-application):
  - `"redirect_uri": "{BACKEND_URL}/api/enterprise/authentication/v2/oidc/callback"`

#### `sensuctl` login with OIDC

1. Run `sensuctl login oidc`
  - sensuctl login oidc

2. If on a desktop a browser will open to OIDC provider allowing you to authenticate and log in.
  - Launching browser to complete the login via your OIDC provider at following URL:
  - https://sensu-backend.example.com:8080/api/enterprise/authentication/v2/oidc/authorize


[sc]: ../../sensuctl/reference#creating-resources
[sp]: #spec-attributes
[1]: ../../dashboard/overview
[2]: ../../sensuctl/reference
[3]: ../../reference/rbac#default-user
[4]: ../../reference/rbac
[5]: ../../guides/create-read-only-user
[6]: ../../getting-started/enterprise
[7]: https://www.openldap.org/
[8]: ../../api/overview
[9]: ../../api/auth
[10]: ../../reference/rbac#roles-and-cluster-roles
[11]: ../../reference/rbac#cluster-roles
[12]: ../../reference/rbac#role-bindings-and-cluster-role-bindings
[13]: ../../reference/rbac#cluster-role-bindings
[17]: ../../reference/rbac#namespaced-resource-types
[18]: ../../reference/rbac#cluster-wide-resource-types
[19]: ../../guides/troubleshooting#log-levels
[20]: #managing-authentication-providers
[21]: #group-search-attributes
[22]: #user-search-attributes
[23]: #active-directory-metadata-attributes
[24]: #metadata-attributes
[25]: #oidc-spec-attributes
[26]: #register-an-oidc-application
