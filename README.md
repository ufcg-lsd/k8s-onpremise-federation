# Kubernetes Federation for on-premises clusters

If you don't know the basics of Kubernetes Federation, you can take a look at this [doc](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/federation.md).

### Scenario

This setup is for on-premise clusters, without being attached to any cloud provider.
This is not suitable for production.

|   Clusters   |  API Server      |  Region    | Zone   |
|:------------:|:----------------:|:----------:|:------:|  
| fcp-cluster  | 10.11.12.1:6443  | europe     | fr     | 
| usa-cluster  | 10.11.12.2:6443  | america    | us     |
| chn-cluster  | 10.11.12.3:6443  | asia       | cn     |

The fields Region and Zone will be used further on, for now, they represent that clusters can be geo distributed.

There are three clusters created using [kubeadm](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/) tool 
- Kubernetes version: 1.7.3
- OS (cluster's nodes): Ubuntu 16.04.2 LTS
- Authorization Mode: RBAC

### Objectives

Deploy a Federation Control Plane (FCP) on `fcp-cluster` and to join the other two clusters to Federation. 

There are some necessary steps to do it:
1. [Get the credentials of the clusters](#get_credentials)
1. [Deploy keepalived to exopose services with LoadBalancer type](#deploy_keepalived)
1. [Add the `region` and `zone` labels for each nodes](#label_nodes)
1. [Deploy etcd-operator as backend of CoreDNS](#deploy_etcd_operator)
1. [Deploy the CoreDNS as DNS provider](#deploy_coredns)
1. [Initialize your FCP](#initialize_fcp)
1. [Join the clusters](#join_clusters)

### Tools

This setup requires the kubefed, the main tool to initialize your Federation Control Plane and join/unjoin the clusters to Federation. 
<!-- 1. [Helm](https://github.com/kubernetes/helm), the Kubernetes Package Manager. [Install docs](https://docs.helm.sh/using_helm/#installing-helm). 
```bash
Getting Helm

$ curl https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get > get_helm.sh
$ chmod 700 get_helm.sh
$ ./get_helm.sh
``` -->

```bash
# Getting Kubefed
# Linux
$ curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/kubernetes-client-linux-amd64.tar.gz
$ tar -xzvf kubernetes-client-linux-amd64.tar.gz

# OS X
$ curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/kubernetes-client-darwin-amd64.tar.gz
$ tar -xzvf kubernetes-client-darwin-amd64.tar.gz

# Windows
$ curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/kubernetes-client-windows-amd64.tar.gz
$ tar -xzvf kubernetes-client-windows-amd64.tar.gz
```

> Note:
> - You can download it to other architectures than amd64 in the [release page](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG.md#client-binaries-1).

```bash
# Copy the extracted binaries to */usr/local/bin* path and set the executable permission to them.
cp kubernetes/client/bin/kubefed /usr/local/bin
chmod +x /usr/local/bin/kubefed
cp kubernetes/client/bin/kubectl /usr/local/bin
chmod +x /usr/local/bin/kubectl
```

### <a name="get_credentials"></a> Get the credentials

There is supposed you have multiple clusters and you should [authenticate across them with kubeconfig](https://kubernetes.io/docs/tasks/access-application-cluster/authenticate-across-clusters-kubeconfig/)

Basically, each one cluster have a kubeconfig file (by default `$HOME/.kube/config` on master node) and you should merge this files to access your clusters from a single point.

A pair of information composed by "Where your API server is running" and the auth info to access this API is called context.
See `kubectl config --help`.

```bash
$ kubectl config get-contexts
CURRENT   NAME      CLUSTER       AUTHINFO    NAMESPACE
          usa       usa-cluster   usa-admin   
*         fcp       fcp-cluster   fcp-admin  
          chn       chn-cluster   chn-admin
```

As an example, this scenario has the following kubeconfig: 

```yaml
$ kubectl config view
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: REDACTED
    server: https://10.11.12.2:6443
  name: usa-cluster
- cluster:
    certificate-authority-data: REDACTED
    server: https://10.11.12.1:6443
  name: fcp-cluster
- cluster:
    certificate-authority-data: REDACTED
    server: https://10.11.12.3:6443
  name: chn-cluster
contexts:
- context:
    cluster: usa-cluster
    user: usa-admin
  name: usa
- context:
    cluster: fcp-cluster
    user: fcp-admin
  name: fcp
- context:
    cluster: chn-cluster
    user: chn-admin
  name: chn
current-context: fcp
kind: Config
preferences: {}
users:
- name: usa-admin
  user:
    client-certificate-data: REDACTED
    client-key-data: REDACTED
- name: fcp-admin
  user:
    client-certificate-data: REDACTED
    client-key-data: REDACTED
- name: chn-admin
  user:
    client-certificate-data: REDACTED
    client-key-data: REDACTED
```

### <a name="deploy_keepalived"></a> Deploy Keepalived

IN PROGRESS
<!-- *TODO(Walter): Add docs, references and examples -->


### <a name="label_nodes"></a> Label the nodes

We should add `failure-domain.beta.kubernetes.io/region=<region>` and `failure-domain.beta.kubernetes.io/zone=<zone>` labels for all nodes.

<!-- TODO(Arthur): Add a script for this -->

```bash
# Adding region "america" label in all nodes of usa-cluster
$ kubectl label nodes usa-master usa-worker1 usa-worker2 failure-domain.beta.kubernetes.io/region=america --context usa
node "usa-master" labeled
node "usa-worker1" labeled
node "usa-worker2" labeled

# Adding zone "us" label in all nodes of usa-cluster
$ kubectl label nodes usa-master usa-worker1 usa-worker2 failure-domain.beta.kubernetes.io/zone=us --context usa
node "usa-master" labeled
node "usa-worker1" labeled
node "usa-worker2" labeled

# Adding region "asia" label in all nodes of chn-cluster
$ kubectl label nodes c-master chn-worker1 chn-worker2 failure-domain.beta.kubernetes.io/region=asia --context chn
node "chn-master" labeled
node "chn-worker1" labeled
node "chn-worker2" labeled

# Adding zone "cn" label in all nodes of chn-cluster
$ kubectl label nodes chn-master chn-worker1 chn-worker2 failure-domain.beta.kubernetes.io/zone=cn --context chn
node "chn-master" labeled
node "chn-worker1" labeled
node "chn-worker2" labeled

# Adding region "europe" label in all nodes of fcp-cluster
$ kubectl label nodes fcp-master fcp-worker1 fcp-worker2 failure-domain.beta.kubernetes.io/region=europe --context fcp
node "fcp-master" labeled
node "fcp-worker1" labeled
node "fcp-worker2" labeled

# Adding zone "fr" label in all nodes of fcp-cluster
$ kubectl label nodes fcp-master fcp-worker1 fcp-worker2 failure-domain.beta.kubernetes.io/zone=fr --context fcp
node "fcp-master" labeled
node "fcp-worker1" labeled
node "fcp-worker2" labeled
```

### <a name="deploy_etcd_operator"></a> Deploy etcd-operator
<!-- 
NOTE(Arthur): this setup currently fails, so the helm dependency was removed

Create a `tiller` service account in the `kube-system` namespace.
Grant cluster-admin permissions to the `tiller` service account in the `kube-system` namespace. 
See more in [default roles and role bindings](https://kubernetes.io/docs/admin/authorization/rbac/#default-roles-and-role-bindings)

```bash
# Initializing and configuring Helm
$ kubectl -n kube-system create sa tiller
$ kubectl create clusterrolebinding tiller --clusterrole cluster-admin --serviceaccount=kube-system:tiller
$ helm init --service-account tiller
```

You can use the [etcd-operator helm chart](https://github.com/kubernetes/charts/tree/master/stable/etcd-operator) to do it.
```bash
$ helm install --name etcd-operator stable/etcd-operator 
$ helm upgrade etcd-operator stable/etcd-operator --set rbac.install=true --set cluster.enabled=true 
``` 
-->

<!-- TODO(Arthur): Add what are this files. -->
```
$ kubectl apply -f etcd-operator/
etcdcluster "etcd-cluster" created
deployment "etcd-operator" created
serviceaccount "etcd-operator" created
clusterrole "etcd-operator" created
clusterrolebinding "etcd-operator" created
```

<!-- TODO(Arthur): Allow use specific namespaces instead of kube-system -->
Now you should see the `etcd-cluster-client` service running
```bash
$ get svc etcd-cluster-client -n kube-system
NAME                  CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
etcd-cluster-client   10.27.240.226   <none>        2379/TCP   2m
```

### <a name="deploy_coredns"></a> Deploy CoreDNS

<!-- NOTE(Arthur) The helm dependency was removed
Create a configuration file for [CoreDNS chart](https://github.com/kubernetes/charts/tree/master/stable/coredns):

```bash
cat <<EOF > $HOME/coredns-chart-values.yaml
isClusterService: false
serviceType: "NodePort"
middleware:
  kubernetes:
    enabled: false
  etcd:
    enabled: true
    zones:
    - "example.com."
    endpoint: "http://etcd-cluster-client.default:2379"
EOF
```
> Notes:
> - The `zones` field must have your suffix domain of Federation
> - The `endpoint` field must be composed by `etcd-cluster-client` (service) `default` (namespace) `2379` (port) of your etcd client service 

Then, install CoreDNS chart with the config file above
```bash
helm install --name=coredns -f=coredns-chart-values.yaml stable/coredns
``` 
-->
<!-- TODO(Arthur): Add what are this files. -->
```
$ kubectl apply -f coredns/
configmap "coredns" created
deployment "coredns" created
serviceaccount "coredns" created
clusterrole "system:coredns" configured
clusterrolebinding "system:coredns" configured
service "coredns" created
```

Now you should see the `coredns-coredns` service running
```
$ kubectl get svc coredns -n kube-system
NAME      CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
coredns   10.27.241.14   10.210.38.67  53:32156/TCP   1m
```
<!-- TODO(Arthur): Add Note about [serviceProtocol: "TCP"] -->
### <a name="initialize_fcp"></a> Initilizing the Federation Control Plane

Create the **coredns-provider.conf** file with the following content:

```bash
cat <<EOF > $HOME/coredns-provider.conf
[Global]
etcd-endpoints = http://etcd-cluster-client.kube-system:2379
zones = example.com.
coredns-endpoints = 10.210.38.67:53
EOF
```
> Notes:
> - The `zones` field must be the same value of CoreDNS configuration
> - `coredns-endpoints` is the endpoint to access coredns server. This is an optional parameter introduced from v1.7 onwards.

Now, initialize the Federation Control Plane (Use `kubefed init --help` for more information about the parameters)

<!-- TODO: Change kubedns -->

```bash
kubefed init federation \
    --host-cluster-context="fcp" \
    --dns-provider="coredns" \
    --dns-zone-name="example.com." \
    --api-server-advertise-address=10.11.4.49 \
    --api-server-service-type='NodePort' \
    --dns-provider-config="$HOME/coredns-provider.conf" \
    --etcd-persistent-storage=false
```
> Notes:
> - The `--host-cluster-context` param is "In what context do I want my Federation Control Plane?"
> - The `--api-server-advertise-address` can be a IP address of a node of your cluster
> - You must configure [dynamic persistet volume](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#dynamic) to ensure correct operation across in case of your federation control plane restarts and set`--etcd-persistent-storage=true`

After the FCP is running, a new context are created:

```bash
$ kubectl config get-contexts
CURRENT   NAME         CLUSTER      AUTHINFO     NAMESPACE
          usa          usa          usa          
*         fcp          fcp          fcp          
          chn          chn          chn      
          federation   federation   federation
```

### <a name="join_clusters"></a> Joining Clusters

To join clustes to federation we should use the federation context

```bash
$ kubectl config use-context federation
Switched to context "federation".

# We don't have federation yet 
$ kubectl config get clusters
No resources found.
```

Now, we can join the clusters to federation

```bash
$ kubefed join fcp --host-cluster-context=fcp
cluster "fcp" created
$ kubefed join usa --host-cluster-context=fcp
cluster "usa" created
$ kubefed join chn --host-cluster-context=fcp
cluster "chn" created
```
<!--TODO CREATE NS DEFAULT-->

```bash
$ kubectl get clusters
NAME        STATUS    AGE
usa         Ready     2m
fcp         Ready     3m
chn         Ready     3m
```

### Limitations:

IN PROGRESS
<!-- TODO(Arthur/Walter) Explain the fail in service discovery -->


### Contact

Feel free to submit a PR if you want
Any questions or suggestions, contact us at artmr@lsd.ufcg.edu.br