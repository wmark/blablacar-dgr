# dgr - container build and runtime tool


[![GoDoc](https://img.shields.io/badge/godoc-reference-5874B0.svg)](https://godoc.org/github.com/blablacar/dgr)
[![Build Status](https://img.shields.io/travis/blablacar/dgr/master.svg)](https://travis-ci.org/blablacar/dgr)

<img src="https://raw.githubusercontent.com/blablacar/dgr/gh-pages/logo.png" width="300">

**dgr** (pronounced "*digg-er*") is a command line utility designed to build and to configure at runtime App Containers Images ([ACI](https://github.com/appc/spec/blob/master/spec/aci.md)) and App Container Pods ([POD](https://github.com/appc/spec/blob/master/spec/pods.md)) based on convention over configuration.

dgr allows you to build generic container images for a service and to configure them at runtime. Therefore you can use the same image for different environments, clusters, or nodes by overriding the appropriate attributes when launching the container.

_dgr is actively used at blablacar to build and run more than an hundred different aci and pod to [run all platforms](http://blablatech.com/blog/why-and-how-blablacar-went-full-containers)._


## Build the ACI once, configure your app at runtime.

dgr provides various resources to build and configure an ACI:

- scripts at runlevels (build, prestart...)
- templates and attributes
- static files
- images dependencies

**Scripts** are executed at the image build, before your container is started and more. See [runlevels](#runlevels) for more information.

**Templates** and **attributes** are the way dgr deals with environment-specific configurations. **Templates** are stored in the image and resolved at runtime ; **attributes** are inherited from different contexts (aci -> pod -> environment).

**Static files** are copied to same path in the container.

**Image dependencies** are used as defined in [APPC spec](https://github.com/appc/spec/blob/master/spec/aci.md#dependency-matching).


![demo](https://raw.githubusercontent.com/blablacar/dgr/gh-pages/aci-dummy.gif)


## Comparison with alternatives

### dgr vs Dockerfile

A Dockerfile is purely configuration, describing the steps to build the container. It does not provide a common way of building containers across a team.
It does not provide scripts levels, ending with very long bash scripting for the run option in the dockerfile.
It does not handle configuration, nor at build time nor at runtime.

### dgr vs acbuild

acbuild is a command line tools to build ACIs. It is more flexible than Dockerfiles as it can be wrapped by other tools such as Makefiles but like Dockerfiles it doesn't provide a standard way of configuring the images. acbuild does not provide image dependencies during build nor builder dependencies.

## Commands

```bash
$ dgr init          # init a sample project
$ dgr build         # build the image
$ dgr clean         # clean the build
$ dgr clean install # clean, build and install the build
$ dgr clean push    # clean, build and push the build
$ dgr install       # store target image to rkt local store
$ dgr push          # push target image to remote storage
$ dgr test          # test the final image
```

## Configuration file

dgr global configuration is a yaml file located at `~/.config/dgr/config.yml`. Home is the home of starting user (the caller user if running with sudo).
It is used to indicate the target work directory where dgr will create the ACI and the push endpoint informations. Both are optional.

Example of configuration:

```yml
targetWorkDir: /tmp/target     # if you want to use another directory for all builds
push:
  type: maven
  url: https://localhost/nexus
  username: admin
  password: admin
```


# Building an ACI

## Initializing a new project

Run the following commands to initialize a new project:

```bash
$ mkdir aci-myapp
$ cd aci-myapp
$ dgr init
```

It will generate the following file tree:

```text
.
|-- attributes
|   `-- attributes.yml                 # Attributes files that will be merged and used to resolve templates
|-- aci-manifest.yml                   # Manifest
|-- templates
|   |-- etc
|   |   |-- templated.tmpl             # template file that will end up at /etc/templated
|   |   `-- templated.tmpl.cfg         # configuration of the targeted file, like user and mode (optional file)
|   `-- header.partial                 # template part that can be included in template files
|-- files
|   `-- dummy                          # Files to be copied to the same location in the target rootfs
|-- runlevels
|   |-- builder
|   |   `-- 10.prepare.sh              # Scripts to be run inside the builder to prepare the aci for build
|   |-- build
|   |   `-- 10.install.sh              # Scripts to be run when building inside aci's rootfs
|   |-- build-late
|   |   `-- 10.build-late.sh           # Scripts to be run when building inside aci's rootfs after the copy of files
|   |-- inherit-build-early
|   |   `-- 10.inherit-build-early.sh  # Scripts stored in ACI and executed while used as a dependency
|   |-- inherit-build-late
|   |   `-- 10.inherit-build-early.sh  # Scripts stored in ACI and executed while used as a dependency
|   |-- prestart-early
|   |   `-- 10.prestart-early.sh       # Scripts to be run when starting ACI before templating
|   `-- prestart-late
|       `-- 10.prestart-late.sh        # Scripts to be run when starting ACI after templating
`-- tests
    |-- dummy.bats                     # Bats tests for this ACI
    `-- wait.sh                        # Script to wait until the service is up before running tests
```

This project is already valid which means that you can build it and it will result in a runnable ACI (dgr always adds busybox to the ACI). But you probably want to customize it at this point.


## Customizing

### The manifest

The dgr manifest looks like a light ACI manifest. dgr will take this manifest and convert it to the format defined in the APPC spec.

Example of a *aci-manifest.yml*:

```yaml
name: example.com/myapp:0.1

builder:
  dependencies:
    - example.com/base:1

tester:
  dependencies:
    - example.com/base:1
    
aci:
  dependencies:
    - example.com/base:1
  app:
    exec:
      - /bin/myapp
      - -c
      - /etc/myapp/myapp.cfg
    mountPoints:
      - name: myapp-data
        path: /var/lib/myapp
        readOnly: false
```

The **name**, well, is the name of the ACI you are building.
Under the **aci** key, you can add every key that is defined in the [APPC spec](https://github.com/appc/spec/blob/master/spec/aci.md) such as:

- **exec** which contains the absolute path to the executable your want to run at the start of the ACI and its args.
- **mountPoints** even though you can do it on the command line with recent versions of RKT.
- **isolators**...

Except **handlers** that are directly mapped to **prestart** runlevels 

### The build scripts

The scripts in `runlevels/build` dir are executed during the build to install in the ACI everything you need. For instance if your dependencies are based on debian, a build script could look like:

```bash
#!/bin/bash
apt-get update
apt-get install -y myapp
```

### Templates

You can create templates in your ACI. Templates are stored in the ACI as long as attributes and are resolved at start of the container.

Example:

*templates/etc/resolv.conf.tmpl*

```
{{ range .dns.nameservers -}}
nameserver {{ . }}
{{ end }}

{{ if .dns.search -}}
search {{ range .dns.search }} {{.}} {{end}}
{{end}}
```

*templates/etc/resolv.conf.tmpl.cfg*

```
uid: 0
gid: 0
mode: 0644
checkCmd: /dgr/bin/busybox true
```

`checkCmd` is a command to run after the templating to check that the configuration is valid or fail container start.

When you have to reuse the same part in multiple templates, you can create a partial template like defined in the [go templating](https://golang.org/pkg/text/template/#hdr-Nested_template_definitions).

*templates/header.partial*

```
{{define "header"}}
whatever
{{end}}
```

and include it in a template:

```
{{template "header" .}}
```

Templater provides functions to manipulate data inside the template. Here is the list:

| Tables    |      Function        |  Description                                                |
|-----------|:---------------------|:------------------------------------------------------------|
| base      | path.Base            |                                                             |
| split     | strings.Split        |                                                             |
| json      | UnmarshalJsonObject  |                                                             |
| jsonArray | UnmarshalJsonArray   |                                                             |
| dir       | path.Dir             |                                                             |
| getenv    | os.Getenv            |                                                             |
| join      | strings.Join         |                                                             |
| datetime  | time.Now             |                                                             |
| toUpper   | strings.ToUpper      |                                                             |
| toLower   | strings.ToLower      |                                                             |
| contains  | strings.Contains     |                                                             |
| replace   | strings.Replace      |                                                             |
| orDef     | orDef                | if first element is nil, use second as default              |
| orDefs    | orDefs               | if first array param is empty use second element to fill it |
| ifOrDef   | ifOrDef              | if first param is not nil, use second, else third           |

It also provide all function defined by [gtf project](https://github.com/leekchan/gtf)

*We can add functions on demand*

### Attributes

All the YAML files in the directory **attributes** are read by dgr. The first node of the YAML has to be "default" as it can be overridden in a POD or with a json in the env variable TEMPLATER_OVERRIDE in the cmd line.

*attributes/resolv.conf.yml*

```
default:
  dns:
    nameservers:
      - "8.8.8.8"
      - "8.8.4.4"
    search:
      - bla.com
```

### The prestart

dgr uses the "pre-start" eventHandler of the ACI to customize the ACI rootfs before the run depending on the instance or the environment.
It resolves at that time the templates so it has all the context needed to do that.
You can also run custom scripts before (prestart-early) or after (prestart-late) this template resolution. This is useful if you want to initialize a mountpoint with some data before running your app for instance.

*runlevels/prestart-late/init.sh*

```bash
#!/bin/bash
set -e
/usr/bin/myapp-init
```

## Troubleshoot

dgr start by default with info log level. You can change this level with the `-L` command line argument.
The log level is also propaged to all runlevels with the environment variable: **LOG_LEVEL**.

You can activate debug on demand by including this code in your scripts:

```bash
source /dgr/bin/functions.sh
isLevelEnabled "debug" && set -x
```

and for build-setup runlevel (that is not running inside the container):

```bash
source ${TARGET}/rootfs/dgr/bin/functions.sh
isLevelEnabled "debug" && set -x
```


## Ok, but concretely how should I use it? (Need rework since deprecation of from)

Currently, Linux distrib and package managers are not designed to support container the way it should be used. Especially on the **from** & **dependencies** parts.

So, here is how we are using it at Blablacar: where you want to use a package manager like *apt*, you should debootstrap a debian version in *build_setup* to create a **aci-debian**. Then use this image as *from* in the other aci to be able to run *apt get install* at *build* runlevel.

When you want to install application without dependencies, like Go (with libc linking), or java application:
create an **aci-base** image that will just copy the libc, and template minor stuff like */etc/hosts*, */etc/hostname* and */etc/resolv.conf* to this image. then use this tiny image as from for you go application. For a Java application, use this image from and **aci-java** as dependencies.

Globally, our rule is to have only basic images as from to provide package manager, and get hand maid dependencies using *dependencies* tag.

*We are working on a cleaner solution to install application in the container without the need of the package manager coming along.*


## Building a POD


### Standard FileTree for POD

TODO

```bash
├── aci-elasticsearch               # Directory that match the pod app shortname (or name)
│   ├── attributes
│   │   └── attributes.yml          # Attributes file for templating in this ACI
│   ├── files                       # Files to be inserted into this ACI
│   ...
├── pod-manifest.yml            # Pod Manifest

```



## Requirement

- [rkt](https://github.com/coreos/rkt) in your `$PATH` or configured in dgr global conf
- being root is required to construct the filesystem
- linux >= 3.18 with overlay filesystem

## I want to extend dgr

If you think your idea can be integrated directly in the core of dgr, please create an issue or a pull request.

If you want want to extend the way the **builder** is working (attributes, templates, files, ...), you can create a new **stage1 builder** and replace the internal one with : 
```
...
builder:
  image: dgrtool.com/aci-builder:1
...
```
You can do the same for the **tester**.




