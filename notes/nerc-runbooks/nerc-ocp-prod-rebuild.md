# Procedure Overview

1. [Disconnecting from ACM](#disconnecting-from-acm)
1. [Bootstrapping nerc-ocp-prod](#bootstrapping-nerc-ocp-prod)
1. [APIServer](#apiserver)
1. [External Secrets](#external-secrets)
1. [NMState](#nmstate)
1. [Open Data Foundations (ODF)](#odf)
1. [Advanced Cluster Management (ACM)](#acm)
1. [ArgoCD](#argocd)
1. [Loki](#loki)
1. [Troubleshooting](#troubleshooting)

## Disconnecting from ACM

If the cluster is connected to ACM on the infra cluster and a rebuild of the
infra cluster is not planned, you should first cleanly detach the cluster from
ACM.

Navigate to the clusters page of ACM as the `kubeadmin` user:

<https://console-openshift-console.apps.nerc-ocp-infra.rc.fas.harvard.edu/multicloud/infrastructure/clusters/managed>

Click on the `nerc-ocp-prod` cluster in the list of clusters.

Go to Actions -> Detach cluster. Confirm you want to detach the cluster by
typing `nerc-ocp-prod` in the "Detach clusters?" dialog and press the "Detach"
button. You should see a message that the cluster is being detached and that
the process might take a while.

If this completes successfully the cluster should switch to state "Pending
import". Running the following query against the prod cluster should give the
following error:

```sh
$ oc get klusterlets -A --as=system:admin
error: the server doesn't have a resource type "klusterlets"
```

Proceed to the next section to bootstrap the initial OCP install.

## Bootstrapping nerc-ocp-prod

### Assisted installer

Login to the redhat console: <https://console.redhat.com/openshift>

Click the blue "Create cluster" button then select the "Datacenter" tab. Under
the "Assisted Installer" section in the datacenter tab, click the blue "Create
cluster" button.

Fill in the cluster details:

```yaml
name: shift
base domain: nerc.mghpcc.org
version: 4.10.x
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
[root@nerc-bootstrap ~]# mkdir /srv/pxe/httpboot/openshift-discovery-iso/shift/$OPENSHIFT_VERSION_HERE
[root@nerc-bootstrap ~]# cd /srv/pxe/httpboot/openshift-discovery-iso/shift/$OPENSHIFT_VERSION_HERE
[root@nerc-bootstrap 4.10.x]# wget -O discovery_image_nerc-ocp-prod.iso '$DISCOVERY_ISO_URL_HERE'
[root@nerc-bootstrap 4.10.x]# bash /srv/repos/boot-openshift-ai/extract-iso.sh discovery_image_nerc-ocp-prod.iso
config.ign
18 blocks
[root@nerc-bootstrap 4.10.x]# ls
config.ign  discovery_image_nerc-ocp-prod.iso  images
```

Submit a merge request to nerc-ansible repo to set `ocp_version` for the
`nerc-ocp-prod` hosts to `$OPENSHIFT_VERSION_HERE`. Here's an example merge
request where this was done previously:

<https://gitlab-int.rc.fas.harvard.edu/nerc/nerc-ansible/-/merge_requests/56>

Next PXE boot the nerc-ocp-prod hosts via OBM:

```sh
$ read -s IPMI_PASSWORD
(enter IPMI password)
$ export IPMI_PASSWORD
$ cat << 'EOF' > /tmp/pxe.sh
#!/bin/bash
function pxe_host() {
  set -x
  ipmitool -I lanplus -E -U root -H "${1}-obm.nerc-ocp-prod.rc.fas.harvard.edu" chassis bootdev pxe
  ipmitool -I lanplus -E -U root -H "${1}-obm.nerc-ocp-prod.rc.fas.harvard.edu" chassis power reset
  set +x
}
HOSTS="\
ctl-0
ctl-1
ctl-2
wrk-10
wrk-11
wrk-13
wrk-14
wrk-16
wrk-17
wrk-18
wrk-7
wrk-8
wrk-9
"

for host in $HOSTS; do
  pxe_host $host
done
EOF
$ bash /tmp/pxe.sh
```

Wait for all hosts to show up in the "Add hosts" page. Change the 3x controller
hosts (ie hostname `ctl-XXX`)"Role" to "Control plane node" and the rest of the
nodes (ie hostname `wrk-XXX`) to "Worker". Press the Next button.

Leave the defaults on the "Storage" page and press Next.

On the "Network" page leave the defaults and fill in the API and Ingress IPs
with the following addresses:

```yaml
api_ip: 10.30.6.5
ingress_ip: 10.30.6.6
```

These come from from DNS, e.g.:

```sh
nslookup api.nerc-ocp-prod.rc.fas.harvard.edu
nslookup lb.nerc-ocp-prod.rc.fas.harvard.edu
```

Press the "Next" button.

On the "Review and create" page press the blue "Install cluster" button.

Wait for the installation to finish successfully.

**NOTE**: Data point: With 13 hosts this takes about ~40 minutes.

Securely download and store both the kubeconfig and the kubeadmin password
locally.

Update the nerc-ocp-prod/kubeadmin vault secret with the new kubeadmin
password and kubeconfig.

Update the support subscription settings by clicking on the `nerc-ocp-prod`
cluster in the assisted installer and then click the "Actions" drop down from
the top right corner of the page and go to "Edit Subscription settings". In the
subscription settings dialog set the following:

- SLA -> Standard
- Support type -> Red Hat support (L1-L3)
- Cluster usage -> Production
- Subscription units -> Cores/vCPUs

Click "Save" to save the new subscription settings.

Next go to the support tab on the `nerc-ocp-prod` cluster page in the assisted
installer and click the "Add notification contact". Add all NERC admins that
should be contacted in the event of notifications about the cluster.

### Clear CEPH RBD Pool (if necessary)

If this isn't the first time the RBD pool has been used and it's not empty
you'll need to clear it out first.

Get password from install console, use it to oc login cli

Next get a debug shell on one of the controllers and setup networking manually
to get connectivity to ceph:

```sh
oc debug --as-root=true node/ctl-0
chroot /host
ip link add link ens5f0 name ens5f0.2173 type vlan id 2173
ip addr add 10.30.10.7/24 dev ens5f0.2173
ip link set ens5f0.2173 up
ip route add 10.255.116.0/23 via 10.30.10.1
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
$ cat << EOF > /root/ceph/ceph.client.node-nerc-ocp-prod-1-rbd.keyring
[client.node-nerc-ocp-prod-1-rbd]
key = <key>
EOF
```

The key for `client.node-nerc-ocp-prod-1-rbd` is nested in a vault secret:

<https://vault-ui-vault.apps.nerc-ocp-infra.rc.fas.harvard.edu/ui/vault/secrets/nerc/show/nerc-ocp-prod/openshift-storage/rook-ceph-external-cluster-details>

Copy the `userKey` from the `rook-csi-rbd-node` mapping.

Run a ceph pod and link in the ceph conf and keyring:

```sh
podman run --rm -it --privileged --network=host -v /root/ceph:/etc/ceph --entrypoint /bin/bash quay.io/ceph/ceph:v14
```

Verify you can see the rbds and debug issues if not:

```sh
rbd ls --id node-nerc-ocp-prod-1-rbd --pool nerc_ocp_prod_1_rbd
```

Remove all RBD images in the pool:

```sh
rbd ls --id node-nerc-ocp-prod-1-rbd --pool nerc_ocp_prod_1_rbd | xargs -tl1 -I % rbd --id node-nerc-ocp-prod-1-rbd rm nerc_ocp_prod_1_rbd/%
```

Verify the RBD pool is now empty:

```sh
$ rbd ls --id node-nerc-ocp-prod-1-rbd --pool nerc_ocp_prod_1_rbd
$
```

Exit the podman ceph container and clean up the `/root/ceph` directory:

```sh
exit  # get out of podman container
rm -rf /root/ceph  # from the debug pod
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
cd $REPO_ROOT/cluster-scope/overlays/nerc-ocp-prod
kustomize build | kfilt -k namespace | oc apply -f -
kustomize build | kfilt -k operatorgroup,subscription | oc apply -f -
```

Wait for all of these operators to finish installing by watching in the
OpenShift console's "Installed Operators" page. They should all reach the
"Succeeded" phase with a green checkmark.

Next apply all of the machine configs for nerc-ocp-prod:

```sh
cd $REPO_ROOT/cluster-scope/overlays/nerc-ocp-prod
kustomize build | kfilt -k machineconfig | oc apply -f -
```

This will take a while given that it will reboot each node in the cluster
one-by-one. You can check the status using:

```sh
oc get mcp
```

You can see which node in the machineconfig pool is currently being updated by
checking which node currently has scheduling disabled:

```sh
$ oc get nodes | grep -i schedulingdisabled
wrk-10   Ready,SchedulingDisabled   worker   80m   v1.23.12+8a6bfe4
```

**NOTE**: If a node takes an extremely long time to finish and return back to
`Ready` state you might need to manually intervene and reboot the host via
IPMI/OBM.

Update cluster ID in the `OCP-on-NERC/nerc-ocp-config` repo, e.g.:

<https://github.com/OCP-on-NERC/nerc-ocp-config/pull/180>

You can fetch the cluster ID using:

```sh
oc get clusterversion version -o jsonpath='{.spec.clusterID}'
```

## APIServer

In order for vault to successfully authenticate kubernetes service tokens from
the prod cluster, we need to configure a valid cert on the prod cluster's API
server. In order to do that, we'll need to manually extract the AWS route53
credentials from vault and create a secret in the `openshift-config` and
`openshift-ingress` namespaces.

Securely save the `aws-route53-credentials` secret's `AWS_ACCESS_KEY_ID` and
`AWS_SECRET_ACCESS_KEY` fields into files `aws_access_key_id.txt` and
`aws_secret_access_key.txt` respectively:

<https://vault-ui-vault.apps.nerc-ocp-infra.rc.fas.harvard.edu/ui/vault/secrets/nerc/show/nerc-ocp-prod/aws-route53-credentials>

Next create the `aws-route53-credentials` kubernetes secrets on the prod
cluster:

```sh
oc create secret generic -n openshift-config \
  --from-file=AWS_ACCESS_KEY_ID=/path/to/aws-access-key-id.txt \
  --from-file=AWS_SECRET_ACCESS_KEY=/path/to/aws-secret-access-key.txt \
  aws-route53-credentials

oc create secret generic -n openshift-ingress \
  --from-file=AWS_ACCESS_KEY_ID=/path/to/aws-access-key-id.txt \
  --from-file=AWS_SECRET_ACCESS_KEY=/path/to/aws-secret-access-key.txt \
  aws-route53-credentials
```

Once those secrets are in place we can apply the issuers and certificates:

```sh
$ cd $REPO_ROOT/cluster-scope/overlays/nerc-ocp-prod
$ kustomize build | kfilt -k issuer,certificate | oc apply -f -
certificate.cert-manager.io/default-api-certificate created
certificate.cert-manager.io/default-ingress-certificate created
issuer.cert-manager.io/letsencrypt-production-dns01 created
issuer.cert-manager.io/letsencrypt-staging-dns01 created
issuer.cert-manager.io/letsencrypt-production-dns01 created
issuer.cert-manager.io/letsencrypt-staging-dns01 created
```

Wait for the `default-api-certificate` `Certificate` resource to be in a
`Ready` state:

```sh
$ oc get certificates -n openshift-config default-api-certificate | grep -i true
default-api-certificate   True    default-api-certificate   53s
```

Once the `default-api-certificate` is in a `Ready` state we can now update the
`APIServer` resource to use the new letsencrypt cert:

```sh
cd $REPO_ROOT/cluster-scope/overlays/nerc-ocp-prod
kustomize build | kfilt -k apiserver | oc apply -f -
```

**NOTE**: If you're using the kubeconfig from the assisted installer to connect
to the prod cluster, you may encounter cert errors after this command due to
the API server's cert being updated and the fact that the previous self-signed
CA cert is included in the config. You'll need to remove the
`certificate-authority-data` key from the kubeconfig at this point in order to
run commands.

The `kube-apiserver` cluster operator will now be in a `PROGRESSING=True` state:

```sh
$ oc get clusteroperators kube-apiserver
NAME             VERSION   AVAILABLE   PROGRESSING   DEGRADED   SINCE   MESSAGE
kube-apiserver   4.10.XX   True        True          False      24h     NodeInstallerProgressing: 2 nodes are at revision 10; 1 nodes are at revision 11
```

Keep watching this command until `PROGRESSING=False`, `DEGRADED=False`, and
`AVAILABLE=True`.

Once the `kube-apiserver` cluster operator has finished updating you should be
able to curl the API address without cert errors:

```sh
$ curl -I https://api.nerc-ocp-prod.rc.fas.harvard.edu:6443
HTTP/2 403
audit-id: eb8a6b6b-8a40-4a91-b7de-e56318a9d329
cache-control: no-cache, private
content-type: application/json
x-content-type-options: nosniff
x-kubernetes-pf-flowschema-uid: 9f843d9e-bdb5-43eb-a6fe-c8f33afe0dfe
x-kubernetes-pf-prioritylevel-uid: 79874392-548e-4536-9d9b-4f7da9800cc7
content-length: 218
date: Thu, 26 Jan 2023 16:16:51 GMT
```

## External Secrets

Next we setup external-secrets to sync secrets stored in vault into kubernetes
secrets.

```sh
cd $REPO_ROOT/cluster-scope/overlays/nerc-ocp-prod
kustomize build | kfilt -l nerc.mghpcc.org/bundle=external-secrets | oc apply -f -
kustomize build | kfilt -k serviceaccount | kfilt -n vault-secret-reader | oc apply -f -
kustomize build | kfilt -k secret | kfilt -n vault-secret-reader | oc apply -f -
```

Fetch the `eso-vault-auth` service account token:

```sh
SECRET_NAME=$(oc get sa -n external-secrets-operator eso-vault-auth -o jsonpath="{.secrets[0].name}")
oc get secret -n external-secrets-operator $SECRET_NAME -o jsonpath='{.data.token}' | base64 -d
```

Copy and paste the token into the vault kubernetes auth config for
`nerc-ocp-prod` under `Token Reviewer JWT`:

<https://vault-ui-vault.apps.nerc-ocp-infra.rc.fas.harvard.edu/ui/vault/settings/auth/configure/kubernetes%2Fnerc-ocp-prod/configuration>

Save the updated vault kubernetes configuration for `nerc-ocp-prod`.

Once that's in place we can create the secret stores:

```sh
cd $REPO_ROOT/cluster-scope/overlays/nerc-ocp-prod
kustomize build | kfilt -k secretstore | oc apply -f -
```

Check that all secret stores are in a `Valid` state:

```sh
$ oc get secretstores -A
NAMESPACE             NAME                AGE     STATUS   CAPABILITIES   READY
acct-mgt              nerc-secret-store   9m13s   Valid    ReadWrite      True
group-sync-operator   nerc-secret-store   9m13s   Valid    ReadWrite      True
openshift-config      nerc-secret-store   9m13s   Valid    ReadWrite      True
openshift-ingress     nerc-secret-store   9m13s   Valid    ReadWrite      True
openshift-logging     nerc-secret-store   9m13s   Valid    ReadWrite      True
openshift-storage     nerc-secret-store   9m13s   Valid    ReadWrite      True`
```

Next we apply all externalsecrets in the repo:

```sh
cd $REPO_ROOT/cluster-scope/overlays/nerc-ocp-prod
kustomize build | kfilt -k externalsecret | oc apply -f -
```

All of the externalsecrets should be in the `SecretSynced` state:

```sh
$ oc get externalsecrets -A -o 'custom-columns=NAME:.metadata.name,STATUS:.status.conditions[0].reason'
NAME                                 STATUS
nerc-secret-store                    Valid
nerc-secret-store                    Valid
nerc-secret-store                    Valid
nerc-secret-store                    Valid
nerc-secret-store                    Valid
nerc-secret-store                    Valid
github-group-sync                    SecretSynced
default-api-certificate              SecretSynced
github-client-secret                 SecretSynced
oauths-clientsecret-nerc             SecretSynced
default-ingress-certificate          SecretSynced
nerc-route53-creds                   SecretSynced
rook-ceph-external-cluster-details   SecretSynced
```

## NMState

Next we setup the tagged storage vlan interface in order to communicate with
external ceph.

Occassionally hosts will not correctly configure the `bond0` device on first
boot after the machineconfig that configures the device has been applied. To
prevent issues with NMState failing in this case we first check that `bond0` is
in the route table and if not reboot the host, otherwise, NMState will fail to
apply:

```sh
oc get nodes -l node-role.kubernetes.io/worker -o name | xargs -I {} -n 1 oc debug --as-root=true {} -- bash -c 'ip route | grep -qi bond0 || chroot /host systemctl reboot'
```

If nodes rebooted, watch the output of `$ oc get nodes` until all nodes are
available and then run the command again to ensure bond0 is correctly
configured.

Once we've verified `bond0` is active on all hosts, we can move on to creating
the initial NMState resource:

```sh
cd $REPO_ROOT/cluster-scope/overlays/nerc-ocp-prod
kustomize build | kfilt -l nerc.mghpcc.org/bundle=nmstate | oc apply -f -
```

Wait a bit for nmstate to become active - check status with `$ oc get -A nmstate`

Once the nmstate resource is active we can apply the node network configuration
policies that will configure both storage and external ingress vlan device:

```sh
kustomize build | kfilt -k nodenetworkconfigurationpolicy | oc apply -f -
```

Check interface status with:

```sh
$ oc get nnce
NAME                   STATUS
wrk-10.vlan-2173-nese   Available
wrk-11.vlan-2173-nese   Available
wrk-13.vlan-2173-nese   Progressing
wrk-14.vlan-2173-nese   Available
wrk-16.vlan-2173-nese   Available
wrk-17.vlan-2173-nese   Available
wrk-18.vlan-2173-nese   Available
wrk-7.vlan-2173-nese    Progressing
wrk-8.vlan-2173-nese    Available
wrk-9.vlan-2173-nese    Available
...
```

Once these are all `Available` we can continue with the ODF install.

**NOTE**: If any `nnce` resources are in a failed, degraded, etc. state you'll
need to use `$ oc describe nnce <nnce_resource>` to get more details and debug
the issue. Once you think you've resolved the issue you can delete the nnce
resources and they should come back automatically although a reboot of hosts
might be required.

## ODF

First we setup the patch operator and apply a patch to label all of the worker
nodes as openshift-storage cluster nodes:

```sh
cd $REPO_ROOT/cluster-scope/overlays/nerc-ocp-prod
kustomize build | kfilt -l nerc.mghpcc.org/bundle=patch-operator | oc apply -f -
kustomize build | kfilt -k clusterrole | kfilt -n node-labeler | oc apply -f -
kustomize build | kfilt -k serviceaccount,clusterrolebinding | kfilt -l nerc.mghpcc.org/feature=odf | oc apply -f -
kustomize build | kfilt -k patch | kfilt -l nerc.mghpcc.org/feature=odf | oc apply -f -
```

Check that all worker nodes now have the storage label:

```sh
WORKERS=$(oc get nodes -l node-role.kubernetes.io/worker -o name)
STORAGE_NODES=$(oc get nodes -l cluster.ocs.openshift.io/openshift-storage -o name)
[ "$WORKERS" = "$STORAGE_NODES" ] || echo "not all workers have storage label"
```

Next install the fake metrics server:

```sh
cd $REPO_ROOT/fake-metrics-server/overlays/nerc-ocp-prod
kustomize build | oc apply -f -
```

Check that the fake metrics server is ready and available:

```sh
$ oc get deployment -n openshift-storage fake-metrics-exporter
NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
fake-metrics-exporter   1/1     1            1           33s
```

Then install ODF external:

```sh
cd $REPO_ROOT/cluster-scope/overlays/nerc-ocp-prod
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
➜ oc get pvc -n openshift-storage
NAME                                               STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS                           AGE
db-noobaa-db-pg-0                                  Bound    pvc-428bc574-f818-4ca0-93c8-b26bf9ac2280   50Gi       RWO            ocs-external-storagecluster-ceph-rbd   11m
noobaa-default-backing-store-noobaa-pvc-efb33737   Bound    pvc-070de51a-6203-4d2a-aeb5-b9e98e52a0d1   50Gi       RWO            ocs-external-storagecluster-ceph-rbd   9m8s
```

At this point we should have `ocs-external-storagecluster-ceph-rbd` as the
default storage class:

```sh
$ oc get storageclasses | grep -i default
ocs-external-storagecluster-ceph-rbd (default)   openshift-storage.rbd.csi.ceph.com   Delete          Immediate              true                   90m
```

## ACM

Next we need to import the cluster to ACM so that it will be added as a cluster
within ArgoCD.

Navigate to the multicloud console plugin as the `kubeadmin` user:

<https://console-openshift-console.apps.nerc-ocp-infra.rc.fas.harvard.edu/multicloud/home/welcome>

Click on "Clusters" and then click on the `nerc-ocp-prod` cluster and click the
"Copy command" button. Save the copied command locally to a shell script.

**Switch your kubectl context to point to nerc-ocp-prod**.

If you're currently logged in with kubeadmin credentials via the CLI you should
be able to run the shell script without modification. If you're logged in as
a cluster admin (e.g. via github) you will need to make some edits:

Find the two `kubectl` commands (create and apply) and append
`--as=system:admin` and save the file. This is needed due to the sudoers
configuration for cluster admins on the prod cluster.

Run the script to install the open cluster management agent on the
`nerc-ocp-prod` cluster:

```sh
$ bash /tmp/acm.sh
customresourcedefinition.apiextensions.k8s.io/klusterlets.operator.open-cluster-management.io created
namespace/open-cluster-management-agent created
serviceaccount/klusterlet created
clusterrole.rbac.authorization.k8s.io/klusterlet created
clusterrole.rbac.authorization.k8s.io/open-cluster-management:klusterlet-admin-aggregate-clusterrole created
clusterrolebinding.rbac.authorization.k8s.io/klusterlet created
deployment.apps/klusterlet created
secret/bootstrap-hub-kubeconfig created
klusterlet.operator.open-cluster-management.io/klusterlet created
```

You should now see a klusterlet and deployment resource on `nerc-ocp-prod`:

```sh
$ oc get klusterlets -A
NAME         AGE
klusterlet   83s
$ oc get deployment/klusterlet -n open-cluster-management-agent
NAME         READY   UP-TO-DATE   AVAILABLE   AGE
klusterlet   1/1     1            1           2m35s
```

The cluster should flip from "pending-import" to "Ready" status in the ACM
clusters page within a few moments if successful.

## ArgoCD

After the cluster has been imported to ACM it should show up as a cluster
within ArgoCD. You can check that's working by navigating to the ArgoCD
clusters page and confirming `nerc-ocp-prod` is in the list:

<https://openshift-gitops-server-openshift-gitops.apps.nerc-ocp-infra.rc.fas.harvard.edu/settings/clusters>

Additionally, a new applicationset called `nerc-ocp-prod-aoa` should be
present now. This app will be out-of-sync and requires a manual sync by design.
Syncing this applicationset will pull in all other apps defined for the
`nerc-ocp-prod` cluster.

Force a manual sync of the `nerc-ocp-prod` cluster by pressing the grey "SYNC"
button at the top of the `nerc-ocp-prod-aoa` page:

**NOTE**: You'll need to login as `kubeadmin` otherwise you'll get permission
denied when attempting to sync

<https://openshift-gitops-server-openshift-gitops.apps.nerc-ocp-infra.rc.fas.harvard.edu/applications/nerc-ocp-prod-aoa>

## Loki

Next we check that the logging pods have been configured with the correct token.

Fetch the openshift-logging log collector token from the infra cluster and
compute its signature with md5sum:

```sh
# CONTEXT SET TO NERC-OCP-INFRA
$ oc --as system:admin -n openshift-logging get secret/logcollector-token -o jsonpath="{.data.token}" | base64 -d | md5sum
1c89b41ebde79db520054080d5ee2d0a
```

Next we check whether the collector pods on the prod cluster are using the same token:

```sh
# CONTEXT SET TO NERC-OCP-PROD
$ oc exec -n openshift-logging svc/collector -c collector -- md5sum /var/run/ocp-collector/secrets/lokistack-gateway-bearer-token/token
1c89b41ebde79db520054080d5ee2d0a  /var/run/ocp-collector/secrets/lokistack-gateway-bearer-token/token
```

If the md5sum output doesn't match, you'll need to verify the new token has
been synced by external-secrets on the `nerc-ocp-prod` cluster and that its
md5sum matches the same token on the infra cluster:

```sh
# CONTEXT SET TO NERC-OCP-PROD
$ oc get secrets -n openshift-logging lokistack-gateway-bearer-token -o jsonpath='{.data.token}' | base64 -d | md5sum
1c89b41ebde79db520054080d5ee2d0a
```

If they don't match check that all external-secrets have synced successfully on
the `nerc-ocp-prod` cluster. If external-secrets have all synced and they still
don't match you'll need to update the token field in the vault credentials to
match the infra logcollector-token:

<https://vault-ui-vault.apps.nerc-ocp-infra.rc.fas.harvard.edu/ui/vault/secrets/nerc/show/nerc-ocp-prod/lokistack-gateway-bearer-token>

If you had to fix the logging token secret, you'll need to delete the collector
pods in the `nerc-ocp-prod` cluster to reload them with the new log collector
token:

```sh
# CONTEXT SET TO NERC-OCP-PROD
$ oc --as system:admin -n openshift-logging delete $(oc --as system:admin -n openshift-logging get pod -o name | tr '\n' ' ')
```

## Troubleshooting

Some steps that might be required if there are issues.

### openshift-authentication

It might be necessary to delete the `github-client-sceret` in the
`openshift-config` namespace in order to resync the secret from vault which
will restart the oauth pods

```sh
oc delete -n openshift-config secret github-client-secret
```
