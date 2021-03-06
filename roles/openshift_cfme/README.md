# CloudForms Availability

As noted in [Limitations - Product Choice](#product-choice),
[CloudForms](https://www.redhat.com/en/technologies/management/cloudforms)
(CFME) 4.6 is not yet released. Until such time, this role is limited
to installing [ManageIQ](http://manageiq.org) (MIQ), the open source
project that CFME is based on.

After CFME 4.6 is available to customers this role will enable
(optional) logic which will install CFME or MIQ based on your
deployment type (`openshift_deployment_type`):

* `openshift-enterprise` → CloudForms
* `origin` → ManageIQ


# Table of Contents

   * [Introduction](#introduction)
      * [Important Notes](#important-notes)
   * [Requirements](#requirements)
   * [Role Variables](#role-variables)
   * [Getting Started](#getting-started)
      * [All Defaults](#all-defaults)
      * [External NFS Storage](#external-nfs-storage)
      * [Override PV sizes](#override-pv-sizes)
      * [Override Memory Requirements](#override-memory-requirements)
      * [External PostgreSQL Database](#external-postgresql-database)
   * [Limitations](#limitations)
      * [Product Choice](#product-choice)
   * [Configuration](#configuration)
      * [Database](#database)
         * [Podified](#podified)
         * [External](#external)
      * [Storage Classes](#storage-classes)
         * [NFS (Default)](#nfs-default)
         * [NFS External](#nfs-external)
         * [Cloud Provider](#cloud-provider)
         * [Preconfigured (Expert Configuration Only)](#preconfigured-expert-configuration-only)
   * [Customization](#customization)
   * [Additional Information](#additional-information)

# Introduction

This role will allow a user to install CFME 4.6 or MIQ on an OCP
3.7 cluster. The role provides customization options for overriding
default deployment parameters. This role allows the user to deploy
different installation flavors:

* **Fully Podified** - In this way all application services are ran as
  pods in the container platform.
* **External Database** - In this way the application utilizes an
  externally hosted database server. All other services are ran in the
  container platform.

This role includes the following storage class options:

* NFS - **Default** - local, on cluster
* NFS External - NFS somewhere else, like a storage appliance
* Cloud Provider - Use automatic storage provisioning from your cloud
  provider (*gce* or *aws*)
* Preconfigured - **expert only**, assumes you created everything ahead
  of time

You may skip ahead to the [Getting Started](#getting-started) section
now for examples of how to set up your Ansible inventory for various
deployment configurations. However, you are **strongly urged** to
first read through the [Configuration](#configuration) and
[Customization](#customization) sections as well as the following
[Important Notes](#important-notes).

## Important Notes

Not all parameters are present in **both** template versions (podified
db and external db). For example, while the podified database template
has a `POSTGRESQL_MEM_REQ` parameter, no such parameter is present in
the external db template, as there is no need for this information due
to there being no databases that require pods.

*Be extra careful* if you are overriding template
parameters. Including parameters not defined in a template **will
cause errors**.

**Container Provider Integration** - If you want add your container
platform (OCP/Origin) as a *Container Provider* in CFME/MIQ then you
must ensure that the infrastructure management hooks are installed.

* During your OCP/Origin install, ensure that you have the
  `openshift_use_manageiq` parameter set to `true` in your inventory
  at install time. This will create a `management-infra` project and a
  service account user.
* After CFME/MIQ is installed, obtain the `management-admin` service
  account token and copy it somewhere safe.

```bash
$ oc serviceaccounts get-token -n management-infra management-admin
eyJhuGdiOiJSUzI1NiIsInR5dCI6IkpXVCJ9.eyJpd9MiOiJrbWJldm5lbGVzL9NldnZpY2VhY2NvbW50Iiwiy9ViZXJuZXRldy5puy9zZXJ2yWNlYWNju9VubC9uYW1ld9BhY2UiOiJtYW5hZ2VtZW50LWluZnJhIiwiy9ViZXJuZXRldy5puy9zZXJ2yWNlYWNju9VubC9zZWNyZXQuumFtZSI6Im1humFnZW1lunQtYWRtyW4tbG9rZW4tdDBnOTAiLCJrbWJldm5lbGVzLmlvL9NldnZpY2VhY2NvbW50L9NldnZpY2UtYWNju9VubC5uYW1lIjoiuWFuYWbluWVubC1hZG1puiIsImt1YmVyumV0ZXMuyW8vd2VybmljZWFjY291unQvd2VybmljZS1hY2NvbW50LnVpZCI6IjRiZDM2MWQ1LWE1NDAtMTFlNy04YzI5LTUyNTQwMDliMmNkZCIsInN1YiI6InN5d9RluTpzZXJ2yWNlYWNju9VubDptYW5hZ2VtZW50LWluZnJhOm1humFnZW1lunQtYWRtyW4ifQ.B6sZLGD9O4vBu9MHwiG-C_4iEwjBXb7Af8BPw-LNlujDmHhOnQ-Oo4QxQKyj9edynfmDy2yutUyJ2Mm9HfDGWg4C9xhWImHoq6Nl7T5_9djkeGKkK7Ejvg4fA-IkrzEsZeQuluBvXnE6wvP0LCjUo_dx4pPyZJyp46teV9NqKQeDzeysjlMCyqp6AK6-Lj8ILG8YA6d_97HlzL_EgFBLAu0lBSn-uC_9J0gLysqBtK6TI0nExfhv9Bm1_5bdHEbKHPW7xIlYlI9AgmyTyhsQ6SoQWtL2khBjkG9TlPBq9wYJj9bzqgVZlqEfICZxgtXO7sYyuoje4y8lo0YQ0kZmig
```

* In the CFME/MIQ web interface, navigate to `Compute` →
  `Containers` → `Providers` and select `⚙ Configuration` → `⊕
  Add a new Containers Provider`

*See the [upstream documentation](http://manageiq.org/docs/reference/latest/doc-Managing_Providers/miq/index.html#containers-providers) for additional information.*



# Requirements

The **default** requirements are listed in the table below. These can
be overridden through customization parameters (See
[Customization](#customization), below).

**Note** that the application performance will suffer, or possibly
even fail to deploy, if these requirements are not satisfied.


| Item                | Requirement   | Description                                  | Customization Parameter       |
|---------------------|---------------|----------------------------------------------|-------------------------------|
| Application Memory  | `≥ 4.0 Gi`    | Minimum required memory for the application  | `APPLICATION_MEM_REQ`         |
| Application Storage | `≥ 5.0 Gi`    | Minimum PV size required for the application | `APPLICATION_VOLUME_CAPACITY` |
| PostgreSQL Memory   | `≥ 6.0 Gi`    | Minimum required memory for the database     | `POSTGRESQL_MEM_REQ`          |
| PostgreSQL Storage  | `≥ 15.0 Gi`   | Minimum PV size required for the database    | `DATABASE_VOLUME_CAPACITY`    |
| Cluster Hosts       | `≥ 3`         | Number of hosts in your cluster              |                               |

The implications of this table are summarized below:

* You need several cluster nodes
* Your cluster nodes must have lots of memory available
* You will need several GiB's of storage available, either locally or
  on your cloud provider
* PV sizes can be changed by providing override values to template
  parameters (see also: [Customization](#customization))

# Role Variables

The following is a table of the publicly exposed variables that may be
used in your Ansible inventory to control the behavior of this
installer.


| Variable                                       | Required | Default                        | Description                         |
|------------------------------------------------|:--------:|:------------------------------:|-------------------------------------|
| `openshift_cfme_project`                       | **No**   | `openshift-cfme`               | Namespace for the installation.     |
| `openshift_cfme_project_description`           | **No**   | *CloudForms Management Engine* | Namespace/project description.      |
| `openshift_cfme_install_cfme`                  | **No**   | `false`                        | Boolean, set to `true` to install the application |
| **PRODUCT CHOICE**  | | | | |
| `openshift_cfme_app_template`                  | **No**   | `miq-template`                 | The project flavor to install. Choices: <ul><li>`miq-template`: ManageIQ using a podified database</li> <li> `miq-template-ext-db`: ManageIQ using an external database</li> <li>`cfme-template`: CloudForms using a podified database<sup>[1]</sup></li> <li> `cfme-template-ext-db`: CloudForms using an external database.<sup>[1]</sup></li></ul> |
| **STORAGE CLASSES** | | | | |
| `openshift_cfme_storage_class`                 | **No**   | `nfs`                          | Storage type to use, choices: <ul><li>`nfs` - Best used for proof-of-concept installs. Will setup NFS on a cluster host (defaults to your first master in the inventory file) to back the required PVCs. The application requires a PVC and the database (which may be hosted externally) may require a second. PVC minimum required sizes are 5GiB for the MIQ application, and 15GiB for the PostgreSQL database (20GiB minimum available space on a volume/partition if used specifically for NFS purposes)</li> <li>`nfs_external` - You are using an external NFS server, such as a netapp appliance. See the [Configuration - Storage Classes](#storage-classes) section below for required information.</li> <li>`preconfigured` - This CFME role will do NOTHING to modify storage settings. This option assumes expert knowledge and that you have done everything required ahead of time.</li> <li>`cloudprovider` - You are using an OCP cloudprovider integration for your storage class. For this to work you must have already configured the required inventory parameters for your cloud provider. Ensure `openshift_cloudprovider_kind` is defined (aws or gce) and that the applicable cloudprovider parameters are provided. |
| `openshift_cfme_storage_nfs_external_hostname` | **No**   | `false`                        | If you are using an *external NFS server*, such as a netapp appliance, then you must set the hostname here. Leave the value as `false` if you are not using external NFS. <br /> *Additionally*: **External NFS REQUIRES** that you create the NFS exports that will back the application PV and optionally the database PV.
| `openshift_cfme_storage_nfs_base_dir`          | **No**   | `/exports/`                    | If you are using **External NFS** then you may set the base path to the exports location here. <br />**Local NFS Note**: You *may* also change this value if you want to change the default path used for local NFS exports. |
| `openshift_cfme_storage_nfs_local_hostname`    | **No**   | `false`                        | If you do not have an `[nfs]` group in your inventory, or want to simply manually define the local NFS host in your cluster, set this parameter to the hostname of the preferred NFS server. The server must be a part of your OCP/Origin cluster. |
| **CUSTOMIZATION OPTIONS** | | | | |
| `openshift_cfme_template_parameters`           | **No**   | `{}`                           | A dictionary of any parameters you want to override in the application/pv templates.

* <sup>[1]</sup> The `cfme-template`s will be available and
  automatically detected once CFME 4.6 is released


# Getting Started

Below are some inventory snippets that can help you get started right
away.

If you want to install CFME/MIQ at the same time you install your
OCP/Origin cluster, ensure that `openshift_cfme_install_cfme` is set
to `true` in your inventory. Call the standard
`playbooks/byo/config.yml` playbook to begin the cluster and CFME/MIQ
installation.

If you are installing CFME/MIQ on an *already provisioned cluster*
then you can call the CFME/MIQ playbook directly:

```
$ ansible-playbook -v -i <YOUR_INVENTORY> playbooks/byo/openshift-cfme/config.yml
```

*Note: Use `miq-template` in the following examples for ManageIQ installs*

## All Defaults

This example is the simplest. All of the default values and choices
are used. This will result in a fully podified CFME installation. All
application components, as well as the PostgreSQL database will be
created as pods in the container platform.

```ini
[OSEv3:vars]
openshift_cfme_app_template=cfme-template
```

## External NFS Storage

This is as the previous example, except that instead of using local
NFS services in the cluster it will use an external NFS server (such
as a storage appliance). Note the two new parameters:

* `openshift_cfme_storage_class` - set to `nfs_external`
* `openshift_cfme_storage_nfs_external_hostname` - set to the hostname
  of the NFS server

```ini
[OSEv3:vars]
openshift_cfme_app_template=cfme-template
openshift_cfme_storage_class=nfs_external
openshift_cfme_storage_nfs_external_hostname=nfs.example.com
```

If the external NFS host exports directories under a different parent
directory, such as `/exports/hosted/prod` then we would add an
additional parameter, `openshift_cfme_storage_nfs_base_dir`:

```ini
# ...
openshift_cfme_storage_nfs_base_dir=/exports/hosted/prod
```

## Override PV sizes

This example will override the PV sizes. Note that we set the PV sizes
in the template parameters, `openshift_cfme_template_parameters`. This
ensures that the application/db will be able to make claims on created
PVs without clobbering each other.

```ini
[OSEv3:vars]
openshift_cfme_app_template=cfme-template
openshift_cfme_template_parameters={'APPLICATION_VOLUME_CAPACITY': '10Gi', 'DATABASE_VOLUME_CAPACITY': '25Gi'}
```

## Override Memory Requirements

In a test or proof-of-concept installation you may need to reduce the
application/database memory requirements to fit within your
capacity. Note that reducing memory limits can result in reduced
performance or a complete failure to initialize the application.

```ini
[OSEv3:vars]
openshift_cfme_app_template=cfme-template
openshift_cfme_template_parameters={'APPLICATION_MEM_REQ': '3000Mi', 'POSTGRESQL_MEM_REQ': '1Gi', 'ANSIBLE_MEM_REQ': '512Mi'}
```

Here we have instructed the installer to process the application
template with the parameter `APPLICATION_MEM_REQ` set to `3000Mi`,
`POSTGRESQL_MEM_REQ` set to `1Gi`, and `ANSIBLE_MEM_REQ` set to
`512Mi`.

These parameters can be combined with the PV size override parameters
displayed in the previous example.

## External PostgreSQL Database

To use an external database you must change the
`openshift_cfme_app_template` parameter value to `miq-template-ext-db`
or `cfme-template-ext-db`.

Additionally, database connection information **must** be supplied in
the `openshift_cfme_template_parameters` customization parameter. See
[Customization - Database - External](#external) for more
information.

```ini
[OSEv3:vars]
openshift_cfme_app_template=cfme-template-ext-db
openshift_cfme_template_parameters={'DATABASE_USER': 'root', 'DATABASE_PASSWORD': 'r1ck&M0r7y', 'DATABASE_IP': '10.10.10.10', 'DATABASE_PORT': '5432', 'DATABASE_NAME': 'cfme'}
```

# Limitations

This release is the first OpenShift CFME release in the OCP 3.7
series. It is not complete yet.

## Product Choice

Due to staggered release dates, **CFME support is not
integrated**. Presently this role will only deploy a ManageIQ
installation. This role will be updated once CFME 4.6 is released and
this limitation note will be removed.

# Configuration

Before you can deploy CFME you must decide *how* you want to deploy
it. There are two major decisions to make:

1. Do you want an external, or a podified database?
1. Which storage class will back your PVs?

## Database

### Podified

Any `POSTGRES_*` or `DATABASE_*` template parameters in
[miq-template.yaml](files/templates/manageiq/miq-template.yaml) or
[cfme-template.yaml](files/templates/cloudforms/cfme-template.yaml)
may be customized through the `openshift_cfme_template_parameters`
hash.

### External

Any `POSTGRES_*` or `DATABASE_*` template parameters in
[miq-template-ext-db.yaml](files/templates/manageiq/miq-template-ext-db.yaml)
or
[cfme-template-ext-db.yaml](files/templates/cloudforms/cfme-template-ext-db.yaml)
may be customized through the `openshift_cfme_template_parameters`
hash.

External PostgreSQL databases require you to provide database
connection parameters. You must set the required connection keys in
the `openshift_cfme_template_parameters` parameter in your
inventory. The following keys are required:

* `DATABASE_USER`
* `DATABASE_PASSWORD`
* `DATABASE_IP`
* `DATABASE_PORT` - *note: Most PostgreSQL servers run on port `5432`*
* `DATABASE_NAME`

Your inventory would contain a line similar to this:

```ini
[OSEv3:vars]
openshift_cfme_app_template=cfme-template-ext-db
openshift_cfme_template_parameters={'DATABASE_USER': 'root', 'DATABASE_PASSWORD': 'r1ck&M0r7y', 'DATABASE_IP': '10.10.10.10', 'DATABASE_PORT': '5432', 'DATABASE_NAME': 'cfme'}
```

**Note** the new value for the `openshift_cfme_app_template`
parameter, `cfme-template-ext-db` (ManageIQ installations would use
`miq-template-ext-db` instead).

At run time you may run into errors similar to this:

```
TASK [openshift_cfme : Ensure the CFME App is created] ***********************************
task path: /home/tbielawa/rhat/os/openshift-ansible/roles/openshift_cfme/tasks/main.yml:74
Tuesday 03 October 2017  15:30:44 -0400 (0:00:00.056)       0:00:12.278 *******
{"cmd": "/usr/bin/oc create -f /tmp/postgresql-ZPEWQS -n openshift-cfme", "kind": "Endpoints", "results": {}, "returncode": 1, "stderr": "Error from server (BadRequest): error when creating \"/tmp/postgresql-ZPEWQS\": Endpoints in version \"v1\" cannot be handled as a Endpoints: [pos 218]: json: decNum: got first char 'f'\n", "stdout": ""}
```

Or like this:

```
TASK [openshift_cfme : Ensure the CFME App is created] ***********************************
task path: /home/tbielawa/rhat/os/openshift-ansible/roles/openshift_cfme/tasks/main.yml:74
Tuesday 03 October 2017  16:05:36 -0400 (0:00:00.052)       0:00:18.948 *******
fatal: [m01.example.com]: FAILED! => {"changed": true, "failed": true, "msg":
{"cmd": "/usr/bin/oc create -f /tmp/postgresql-igS5sx -n openshift-cfme", "kind": "Endpoints", "results": {}, "returncode": 1, "stderr": "The Endpoints \"postgresql\" is invalid: subsets[0].addresses[0].ip: Invalid value: \"doo\": must be a valid IP address, (e.g. 10.9.8.7)\n", "stdout": ""},
```

While intimidating at first, there are useful bits of information in
here. Examine the error output closely and we can tell exactly what is
wrong.

In the first example we see `Endpoints in version \"v1\" cannot be
handled as a Endpoints: [pos 218]: json: decNum: got first char
...`. This is because in my example I used the value `foo` for the
parameter `DATABASE_PORT`.

In the second example we see `The Endpoints \"postgresql\" is invalid:
subsets[0].addresses[0].ip: Invalid value: \"doo\": must be a valid IP
address ...`. This is because in my example I used the value `doo` in
the `DATABASE_IP` field.

Luckily for us when the templates are processed behind the scenes they
are also running type checking validation. So, don't worry, just look
closely at the errors and ensure you are providing the correct values
for each parameter.

## Storage Classes

OpenShift CFME supports several storage class options.

### NFS (Default)

The NFS storage class is best suited for proof-of-concept and
test/demo deployments. It is also the **default** storage class for
deployments. No additional configuration is required for this
choice.

Customization is provided through the following role variables:

* `openshift_cfme_storage_nfs_base_dir`
* `openshift_cfme_storage_nfs_local_hostname`

### NFS External

External NFS leans on pre-configured NFS servers to provide exports
for the required PVs. For external NFS you must have:

* For CFME: a `cfme-app` and optionally a `cfme-db` (for podified database) exports
* For ManageIQ: an `miq-app` and optionally an `miq-db` (for podified database) exports

Configuration is provided through the following role variables:

* `openshift_cfme_storage_nfs_external_hostname`
* `openshift_cfme_storage_nfs_base_dir`

The `openshift_cfme_storage_nfs_external_hostname` parameter must be
set to the hostname or IP of your external NFS server.

If `/exports` is not the parent directory to your exports then you
must set the base directory via the
`openshift_cfme_storage_nfs_base_dir` parameter.

For example, if your server export is `/exports/hosted/prod/cfme-app`
then you must set
`openshift_cfme_storage_nfs_base_dir=/exports/hosted/prod`.

### Cloud Provider

CFME can also use a cloud provider storage to back required PVs. For
this functionality to work you must have also configured the
`openshift_cloudprovider_kind` variable and all associated parameters
specific to your chosen cloud provider.

Using this storage class, when the application is created the required
PVs will automatically be provisioned using the configured cloud
provider storage integration.

There are no additional variables to configure the behavior of this
storage class.

### Preconfigured (Expert Configuration Only)

The *preconfigured* storage class implies that you know exactly what
you're doing and that all storage requirements have been taken care
ahead of time. Typically this means that you've already created the
correctly sized PVs.

There are no additional variables to configure the behavior of this
storage class.

# Customization

Application and database parameters may be customized by means of the
`openshift_cfme_template_parameters` inventory parameter.

**For example**, if you wanted to reduce the memory requirement of the
PostgreSQL pod then you could configure the parameter like this:

`openshift_cfme_template_parameters={'POSTGRESQL_MEM_REQ': '1Gi'}`

When the CFME template is processed `1Gi` will be used for the value
of the `POSTGRESQL_MEM_REQ` template parameter.

Any parameter in the `parameters` section of the
[miq-template.yaml](files/templates/manageiq/miq-template.yaml) or
[miq-template-ext-db.yaml](files/templates/manageiq/miq-template-ext-db.yaml)
may be overridden through the `openshift_cfme_template_parameters`
hash. This applies to **CloudForms** installations as well:
[cfme-template.yaml](files/templates/cloudforms/cfme-template.yaml),
[cfme-template-ext-db.yaml](files/templates/cloudforms/cfme-template-ext-db.yaml).


# Additional Information

The upstream project,
[@manageiq/manageiq-pods](https://github.com/ManageIQ/manageiq-pods),
contains a wealth of additional information useful for managing and
operating your CFME installation. Topics include:

* [Verifying Successful Installation](https://github.com/ManageIQ/manageiq-pods#verifying-the-setup-was-successful)
* [Disabling Image Change Triggers](https://github.com/ManageIQ/manageiq-pods#disable-image-change-triggers)
* [Scaling CFME](https://github.com/ManageIQ/manageiq-pods#scale-miq)
* [Backing up and Restoring the DB](https://github.com/ManageIQ/manageiq-pods#backup-and-restore-of-the-miq-database)
* [Troubleshooting](https://github.com/ManageIQ/manageiq-pods#troubleshooting)
