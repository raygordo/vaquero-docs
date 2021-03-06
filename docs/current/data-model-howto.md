---
layout: page
page.title: Data Model
---
# The Vaquero Data Model and YOU
[Home]({{ site.url }}) | [Docs Repo](https://github.com/CiscoCloud/vaquero-docs/tree/master)

- [Example Data Models](https://github.com/CiscoCloud/vaquero-examples)

## Table of Contents

1. [Key Concepts](#key-concepts)

2. [Terminology](#terminology)

3. [SOT layout](#sot-layout)

4. [Provisioning Steps](#provisioning-steps)

5. [Serving files](#serving-files)

6. [Metadata and Templating](#metadata-and-templating)

7. [Translation to iPXE](#translation-to-ipxe)

8. [Schemas](#schemas)

9. [Staging Updates](#staging)

### Introduction

The Vaquero data model is meant to be a declarative representation of the state of your data center. You specify the state you want your bare metal to be in, and Vaquero takes the steps to get there.

We treat this data model as a "single source of truth" (SoT) that describes the operating state of your data center. The data model is [parsed and verified](tools.html), then deployed to an on-site Vaquero Agent for execution.

## <a name="key-concepts">Key Concepts</a>

Your data center is expressed as an inventory of `hosts`. Each host belongs to a `workflow`. Each workflow is comprised of one or many `boot` steps that use a combination of `unattended assets` and `operating system` definitions to define a target configured state for a host.

## <a name="terminology">Terminology</a>

*Site*: A managed data center, or group of machines managed by a single Vaquero Agent.

*Host*: A single managed machine. Definition includes identifying attributes (selectors), host-specific metadata, information for LOM (IPMI), and an association to a single workflow.

*Cluster*: A grouping of hosts under a specific site.

*Configuration*: The general name given for the collection of `Operating System`, `Assets`, `Boot` and `Workflows`. Please refer below for explanation.

*Operating System*: An "installation" template containing the details to perform a network boot into a particular OS, specifying kernel, initrd, boot command-line parameters, unattended config, etc.

*Unattended Assets*: An optionally templated unattended config/script (i.e. cloud-init, ignition, kickstart, etc) used for unattended boot and installation operations.

*Boot*: A collection that ties together operating systems and unattended assets. Describes a single network boot.

*Workflow*: A series of network boots that end with a host machine in a desired state.

*SOT (Source of Truth)*: The name given to the collection of both `configurations` and `sites` that is necessary to define a data center. Vaquero supports two types of SOT: a basic SOT or an advanced SOT. The former is simpler to manage, whereas the latter provides better access control and separation of responsibilities.

## Basic SOT

A basic SOT is where all sites and configuration files are defined together in the same local directory or github repo. It's simple to manage and is useful for smaller environments.

![Vaquero Basic SOT]({{ site.url }}/img/vaquero-basic-sot.png)


## Advanced SOT

In this mode, sites are stored in a single github repo or local directory, while configuration files can be spread out across multiple github repos or local directories. This provides a way to see all defined infrastructure in a centralized way, while separating the configuration for each cluster.

![Vaquero Advanced SOT]({{ site.url }}/img/vaquero-advanced-sot.png)

## <a name="sot-layout">SOT Layout</a>

Configuration files are placed in a directory hierarchy. Vaquero parses site `configurations` by reading files placed in specially named subdirectories. The root of your configuration path has four directories:

1. **assets**: grouped by type. These are generally unattended configs or scripts that have been templated to include environment-specific information. Contains named subdirectories (more on that later).
2. **os**: Individual documents, each representing an Operating System
3. **boot**: Individual documents, each representing a Boot
4. **workflows**: Individual documents, each representing a Workflow

`Sites` are another component to a complete SOT. `Sites` are responsible for defining agent-specific settings (networks, asset and TFTP directories, etc.) and the set of hosts the agent is responsible for provisioning.


### Assets

`Assets` are grouped into named subdirectories based on type. There are currently four types:

1. Cloud-Config: [CoreOS Cloudinit System](https://coreos.com/os/docs/latest/cloud-config.html)
2. Ignition: [CoreOS Ignition](https://coreos.com/ignition/docs/latest/)
3. Kickstart: [Fedora Project Kickstart](http://fedoraproject.org/wiki/Anaconda/Kickstart)
4. Untyped: Miscellaneous files. Can be used for "unsupported" configuration types.

Each subdirectory may also have a `snippets` directory for holding partial templates (see below)

Each `asset` is placed under a subdirectory according to it's type. `Assets` are referenced by file name from boots:

```
.
└── assets
    ├── cloud-config
    │   └── base.yml
    ├── ignition
    │   ├── etcd.yml
    │   └── raid-fmt.yml
    ├── kickstart
    │   ├── snippets
    │   │   ├── snip1
    │   │   └── snip2
    │   └── centos.ks
    └── untyped
        ├── autoyast.xml
        └── preseed.cfg
```


Validation is performed on typed assets to verify that rendered templates produce valid configuration scripts.

`Assets` are retrieved dynamically from the Vaquero Agent asset server through the `/config/<mac-addr>` endpoint. An optional `boot` query parameter can be used to specify the ID of the Boot to use.

For instance, a host with mac address `00:00:00:00:00:01` could retrieve it's default configuration by requesting

```
{% raw %}
  {{ .env.assetURL }}/config/00:00:00:00:00:01
{% endraw %}
```

Or the configuration from a particular boot by requesting

```
{% raw %}
{{.env.assetURL}}/config/00:00:00:00:00:01?boot=specific-boot-id
{% endraw %}
```

#### Asset Snippets

Any snippets for a particular config type are `always included` when rendering configurations of that type. Most of the time, the preferred use will be to have the snippet file `define` a template, and use the `template` function to include it in the configuration:

```
{% raw %}
/assets/kickstart/snippets/snip1

{{ define "snippet-id-1" }}
  # Template here
{{ end }}
{% endraw %}
```

```
{% raw %}
/assets/kickstart/centos.ks

# My kickstart template
{{ template "snippet-id-1" . }}
{% endraw %}
```

### Operating Systems

Operating systems exist as individual documents under the `os` subdirectory. They are referenced by a self-assigned ID described in the document:

```
.
└── os
    ├── centos-7.yml
    ├── clevos-3.yml
    └── coreos-1053.12.0.yml
```

### Boot

Boots exist as individual documents under the `boot` subdirectory. They are referenced by a self-assigned ID described in the document:

```
.
└── boot
    ├── etcd-cluster.yml
    └── etcd-proxy.yml
```

### Workflows

Workflows exist as individual documents under the `workflows` subdirectory. They are referenced by a self-assigned ID described in the document:

```
.
└── workflows
    ├── clevos-accessor.yml
    ├── k8s-master.yml
    └── k8s-node.yml
```

### Sites

Sites are represented by individual subdirectories under the top level `sites` directory. Each `site` under the `sites` directory corresponds to a managed group
of machines.

Each site requires one file named `env.yml`, all other files will be treated as a cluster definition that describe their host inventory. You may use YAML's
triple-dash `---` separator to combine multiple inventory documents into one cluster file.

#### Clusters

The hosts can be grouped into multiple logical `clusters` in a `site`. Clusters are represented by individual yaml files under a site (by convention these files
are prefixed with with `cluster-` in the file name (example: `cluster-prod.yml`).


```
.
└── sites
    ├── site-a
    │   ├── env.yml
    │   ├── cluster-dev.yml
    │   ├── cluster-stage.yml
    │   └── cluster-prod.yml
    └── site-b
    |   ├── env.yml
    |   ├── cluster-dev.yml
    |   ├── cluster-stage.yml
    |   └── cluster-prod.yml

```

## Basic SOT

In the basic SOT, your configuration -- `assets`, `os`, `boot` and `workflows` -- are defined alongside your `sites`. This enables all `clusters` to share the same configuration, which minimizes the overall complexity of managing your SOT.

An example of a basic SOT structure with two sites with multiple clusters each:

```
.
├── assets
├── os
├── boot
├── workflows
└── sites
    ├── denver
    │   ├── env.yml
    │   ├── cluster-dev.yml
    │   ├── cluster-stage.yml
    │   └── cluster-prod.yml
    └── reno
        ├── env.yml
        ├── cluster-dev.yml
        ├── cluster-stage.yml
        └── cluster-prod.yml
```

In the above example, all clusters defined in the denver and reno sites will share the same configuration. For small teams, this may be appropriate if all administrators are allowed to have access to all configuration. In the event that multiple teams should not see configuration from each other, the advanced SOT below may be more suitable.

## Advanced SOT

For advanced SOTs, each `cluster` can point to their own configuration located on a Github repository or another location on disk. This allows administrators to separate the responsibility of managing the configuration of their `sites`.

An example of two sites with dev and prod configuration:

```
.
└── sites
    ├── denver
    │   ├── env.yml
    │   ├── cluster-dev.yml
    │   └── cluster-prod.yml
    └── reno
        ├── env.yml
        ├── cluster-dev.yml        
        └── cluster-prod.yml
```

In the above example, your reno site is able to manage their production configuration separately from both their development config as well as the reno `site`. This allows for separation of responsibility across the teams administering your datacenters and minimizes the opportunity for sensitive information to be exposed.

In order to link a `cluster` with a `configuration`, place a `config` section at the top of every cluster file. This section describes how to obtain the SOT configuration for this cluster of hosts.

#### Github SOT

```yaml
---
config:
  type: git
  url: "https://github.com/someuser/vaquero-configurations"
  ref: "master"
  token: "github_api_token_here"
```

#### Local SOT

```yaml
---
config:
  type: local
  url: "/var/vaquero/local-config"
```

## <a name="provisioning-steps">Provisioning Steps</a>

Configurations are roughly executed in the following order:

1. Host makes DHCP request.
2. DHCP causes host to chainload iPXE (`undionly.kpxe`) and indicates Vaquero Agent as next-server
3. Vaquero Agent provides a default iPXE script to discover host interface information (mac)
4. Host requests dynamic iPXE script based on basic information
5. Vaquero Agent renders iPXE script using os, boot, and host information
6. Host executes iPXE script, requesting resources (kernel, intitrd, unattended configs/scripts) as required

The default ipxe script chains back to Vaquero Agent, injecting basic information:

```bash
#!ipxe
chain ipxe?uuid=${uuid}&mac=${net0/mac:hexhyp}&domain=${domain}&hostname=${hostname}&domain=${domain}
```


## <a name="serving-files">Serving Files</a>

Vaquero Agent will expose an endpoint `/files` for hosting static content. This endpoint acts transparently as a file server, or a reverse proxy, according to
the configuration file.

## Identifying a Host

A booting machine is identified as a particular host based on the selecting information used. Currently, a host will be identified by `mac`, as reported by iPXE. According to validation rules, every host has an exclusive collection of interfaces (all unique by mac address).

Additionally, the first non-BMC type (excluding SSH) interface listed for the host is the `Primary Interface` for that host. The Primary Interface is the preferred interface to use when provisioning hosts.

## <a name="metadata-and-templating">Metadata and Templating</a>

Templates are written using [Go's standard templates](https://golang.org/pkg/text/template/). Templated information occurs in the following areas:

1. In any files under `assets`
2. In os objects in `boot.kernel`, `boot.initrd`, and values in `cmdline`

Metadata is used primarily to render templated information. It is "unstructured" data, consisting of nested key-value maps, and lists. Metadata is included in three separate places in your configuration:

1. In the environment `env.yml` file
2. In a boot file
3. In an inventory document, under each host

Metadata is made available during template execution as separate fields under the template's "dot". With a few exceptions, each entry directly relates to the scheme. The cases where proxies or other values are included are noted:

1. `.env`: the site's Environment information.
  - `.env.agentURL`: The scheme://host:port that the host can use to reach Vaquero Agent.
2. `.boot`: the current Boot
  - `.boot.configURL`: The scheme://host:port/path?query needed to retrieve the unattended configuration information for the boot. Used for manually inserting config retrieval, i.e. ClevOS answers.
  - `.boot.{operating_system,os}`: the OS ID is replaced with the Operating System object that it refers to
3. `.host`: the current Host
  - `.host.interfaces.subnet`: subnet ID is replaced with Subnet object that it refers to
4. `.interface`: the Interface that the Host is using to connect with Vaquero Agent
  - `.interface.subnet`: subnet ID is replaced with Subnet object that it refers to

* Any object/scheme that includes `metadata` proxies the same information under `md`
* ex: `env.metadata.initial_etcd_cluster` == `env.md.initial_etcd_cluster`

By way of example, this template snippet defines a networkd configuration:

```yaml
{% raw %}
networkd:
  units:
    - name: 10-static.network
      contents: |
        [Match]
        MACAddress={{.interface.mac}}
        [Network]
        Gateway={{.interface.subnet.gateway}}
        DNS={{index .interface.subnet.dns 0}}
        Address={{.interface.ipv4}}
{% endraw %}
```

## <a name="translation-to-ipxe">Translation to iPXE</a>

Currently, all network boots and installations are performed using iPXE scripts. Operating system boot parameters and command line options are translated into iPXE scripts to perform boot/installation tasks.

Any unattended configs or scripts included in a boot are inserted during this process. Inconsistencies (i.e. using ignition for a CentOS operating system) should be detected during validation.

### Command Line Parameters

Rules for translating command line parameters:

1. Keys with empty values (i.e. "" or '') are formatted as `key` in the boot options
2. Keys with non-empty values are formatted as `key=value` in the boot options
3. Keys that contain a list will be included once each time for every list element. If a list element cannot be parsed not a string (i.e. is a map, list, etc), it is ignored.

For example, given this OS:

```yaml
---
id: centos-example
major_version: '7'
minor_version: '2'
os_family: CentOS
release_name: stable
boot:
  kernel: centos_kernel
  initrd:
  - centos_initrd
cmdline:
  - console:
    - ttyS0,115200
    - ttyS1
    - nested_map: is
      ignored: true
    - same:
      - with
      - nested
      - lists
  - lang: ' '
  - debug: ''
  - enforcing: ''
```


The iPXE script will be roughly generated as (not taking unattended info from boot):

```bash
#!ipxe
kernel centos_kernel console=ttyS0,115200 console=ttyS1 lang=  debug enforcing
initrd centos_initrd
boot
```

**For UEFI you must add an initrd to the cmdline. Bug: https://github.com/coreos/bugs/issues/1239**

```bash
#!ipxe
kernel http://127.0.0.1:24601/files/coreos_production_pxe.vmlinuz coreos.autologin initrd=coreos_production_pxe_image.cpio.gz coreos.first_boot coreos.config.url=http://127.0.0.1:24601/config/00:00:00:00:00:01?boot=etcd-master
initrd http://127.0.0.1:24601/files/coreos_production_pxe_image.cpio.gz
boot
```

Note how `lang` appears with a trailing `=`, because it's value was non-empty `' '`


## <a name="schemas">Schemas</a>

### boot

Defines a configured state (combination of os with unattended configuration and metadata) that may be applied to a group of hosts.

| name             | description                                               | required | schema          | default |
|:-----------------|:----------------------------------------------------------|:---------|:----------------|:--------|
| id               | A self-assigned identifier (should be unique)             | yes      | string          |         |
| name             | A human-readable name for this group                      | no       | string          | id      |
| operating_system | The ID of the os associated with this group               | yes      | string          |         |
| unattended       | Unattended config/script details                          | no       | boot.unattended |         |
| metadata         | unstructured, boot-specific information                   | no       | object          |         |
| validate         | A series of containers run to ensure proper configuration | no       | container       |         |
| before_shutdown  | Containers to run before a manual reboot/reprovision      | no       | container       |         |

#### boot.unattended

Allow a network boot or installation to proceed automatically by providing canned answers.

| name | description                                             | required | schema | default |
|:-----|:--------------------------------------------------------|:---------|:-------|:--------|
| type | The type of unattended config/script to use             | yes      | string |         |
| use  | The file name used to find the unattended config/script | yes      | string |         |

### container

| name          | description                              | required | schema        |
|:--------------|:-----------------------------------------|:---------|:--------------|
| image         | Docker image name (e.g. alpine:latest)   | yes      | string        |
| commands      | List of commands to run (via /bin/sh -c) | yes      | list          |
| env           | Map of environment variables.            | no       | map           |
| registry_auth | Registry Credentials                     | no       | registry_auth |

#### container.registry_auth

| name       | description                | required | schema  |
|:-----------|:---------------------------|:---------|:--------|
| url        | api versioned registry url | yes      | string  |
| username   | registry username          | yes      | string  |
| password   | registry password          | yes       | string  |

### env

Provides information for a single deployment/data center/etc.


| name                           | description                                               | required | schema           | default |
|:-------------------------------|:----------------------------------------------------------|:---------|:-----------------|:--------|
| id                             | A self-assigned identifier (should be unique)             | yes      | string           |         |
| name                           | A human-readable name for this group                      | no       | string           | id      |
| [agents](#envagent)            | Details for establishing a connection to the site's agent | yes      | env.agent  array |         |
| [subnets](#envsubnet)          | List of subnets for this cluster                          | yes      | env.subnet array |         |
| metadata                       | unstructured, site-specific information                   | no       | object           |         |
| [release_tag](#envrelease_tag) | Github release tag                                        | no       | string           |         |


#### env.agent

Details for establishing a connection to a site's agent


| name                                 | description                              | required | schema                 | default            |
|:-------------------------------------|:-----------------------------------------|:---------|:-----------------------|:-------------------|
| [asset_server](#envagentassetserver) | Asset Server configuration               | no       | env.agent.asset_server |                    |
| dhcp_mode                            | The mode to run DHCP in, server or proxy | no       | string                 | server             |
| save_path                            | The dir path to save local files         | no       | string                 | /var/vaquero       |
| ssh_container                        | Docker registry URL for ssh container.   | no       | string                 | gemtest/openssh<sup>1</sup>  |
| ipmi_container                       | Docker registry URL for IPMI container.  | no       | string                 | gemtest/ipmitool<sup>2</sup> |

<sup>1. docker.io/gemtest/openssh</sup><br />
<sup>2. docker.io/gemtest/ipmitool</sup>

#### env.agent.asset_server

Configuration for the asset server


| name       | description                                                   | required | schema  | default            |
|:-----------|:--------------------------------------------------------------|:---------|:--------|:-------------------|
| addr       | Asset Server configuration                                    | no       | string  | 127.0.0.1          |
| port       | The mode to run DHCP in, server or proxy                      | no       | integer | server             |
| base_dir   | The directory that vaquero agent will use to serve files from | no       | string  | /var/vaquero/files |
| scheme     | The protocol scheme to use for the agent                      | no       | string  | http               |
| cdn_addr   | The address of the CDN vaquero agent should reverse proxy to  | no       | string  |                    |
| cdn_port   | The port of the CDN vaquero agent should reverse proxy to     | no       | integer |                    |
| cdn_scheme | The scheme of the CDN vaquero agent should reverse proxy to   | no       | string  | http               |


By declaring cdn_addr and a cdn_port we will use that as a source. The agent will serve the asset_server off `0.0.0.0:<port>` so an agent can be dual homed. Vaquero agent has logic to create ipxe scripts on the proper interface that is routable from the booting host. The `env.agent.asset_server.addr` is used as a fall back address in ipxe scripts.

The transport (http/s) should be included with the agent URL.

#### env.agent.tftp_server

Configuration for the tftp_server

| name       | description                                                   | required | schema  | default            |
|:-----------|:--------------------------------------------------------------|:---------|:--------|:-------------------|
| base_dir | The directory that vaquero agent will use to serve tftp files from  | no       | string  | /var/vaquero/pxeroms  |


#### env.release_tag
`release_tag` must be a valid [commit_ish](https://git-scm.com/docs/git#git-ltcommit-ishgt) string corresponding to a specific [Github Release](https://help.github.com/articles/about-releases/) (e.g., v0.1.0) or commit_id.

If `release_tag` is specified, Vaquero will attempt to use the data model stored in the specified release instead of _this_ model (i.e. where release_tag was specified). If the tag does not exist, Vaquero will fall back to using _this_ model.

This option is only supported when [staging via Github](#staging).

###### Example
Branch `master` defines three sites: `site-a` with `release_tag: v0.1.0`, `site-b` with `release_tag: v0.1.1`, and `site-c` with no tag.

When Vaquero loads `master`, it will end up using three different data models for the three different sites. `site-a` will get the version of itself defined in release `v0.1.0`, `site-b` will get `v0.1.1` and `site-c` will get the version defined in `master`.

#### env.subnet

| name         | description                                          | required | schema             | default |
|:-------------|:-----------------------------------------------------|:---------|:-------------------|:--------|
| id           | A self-assigned identifier (should be unique in env) | yes      | string             |         |
| cidr         | CIDR for this subnet                                 | yes      | string             |         |
| dns          | List of DNS URLs                                     | yes      | string array       |         |
| ntp          | List of NTP URLs                                     | yes      | string array       |         |
| gateway      | Gateway for this subnet                              | no       | string             |         |
| domain_name  | Domain name for this subnet                          | no       | string             |         |
| vlan         | VLAN for the subnet                                  | no       | integer            | 1       |
| dhcp_options | Additional DHCP options                              | no       | dhcp_options array |         |
| metadata     | unstructured, host-specific information              | no       | object             |         |


#### env.subnet.dhcp_options

Represents a single DHCP Option as defined in [RFC2132](http://www.iana.org/go/rfc2132) or listed in [this IANA table](http://www.iana.org/assignments/bootp-dhcp-parameters/bootp-dhcp-parameters.xhtml) of BOOTP Vendor Extensions and DHCP Options.


| name   | description                                       | required | schema   | default                                                                                                      |
|:-------|:--------------------------------------------------|:---------|:---------|:-------------------------------------------------------------------------------------------------------------|
| option | DHCP option tag.                                  | yes      | uint8    |                                                                                                              |
| value  | List of DNS URLs                                  | yes      | variable |                                                                                                              |
| type   | Denotes the type of `value`. Accepted values:     | yes      | string   |                                                                                                              |
|        | string, uint8, uint16, uint32, int8, int16, int32 |          |          |                                                                                                              |
|        | addresses<sup>1</sup>, base64<sup>2</sup>         |          |          | |         |         |                                                      |          |                    | |

<sup>1. Type `addresses` is a comma seperated string of ip addresses.</sup><br>
<sup>2. Type `base64` is a base64 encoded value.</sup>

[Examples](dhcp-options.html)

| name    | description                                          | required | schema       | default |
|:--------|:-----------------------------------------------------|:---------|:-------------|:--------|
| id      | A self-assigned identifier (should be unique in env) | yes      | string       |         |
| cidr    | CIDR for this subnet                                 | yes      | string       |         |
| dns     | List of DNS URLs                                     | yes      | string array |         |
| ntp     | List of NTP URLs                                     | yes      | string array |         |
| domain  | Client DNS domain                                    | no       | string       |         |
| gateway | Gateway for this subnet                              | no       | string       |         |
| vlan    | VLAN for the subnet                                  | no       | integer      | 1       |

#### config

| name  | description                                        | required          | schema | default |
|:------|:---------------------------------------------------|:------------------|:-------|:--------|
| type  | The type of config (local, git)                    | yes               | object |         |
| url   | The url for git repository or local path of sot    | yes               | string |         |
| ref   | The Git reference (branch, SHA, tag, HEAD)         | yes (if type git) | string |         |
| token | The token to use to call Github API                | yes               | string |         |

#### host

| name       | description                                        | required | schema    | default |
|:-----------|:---------------------------------------------------|:---------|:----------|:--------|
| name       | Name for the host machine.                         | yes      | string    |         |
| interfaces | Network interfaces for this host                   | no       | interface |         |
| metadata   | unstructured, host-specific information            | no       | object    |         |
| workflow   | The ID of the workflow used to provision this host | yes      | string    |         |
| never_provision   | Prevent Vaquero from rebooting/provisioning hosts. | no      | string    |     false    |


#### interface

| name        | description                                                | required | schema        | default |
|:------------|:-----------------------------------------------------------|:---------|:--------------|:--------|
| type        | Interface type. physical or bmc<sup>1</sup>                | yes      | string        |         |
| mac         | MAC address identifying this interface                     | yes      | string        |         |
| subnet      | ID of subnet (specified in env)                            | yes      | string        |         |
| bmc         | Details for BMC interface<sup>2</sup>                      | no       | interface.bmc |         |
| identifier  | Identifier for interface                                   | no       | string        |         |
| ignore_dhcp | If true, stops Vaquero from provisioning on this interface | no       | boolean       |         |
| ipv4        | IPv4 address                                               | yes      | dotted quad   |         |
| ipv6        | IPv6 address                                               | no       | string        |         |
| hostname    | Hostname for machine                                       | no       | string        |         |

<sup>1. An interface of type `physical` can define an `interface.bmc` for ssh power management _only_ -- i.e. a physical interface may *not* include an `interface.bmc.type` set to `ipmi`.</sup><br>
<sup>2. An interface of type `bmc` describes a power management interface. This interface will not be used for PXE booting the machine (but it may acquire an IP from vaquero's DHCP Server).</sup>

##### interface.bmc

| name         | description                       | required | schema    | default |
|:-------------|:----------------------------------|:---------|:-------   |:--------|
| type         | Specifies protocol type. IPMI/SSH | yes      | string    |         |
| username     | User for managing BMC             | ipmi/ssh | string    |         |
| password     | Password for specified user       | ipmi     | string    |         |
| keypath      | File path to ssh private key      | ssh      | string    |         |
| container    | Custom bmc reboot container       | custom   | container |         |
| soft_reboot  | Attempt Graceful Shutdown.<sup>1</sup>        | no       | bool      | false   |
| soft_timeout | Soft Reboot Timeout (seconds)<sup>2</sup>    | no       | integer   | 0    |

<sup>1. By default, IPMI will do a power cycle. `soft_reboot` may be specified to do a soft (graceful) restart instead.</sup><br>
<sup>2. `soft_timeout` specifies the amount of time to try a soft reboot. If the machine is not able to restart in this time, a power cycle is issued. `soft_timeout=0` means the host will never be forcefully restarted.</sup>

This bmc struct defines the method used to reboot the host. There are three possible configurations:

-  Use IPMI with the provided credentials to reboot the machine:

```
type: ipmi
username: ipmiusername
password: ipmipassword
soft_reboot: true
soft_timeout: 30
```

-  SSH into the machine and do a sudo reboot. This requires key management, which is left as an exercise to the reader.

```
type: ssh
username: core
keypath: /home/core/.ssh/id_rsa
```

-  Use a custom container to reboot the machine.

```
type: custom
container:
  image: vaquero.registry.com/ipmi:latest
  pull: yes
  commands:
    - ipmitool -H 10.10.10.103 -I imb -U root -P secret power cycle
  timeout: 60
```

### os

Represents a single operating system with boot/installation parameters.


| name          | description                      | required | schema       | default |
|:--------------|:---------------------------------|:---------|:-------------|:--------|
| id            | self-assigned identifier         | yes      | string       |         |
| name          | human-readable name              | yes      | string       | id      |
| major_version | major version                    | yes      | string       |         |
| minor_version | minor version                    | no       | string       |         |
| os_family     | family (i.e. CoreOS, CentOS)     | yes      | string       |         |
| release_name  | release name (i.e. stable, beta) | no       | string       |         |
| boot          | kernal & initrd img info         | yes      | os.boot      |         |
| cmdline       | boot/installation options        | no       | string array |         |


Cmdline values may be templated. They will be rendered on-demand for individual hosts.

**Note initrd must be added in the cmdline to work with UEFI. Bug: https://github.com/coreos/bugs/issues/1239**

```yaml
{% raw %}
---
id: coreos-1053.2.0-stable
name: CoreOS Stable 1053.2.0
major_version: '1053'
minor_version: '2.0'
os_family: CoreOS
release_name: stable
boot:
  kernel: "{{.env.agentURL}}/files/{{.boot.os.release_name}}/{{.boot.os.major_version}}/{{.boot.os.minor_version}}/coreos_production_pxe.vmlinuz"
  initrd:
  - "{{.env.agentURL}}/files/coreos_production_pxe_image.cpio.gz"
cmdline:
  - coreos.autologin: ''
  #This line is needed for EFI PXE boots. https://github.com/coreos/bugs/issues/1239
  - initrd: "coreos_production_pxe_image.cpio.gz"
{% endraw %}
```

#### os.boot

Contains information about the kernal/initrds for an operating system.


| name   | description                             | required | schema       | default |
|:-------|:----------------------------------------|:---------|:-------------|:--------|
| kernel | URL for retrieving kernel on boot       | yes      | string       |         |
| initrd | URL for retrieving initrds/imgs on boot | yes      | string array |         |



Kernel and initrd values may be templated. They will be rendered on-demand for inidividual hosts.

### workflow

A workflow chains multiple boots together to provision a host. The workflow is also responsible for specifying basic policy for rebutting hosts that use it.

| name             | description                                                      | required   | schema                 | default   |
| :--------------- | :--------------------------------------------------------------- | :--------- | :--------------------- | :-------- |
| id               | self-assigned identifier                                         | yes        | string                 |           |
| workflow         | Series of boots to provision the host                            | yes        | workflow.stage array   |           |
| deps             | Host provisioning rollout policy                                              | no         | workflow.deps          |           |


#### workflow.deps

Workflow deps defines interworkflow dependencies and specifies policy for provisioning all hosts in this workflow.

| name             | description                                                                | required   | schema                 | default   |
| :--------------- | :---------------------------------------------------------------           | :--------- | :--------------------- | :-------- |
| max_concurrent   | Max simultaneous hosts actively provisioning                               | no         | int                    | 0         |
| min_standing     | Minimum number of hosts that must be active to unblock dependant workflows | no         | int                    | 0         |
| block_deps       | IDs of blocking dependency workflows                                       | no         | string array           |           |
| validate_on      | IDs of workflows that cause this workflow to rerun validation              | no         | string array           |           |
| max_fail         | How many hosts can fail before halted                                      | no         | int                    | 0         |

*Be aware that `min_standing` could become a blocking condition, if `min_standing` is set to 3 and there are only 3 hosts in that workflow.*

## <a name="staging">Staging Updates</a>

Github will be used to stage models for updating, vaquero will receive webhooks from specified branches. Submitting PR's and merging other branches into the vaquero branch would be how teams manage updating their source of truth. Once a model lands in the branch vaquero is watching, it will push it out and begin provisioning against that source of truth.
