# OpenShift Migration Pipeline

This Jenkins pipeline migrates namespaces from one OpenShift platform to another one.

What does the pipline migrate:

* Projects, including Labels
* Imagestreams
* Images, which don't have the imported annotation and are still available on the source imagestream
* DeploymentConfigs, Image Triggers(`spec.triggers[ImageChange].imageChangeParams.from.namespace`), namespaces are migrated if needed source to destination Namesapce. References in Containers (`spec.template.spec.container.image`) are migratied source to destination Namespace.
* Deployments, References in Containers(`spec.template.spec.container.image`) are migrated source to destination Namespace.
* Statefulsets, References in Containers(`spec.template.spec.container.image`) are migrated source to destination Namespace.
* BuildConfigs
* Services, except the gluster services
* Routes, default Names like `http(s)://name-[source_namespace].source.app.domain` are migrated to `http(s)://name-[destination_namespace].destination.app.domain`
* ConfigMaps
* Secrets (non existent and starting with ["builder-dockercfg-", "default-dockercfg-", "deployer-dockercfg-", "builder-token-", "default-token-", "deployer-token-"])
* Service Accounts (non existend and starting with ["builder", "default", "deployer"])
* Roles
* RoleBindings (only Rolebindins that starts with ["admin", "edit", "view"]), Project Membership
* PVCs, volumename element will be deleted and autoprovisioned by the gluster cluster. takes `status.capacity.storage` as size.
  * PVC labeled with `pitc-storage-class` will migrate the storage class to the given value
* CronJobs
* PrometheusRules
* Limitranges
* Quotas
* Templates

## How does it work

The migration job is based on the jenkins [openshift client plugin](https://github.com/openshift/jenkins-client-plugin) and runs as a parametrized jenkins job inside OpenShift

Trigger the parametirzed build and define soure and destination namespace. We use the customtool plugin to deploy the oc tool.

**Important:** Nothing in the source namespace will be touched, it only will be exported.

## Setup

First we need two service accounts on both the source and destination cluster, we also create two namespaces `migration-pipeline` which can be deleted after the migration easily.

**Source Cluster**, the SA needs the permission to read all resources in all namespaces:

```bash
$ oc new-project migration-pipeline
$ oc create -f jenkins-migration-sa.yaml -n migration-pipeline
```

**Destination Cluster**, the SA needs the permission to create namespaces

```bash
$ oc new-project migration-pipeline
$ oc create -f jenkins-migration-sa.yaml -n migration-pipeline
$ oc adm policy add-cluster-role-from-user self-provisioners -z jenkins-migration-sa -n migration-pipeline
```

Before you're able to run the Pipeline you'll have to make sure, that the credentials to your openshift instances are available within your jenkins server.

Source OpenShift Cluster

* OpenShift client plugin credential (Type: OpenShift Token for OpenShift Client Plugin) with id `openshift_source_jenkins_migration_prod_token_client_plugin` token of service account
* OpenShift registry credential (Type: Username Password) with id `openshift-source-prod-jenkins-migration-username-password` Username of the service account, pw token

Destination OpenShift Cluster

* OpenShift client plugin credential (Type: OpenShift Token for OpenShift Client Plugin) with id `openshift_destination_jenkins_migration_prod_token_client_plugin` token of service account
* OpenShift registry credential (Type: Username Password) with id `openshift-destination-prod-jenkins-migration-username-password` Username of the service account pw token

Read the token:

```bash
$ oc serviceaccounts get-token jenkins-migration-sa -n migration-pipeline
```

The Migration Pipeline and its configuration is available in the JenkinsFile - File. Create your own Git Repo containing a JenkinsFile with your own configuration; `environment` Section of the JenkinsFile, then create a JenkinsJob which connects to the given git repo.

It's also possible to setup the migration pipeline as [OpenShift Pipeline](https://docs.openshift.com/container-platform/3.11/dev_guide/openshift_pipeline.html) within an OpenShift Cluster.

### Jenkins Dependencies

Your Jenkins Installation (at least 2.190.x) will need the following Plugins / Tools installed:

* Custom Tools Plugin, you can either use the custom tools plugin as way to install the oc-tool on your jenkins slaves or deploy the oc-tool manually, in that case remove the oc tool parts from your environment configuration in the JenkinsFile
* Pipeline
* OpenShift Pipeline Jenkins Plugin

# Things we had to consider at the Puzzle OpenShift Migration

## DNS

DNS entries must be migrated manually, make sure to do the migration together with your customer if you don't have access to the DNS configuration.

## Let's encrypt integration

We move the the new acme integration, you'll have to annotate your let's encrypt routes with the following annotation `kubernetes.io/tls-acme: true`

## CI/CD Integration with jenkins

Make sure to configure your CI/CD Pipelines so that it uses the new cluster.

## SMTP Server

you might need to reconfigure your SMTP Server using Authentication.
SPF Records, if you send emails from a different Domain, you might need to update the SPF Record: https://www.spf-record.de/index

## External Database

The external Database must be migrated as well.

## IP Whitelisting

Make sure that customer with IP Whitelisting in place, configure the new IP addresses for out and inbound trafic.

## Data within PV

The script doesn't currently migrate the data within PVs.

You'll need to migrate the data manually

* scale down the pod in the source namespace, make sure all Files are written to the PV, before your start migrating it
* deploy a debug pod, where you mount the PV
* use oc rsync or cp to synch all data to your local machine
* scale down the pod in the destination namespace, make sure all Files are written to the PV, before your start migrating it
* deploy a debug pod, where the PV is mounted to
* remove all data on the PV
* use oc rsync or cp to synch the data to the PVC
* scale the deployment up again to the replica value before the migration

Export and import a database Dump for Database files

## Migrate PV data
Dont use this with databases, if it's possible to create a db-dump

1. Login to the old cluster
3. Stop the Application (scale down)
4. Take the pv-name and find out the vol_id with get_pv_path.py
```
# Example
./get_pv_path.py prometheus-k8s-db-prometheus-k8s-1
vol_7dfff8644d7e1f38c5df02ac54232fb1
```
5. With the vol_id you can mount the gluster-device on the bastion-host
```
ssh your-user@bastion01.cloud.puzzle.ch
# gluster-endpoints
10.2.111.63

# mount pv ready-only
mount -t glusterfs [endpoint]:vol_id /root/data_migration/old/ -o ro
```
6. Login to the New Cluster
```
oc login https://yourcluster
```
7. Same steps as in Step 4 to get the new vol_id
8. Mount the new pv also on the bastion-host
```
mount -t glusterfs [endpoint]:vol_08672969b7f7aaa2971f324330a16836 /root/data_migration/mnt/ -o rw
```
9. Copy /root/data_migration/old to /root/data_migration/mnt
```
rsync -avxrSHA /root/data_migration/old/* /root/data_migration/mnt/

# -a archive
# -v verbose
# -x don't cross filesystem boundaries
# -r recursive
# -S handle sparse files efficiently
# -H preserve Hardlinks
# -A preserve ACL's
```
10. Unmount and remove tmp dir
```
umount /root/data_migration/mnt
umount /root/data_migration/old
```

## Known issues

* verify Image Streams after the migration script ran
