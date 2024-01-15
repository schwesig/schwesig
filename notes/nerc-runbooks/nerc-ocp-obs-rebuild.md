- [ ] wip - from nerc-ocp-infra-rebuild.md to nerc-ocp-obs-rebuild.md

# Procedure Overview

1. [Bootstrapping nerc-ocp-infra](#bootstrapping-nerc-ocp-infra)
1. [Hostpath Provisioner](#hostpath-provisioner)
1. [Vault](#vault)
1. [External Secrets](#external-secrets)
1. [NMState](#nmstate)
1. [Open Data Foundations (ODF)](#odf)
1. [ArgoCD](#argocd)
1. [Observability](#observability)
1. [Loki](#loki)
1. [Advanced Cluster Management (ACM)](#acm)
1. [Troubleshooting](#troubleshooting)

## Bootstrapping nerc-ocp-infra

### Assisted installer

Login to the redhat console: <https://console.redhat.com/openshift>

Click the blue "Create cluster" button then select the "Datacenter" tab. Under
the "Assisted Installer" section in the datacenter tab, click the blue "Create
cluster" button.

Fill in the cluster details:

```yaml
name: nerc-ocp-obs
base_domain: rc.fas.harvard.edu
version: 4.10.x (?tbd 4.14.x)
```

Leave the rest of the defaults and click next.

Leave the operators page defaults and press next.

On the "Host discovery" page press the "Add hosts" button.

In the "Add hosts" dialogue select the "Full image file" radio button.

Copy the contents of the nerc-admin01 pub key in `/root/.ssh` into the "SSH
public key" text area in the "Add hosts" dialogue.

Next click the blue "Generate Discovery ISO" button.

Download the generated discovery ISO either using the direct URL or the "wget"
command provided.

Copy the discovery ISO to nerc-bootstrap and extract

```sh
$ ssh root@nerc-admin01
[root@nerc-admin01 ~]# ssh nerc-bootstrap
[root@nerc-bootstrap ~]# mkdir /srv/pxe/httpboot/openshift-discovery-iso/nerc-ocp-infra/$OPENSHIFT_VERSION_HERE
[root@nerc-bootstrap ~]# cd /srv/pxe/httpboot/openshift-discovery-iso/nerc-ocp-infra/4.10.47
[root@nerc-bootstrap 4.10.47]# wget -O discovery_image_nerc-ocp-infra.iso '$DISCOVERY_ISO_URL_HERE'
[root@nerc-bootstrap 4.10.47]# bash /srv/repos/boot-openshift-ai/extract-iso.sh discovery_image_nerc-ocp-infra.iso
config.ign
18 blocks
[root@nerc-bootstrap 4.10.47]# ls
config.ign  discovery_image_nerc-ocp-infra.iso  images
```

Submit a merge request to nerc-ansible repo to set `ocp_version` for the three
controllers to `$OPENSHIFT_VERSION_HERE` used previously. Example merge request
to follow:

<https://gitlab-int.rc.fas.harvard.edu/nerc/nerc-ansible/-/merge_requests/49>

Next PXE boot `ctl-[0-2]` via OBM

Wait for all hosts to show up in the "Add hosts" page and then press the Next
button.

Leave the defaults on the "Storage" page and press Next.

On the "Network" page leave the defaults and fill in the API and Ingress IPs
with the following addresses:

```yaml
api_ip: 10.30.9.5
ingress_ip: 10.30.9.6
```

These come from from DNS, e.g.:

```sh
nslookup api.nerc-ocp-infra.rc.fas.harvard.edu
nslookup lb.nerc-ocp-infra.rc.fas.harvard.edu
```

Press the "Next" button.

On the "Review and create" page press the blue "Install cluster" button.

Wait for the installation to finish successfully.

Securely download both the kubeconfig and the kubeadmin password locally.

Update the nerc-ocp-infra/kubeadmin vault secret with the new kubeadmin
password.

### Clear CEPH RBD Pool (if necessary)

If this isn't the first time the RBD pool has been used and it's not empty
you'll need to clear it out first.

Get password from install console, use it to oc login cli

Next get a debug shell on one of the controllers and setup networking manually
to get connectivity to ceph:

```sh
oc debug --as-root=true node/ctl-0
chroot /host
ip link add link ens6f0 name ens6f0.2177 type vlan id 2177
ip addr add 10.30.13.7/24 dev ens6f0.2177
ip link set ens6f0.2177 up
ip route add 10.255.116.0/23 via 10.30.13.1
```

Next setup ceph.conf on that same controller:

```sh
$ oc debug --as-root=true node/ctl-0
$ chroot /host
$ mkdir /root/ceph
$ cd /root/ceph
$ cat << EOF > /root/ceph/ceph.conf
[global]
mon_host = 10.255.116.11
EOF
```

Then setup the ceph client key file:

```sh
$ cat << EOF > /root/ceph/ceph.client.node-nerc-ocp-infra-1-rbd.keyring
[client.node-nerc-ocp-infra-1-rbd]
key = <key>
EOF
```

Run a ceph pod and link in the ceph conf and keyring:

```sh
podman run --rm -it --privileged --network=host -v /root/ceph:/etc/ceph --entrypoint /bin/bash quay.io/ceph/ceph:v14
```

Verify you can see the rbds and debug issues if not:

```sh
rbd ls --id node-nerc-ocp-infra-1-rbd --pool nerc_ocp_infra_1_rbd
```

Remove all RBD images in the pool:

```sh
rbd ls --id node-nerc-ocp-infra-1-rbd --pool nerc_ocp_infra_1_rbd | xargs -tl1 -I % rbd --id node-nerc-ocp-infra-1-rbd rm nerc_ocp_infra_1_rbd/%
```

### Install All Operators and Machine Configs

Next we install all of the operators and machine configs. The operators will be
needed in later steps and the machine configs take a while to apply given that
they cause a reboot of all nodes in the cluster.

(using <https://github.com/ryane/kfilt/releases/tag/v0.0.7>)

Use either the kubeconfig file or the kubeadmin password from the assisted
installer to configure the OpenShift CLI (ie oc or kubectl).

Install all of the operators:

```sh
cd $REPO_ROOT/cluster-scope/overlays/nerc-ocp-infra
kustomize build | kfilt -k namespace | oc apply -f -
kustomize build | kfilt -k operatorgroup,subscription | oc apply -f -
cd $REPO_ROOT/grafana/overlays/nerc-ocp-infra
kustomize build | kfilt -k operatorgroup,subscription | oc apply -f -
```

Wait for all of these operators to finish installing by watching in the
OpenShift console's "Installed Operators" page. They should all reach the
"Succeeded" phase with a green checkmark.

Next apply all of the machine configs for nerc-ocp-infra:

```sh
cd cluster-scope/overlays/nerc-ocp-infra
kustomize build |kfilt -k machineconfig |oc apply -f -
```

This will take a while given that it will reboot each node in the cluster
one-by-one. You can check the status using:

```sh
oc get mcp
```

Update cluster ID in the `OCP-on-NERC/nerc-ocp-config` repo, e.g.:

<https://github.com/OCP-on-NERC/nerc-ocp-config/pull/180>

You can fetch the cluster ID using:

```sh
oc get clusterversion version -o jsonpath='{.spec.clusterID}'
```

## hostpath-provisioner

Next up is the hostpath-provisioner. This is mostly needed for the vault
installation which does not use ceph RBDs in order to avoid a chicken-and-egg
problem with ODF secrets living in vault.

```sh
cd $REPO_ROOT/hostpath-provisioner/overlays/nerc-ocp-infra
kustomize build | oc apply -f -
cd $REPO_ROOT/cluster-scope/overlays/nerc-ocp-infra
kustomize build | kfilt -l nerc.mghpcc.org/bundle=hostpath-provisioner | oc apply -f -
```

## Vault

Now that the hostpath-provisioner is in place we can install vault for storing
secrets:

```sh
cd vault/overlays/nerc-ocp-infra
kustomize build | oc apply -f -
cd $REPO_ROOT/cluster-scope/overlays/nerc-ocp-infra
kustomize build | kfilt -l nerc.mghpcc.org/bundle=vault | oc apply -f -
```

**NOTE**: The vault pods will be in a `Running` state but not yet ready. They
will transition to a ready state after the vault has been initialized

### Initializing Vault

In order to get the vault pods to a ready state we need to initialize and
unseal it.

We perform the following steps based on the following upstream documents:

Initializing a vault cluster:

<https://developer.hashicorp.com/vault/tutorials/raft/raft-storage#join-nodes-to-the-cluster>

Restoring vault secrets from backup:

<https://developer.hashicorp.com/vault/tutorials/standard-procedures/sop-restore>

Create an initial vault cluster on the `nerc-vault-0` pod:

```sh
oc rsh -n vault nerc-vault-0
vault operator init
```

Securely save the unseal keys and root token from this command locally.

Write down the `nerc-vault-0` pod's IP address:

```sh
oc rsh -n vault nerc-vault-0
ip addr  # capture pods 10.x.x.x ip address
```

Unseal the vault using the unseal keys in the previous `vault operator init`
command:

```sh
oc rsh -n vault nerc-vault-0
vault operator unseal
```

hop on nerc-vault-1

```sh
oc rsh -n vault nerc-vault-1
vault operator raft join http://10.129.0.13:8200
vault operator unseal
```

same thing on nerc-vault-2

Now you should be able to see all the cluster nodes from `nerc-vault-0` pod:

```sh
$ oc rsh -n vault nerc-vault-0
$ vault login
# enter root token from earlier run of `$ vault operator init`
$ vault operator raft list-peers
```

### Restoring from a backup snapshot

To restore the vault from a backup snapshot:

```sh
oc cp /path/to/backup.snap nerc-vault-0:/tmp
oc rsh -n vault nerc-vault-0
oc login # enter root token from earlier run of `$ vault operator init`
vault operator raft snapshot restore --force /tmp/backup.snap
```

You will now need to unseal all nodes again with the unseal keys that
correspond to the backup snapshot.

## External Secrets

Once vault is up and running we can install external-secrets to sync secrets
stored in vault into kubernetes secrets.

```sh
cd $REPO_ROOT/cluster-scope/overlays/nerc-ocp-infra
kustomize build | kfilt -l nerc.mghpcc.org/bundle=external-secrets | oc apply -f -
kustomize build | kfilt -l nerc.mghpcc.org/bundle=external-secrets-clustersecretstore | oc apply -f -
```

Verify external secrets are working by applying all of the external secrets
defined for nerc-ocp-infra:

```sh
cd $REPO_ROOT/cluster-scope/overlays/nerc-ocp-infra
kustomize build | kfilt -k externalsecret | oc apply -f -
```

All of the externalsecrets should be in the `SecretSynced` state:

```sh
$ oc get externalsecrets -A -o 'custom-columns=NAME:.metadata.name,STATUS:.status.conditions[0].reason'
NAME                                                                         STATUS
dex-clients                                                                  SecretSynced
oauth-client-secret                                                          SecretSynced
github-group-sync                                                            SecretSynced
open-cluster-management-observability-alertmanager-config                    SecretSynced
open-cluster-management-observability-multiclusterhub-operator-pull-secret   SecretSynced
open-cluster-management-observability-thanos-object-storage                  SecretSynced
default-api-certificate                                                      SecretSynced
github-client-secret                                                         SecretSynced
default-ingress-certificate                                                  SecretSynced
openshift-logging-lokistack-gateway-bearer-token                             SecretSynced
loki-thanos-object-storage                                                   SecretSynced
rook-ceph-external-cluster-details                                           SecretSynced
vault-backup-s3-endpoint                                                     SecretSynced
nerc-cluster-secrets                                                         Valid
```

## NMState

Next we setup the tagged storage vlan interface in order to communicate with
external ceph.

```sh
cd $REPO_ROOT/cluster-scope/overlays/nerc-ocp-infra
kustomize build | kfilt -k nmstate | oc apply -f -
```

Wait a bit for nmstate to become active - check status with `$ oc get -A nmstate`

```sh
kustomize build | kfilt -k nodenetworkconfigurationpolicy | oc apply -f -
```

Check interface status with:

```sh
$ oc get nnce
NAME                   STATUS
ctl-0.vlan-2177-nese   Available
ctl-1.vlan-2177-nese   Available
ctl-2.vlan-2177-nese   Progressing
```

Once these are all 'available' we can continue with the ODF install

## ODF

First label the nodes as openshift-storage cluster nodes:

```sh
oc label node ctl-0 cluster.ocs.openshift.io/openshift-storage=""
oc label node ctl-1 cluster.ocs.openshift.io/openshift-storage=""
oc label node ctl-2 cluster.ocs.openshift.io/openshift-storage=""
```

Next install the fake metrics server:

```sh
cd $REPO_ROOT/fake-metrics-server/overlays/nerc-ocp-infra
kustomize build | oc apply -f -
```

Then install ODF external:

```sh
cd $REPO_ROOT/cluster-scope/overlays/nerc-ocp-infra
kustomize build | kfilt -l nerc.mghpcc.org/bundle=odf-external | oc apply -f -
```

Check pod status in the `openshift-storage` namespace:

```sh
oc get pods -n openshift-storage
```

Look for issues with any pods not initializing. Currently there is a known
issue with the rook-ceph-tools pod not coming up:

<https://github.com/OCP-on-NERC/operations/issues/84>

For now we ignore this issue given that this is a nice-to-have pod for
interacting with ceph via CLI tools and is not in the critical path of the
build.

Once nooba comes up it will request a PVC on RBD storage. If those PVCs are
bound then we know the storage is working:

```sh
$ oc get pvc -n openshift-storage
NAME                                               STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS                           AGE
db-noobaa-db-pg-0                                  Bound    pvc-113af352-3f02-4503-8987-8fcc28e4ba62   50Gi       RWO            ocs-external-storagecluster-ceph-rbd   80m
metrics-backing-store-noobaa-pvc-2e5aa4bf          Bound    pvc-82ea1bdd-2994-4917-9494-a4c98d8961d8   400Gi      RWO            ocs-external-storagecluster-ceph-rbd   33m
metrics-backing-store-noobaa-pvc-2e5e5e59          Bound    pvc-1a4c286e-358e-4c02-8389-4cd1c2bd51b0   400Gi      RWO            ocs-external-storagecluster-ceph-rbd   33m
metrics-backing-store-noobaa-pvc-d997deee          Bound    pvc-cf56ca68-c405-4b68-8afd-c54ff2f9d979   400Gi      RWO            ocs-external-storagecluster-ceph-rbd   33m
noobaa-default-backing-store-noobaa-pvc-c5746e03   Bound    pvc-1309acd3-a87d-4ec0-9c72-3ac4407e2948   50Gi       RWO            ocs-external-storagecluster-ceph-rbd   69m
```

At this point we should have `ocs-external-storagecluster-ceph-rbd` as the
default storage class:

```sh
$ oc get storageclasses | grep -i default
ocs-external-storagecluster-ceph-rbd (default)   openshift-storage.rbd.csi.ceph.com   Delete          Immediate              true                   90m
```

Now we can delete the `vault-snapshots` PVC that was created during the prior
vault installation. It will get recreated with the default RBD storage class:

```sh
$ oc delete -n vault pvc vault-snapshots
persistentvolumeclaim "vault-snapshots" deleted
$ oc get -n vault pvc vault-snapshots
NAME              STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS                           AGE
vault-snapshots   Bound    pvc-68b8859f-3f67-4b2a-84c3-71536f0029ae   10Gi       RWO            ocs-external-storagecluster-ceph-rbd   18s
```

## ArgoCD

To install argocd:

```sh
cd $REPO_ROOT/openshift-gitops/overlays/nerc-ocp-infra
kustomize build | oc apply -f -
cd $REPO_ROOT/cluster-scope/overlays/nerc-ocp-infra
kustomize build | kfilt -l nerc.mghpcc.org/bundle=openshift-gitops | oc apply -f -
```

Once argocd is up login via kubeadmin and change cluster name to
`nerc-ocp-infra` (Settings -> Cluster -> `https://kubernetes.default.svc`). There
will be cert errors accessing the site which is expected at this point in the
install. A proper ingress cert will be installed later.

In the applications page of argocd you should see an applicationset defined
called `nerc-ocp-infra-aoa` that will be out-of-sync. By design, this
application requires a manual sync operation. Syncing this applicationset will
pull in the other apps defined for `nerc-ocp-infra`.

## Observability

Update the secrets in vault for observability: <https://vault-ui-vault.apps.nerc-ocp-infra.rc.fas.harvard.edu/ui/vault/secrets/nerc/show/nerc-ocp-infra/open-cluster-management-observability-thanos-object-storage>

... using these values:

```sh
oc --as system:admin -n open-cluster-management-observability get configmap/observability -o jsonpath='{.data.BUCKET_NAME}'
oc --as system:admin -n open-cluster-management-observability get secret/observability -o jsonpath='{.data.AWS_ACCESS_KEY_ID}' | base64 -d
oc --as system:admin -n open-cluster-management-observability get secret/observability -o jsonpath='{.data.AWS_SECRET_ACCESS_KEY}' | base64 -d
```

## Loki

Update the secret in vault for loki:
<https://vault-ui-vault.apps.nerc-ocp-infra.rc.fas.harvard.edu/ui/vault/secrets/nerc/show/nerc-ocp-infra/loki-thanos-object-storage>

... using these values:

```sh
oc --as system:admin -n openshift-operators-redhat get configmap/openshift-operators-redhat-objectbucketclaim -o jsonpath='{.data.BUCKET_NAME}'
oc --as system:admin -n openshift-operators-redhat get secret/openshift-operators-redhat-objectbucketclaim -o jsonpath='{.data.AWS_ACCESS_KEY_ID}' | base64 -d
oc --as system:admin -n openshift-operators-redhat get secret/openshift-operators-redhat-objectbucketclaim -o jsonpath='{.data.AWS_SECRET_ACCESS_KEY}' | base64 -d
```

Update the Lokistack token for the infra cluster

If you have done a cluster rebuild, the token used by the openshift-logging
collectors will be incorrect. You will likely see errors like this in the
collector pods of the openshift-logging namespace:

```sh
2023-01-11 16:31:45 +0000 [warn]: [loki_infra] failed to write post to https://lokistack-openshift-operators-redhat.apps.nerc-ocp-infra.rc.fas.harvard.edu/api/logs/v1/infrastructure/loki/api/v1/push (302 Found {"error":"failed to find token","errorType":"observatorium-api","status":"error"}
)
```

Log into the infra cluster in the terminal. Try finding the openshift-logging
logcollector token like this:

```sh
oc --as system:admin -n openshift-logging get secret/logcollector-token -o jsonpath="{.data.token}" | base64 -d
```

With the new token value, update the token value in the vault here:

<https://vault-ui-vault.apps.nerc-ocp-infra.rc.fas.harvard.edu/ui/vault/secrets/nerc/show/nerc-ocp-infra/lokistack-gateway-bearer-token>

Delete the collector pods in the infra cluster:

```sh
oc --as system:admin -n openshift-logging delete $(oc --as system:admin -n openshift-logging get pod -o name | tr '\n' ' ')
```

Update the Lokistack token for the prod cluster

With the same token value from the infra cluster, update the token value in the
vault here:

<https://vault-ui-vault.apps.nerc-ocp-infra.rc.fas.harvard.edu/ui/vault/secrets/nerc/show/nerc-ocp-prod/lokistack-gateway-bearer-token>

Delete the collector pods in the prod cluster:

```sh
oc --as system:admin -n openshift-logging delete $(oc --as system:admin -n openshift-logging get pod -o name | tr '\n' ' ')
```

## ACM

Manually detach the production cluster from the prior ACM installation
(if needed) by running the following on `nerc-ocp-prod`:

```sh
# ON THE PROD CLUSTER:
oc delete klusterlet klusterlet --as=system:admin
oc delete crd klusterlets.operator.open-cluster-management.io --as=system:admin
```

Navigate to the multicloud dashboard as kubeadmin:

<https://console-openshift-console.apps.nerc-ocp-infra.rc.fas.harvard.edu/multicloud/home/welcome>

Click on "Clusters" and then click on the `nerc-ocp-prod` cluster and click the
"Copy command" button. Save the copied command locally to a shell script.

**Switch your kubectl context to point to nerc-ocp-prod**.

If you're currently logged in with kubeadmin credentials via the CLI you should
be able to run the shell script without modification. If you're logged in as
a cluster admin (e.g. via github) you will need to make some edits:

Find the two `kubectl` commands (create and apply) and append
`--as=system:admin` in order for them to work. Save the script and run it to
configure the prod cluster to be imported to ACM. This is needed due to the
sudoers configuration for cluster admins on the prod cluster.

The cluster should flip from "pending-import" to "Ready" status within a few
moments if successful.

## Troubleshooting

Some steps that might be required if there are issues.

### openshift-authentication

It might be necessary to delete the `github-client-sceret` in the
`openshift-config` namespace in order to resync the secret from vault which
will restart the oauth pods

```sh
oc delete -n openshift-config secret github-client-secret
```
