# Kubernetes Federation for on-premises clusters

If you don't know the basics of Kubernetes Federation, you can take a look at this [doc](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/federation.md).

## Scenario

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

## Objectives

Deploy a Federation Control Plane (FCP) on `fcp-cluster` and to join the other two clusters to Federation. 

There are some necessary steps to do it:
1. [Get the credentials of the clusters](#get_credentials)
1. [Deploy keepalived-cloud-provider to exopose services with LoadBalancer type](#deploy_keepalived)
1. [Add the `region` and `zone` labels for each nodes](#label_nodes)
1. [Deploy etcd-operator as backend of CoreDNS](#deploy_etcd_operator)
1. [Deploy the CoreDNS as DNS provider](#deploy_coredns)
1. [Initialize your FCP](#initialize_fcp)
1. [Join the clusters](#join_clusters)

## <a name="get_credentials"></a> Get the credentials

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

## <a name="deploy_keepalived"></a> Deploy keepalived-cloud-provider

Before to build our own solution, we are investigating some ways to create services with `LoadBalancer` type because it's necessary for service discovery in federation, one of them is called [keepalived-cloud-provider](https://github.com/munnerz/keepalived-cloud-provider).
<!-- TODO(arthur): Add RBAC -->
We don't worry about RBAC configuration, and this should be updated soon, for now, we created a `permissive-binding` to supply this need.

> Note:
> - Permissive RBAC permission is not a recommended policy or suitable for production. See more [here](https://kubernetes.io/docs/admin/authorization/rbac/).

```bash
# NOTE: Enabling this means the “kube-system” namespace contains secrets that grant super-user access to the API.
kubectl create clusterrolebinding add-on-cluster-admin \
  --clusterrole=cluster-admin \
  --serviceaccount=kube-system:default

$ kubectl -n kube-system create sa tiller
$ kubectl create clusterrolebinding tiller --clusterrole cluster-admin --serviceaccount=kube-system:tiller
$ helm init --service-account tiller

# Creating a permissive binding for each context
$ kubectl create clusterrolebinding permissive-binding \
  --clusterrole=cluster-admin \
  --user=admin \
  --user=kubelet \
  --group=system:serviceaccounts \
  --context=fcp
clusterrolebinding "permissive-binding" created
$ kubectl create clusterrolebinding permissive-binding \
  --clusterrole=cluster-admin \
  --user=admin \
  --user=kubelet \
  --group=system:serviceaccounts \
  --context=chn
clusterrolebinding "permissive-binding" created
$ kubectl create clusterrolebinding permissive-binding \
  --clusterrole=cluster-admin \
  --user=admin \
  --user=kubelet \
  --group=system:serviceaccounts \
  --context=usa
clusterrolebinding "permissive-binding" created
```

keepalived-cloud-provider has an [env variable](https://github.com/ufcg-lsd/k8s-onpremise-federation/blob/master/keepalived-cloud-provider/keepalived-provider.yaml#L33) to set the CIDR reserved for services, our plan is to have a DHCP service to supply IPs for them.

```bash
$ kubectl apply -f keepalived-cloud-provider/ --context fcp
deployment "keepalived-cloud-provider" created
daemonset "kube-keepalived-vip" created
configmap "vip-configmap" created
$ kubectl apply -f keepalived-cloud-provider/ --context chn
deployment "keepalived-cloud-provider" created
daemonset "kube-keepalived-vip" created
configmap "vip-configmap" created
$ kubectl apply -f keepalived-cloud-provider/ --context usa
deployment "keepalived-cloud-provider" created
daemonset "kube-keepalived-vip" created
configmap "vip-configmap" created
```

## <a name="label_nodes"></a> Label the nodes

We should add `failure-domain.beta.kubernetes.io/region=<region>` and `failure-domain.beta.kubernetes.io/zone=<zone>` labels for all nodes.

```bash
# Adding region "america" and zone "us" labels in all nodes of usa-cluster
$ kubectl label --all nodes failure-domain.beta.kubernetes.io/region=america --context usa
$ kubectl label --all nodes failure-domain.beta.kubernetes.io/zone=us --context usa

# Adding region "asia" and zone "cn" labels in all nodes of chn-cluster
$ kubectl label --all nodes failure-domain.beta.kubernetes.io/region=asia --context chn
$ kubectl label --all nodes failure-domain.beta.kubernetes.io/zone=cn --context chn

# Adding region "europe" and zone "fr" labels in all nodes of fcp-cluster
$ kubectl label --all nodes failure-domain.beta.kubernetes.io/region=europe --context fcp
$ kubectl label --all nodes failure-domain.beta.kubernetes.io/zone=fr --context fcp
```

## <a name="deploy_etcd_operator"></a> Deploy etcd-operator

```bash
$ kubectl apply -f etcd-operator/rbac.yaml 
serviceaccount "etcd-operator" created
clusterrole "etcd-operator" created
clusterrolebinding "etcd-operator" created
$ kubectl apply -f etcd-operator/deployment.yaml 
deployment "etcd-operator" created
$ kubectl apply -f etcd-operator/cluster.yaml 
etcdcluster "etcd-cluster" created
```

Now you should see the `etcd-cluster-client` service running on 2379 port.
```bash
$ kubectl get svc etcd-cluster-client
NAME                  CLUSTER-IP     EXTERNAL-IP   PORT(S)             AGE
etcd-cluster-client   10.100.41.37   <none>        2379/TCP            1m

# Testing etcd endpoint (etcd-cluster-client.default:2379) 
$ kubectl run --rm -i --tty fun --image quay.io/coreos/etcd --restart=Never -- /bin/sh
If you don't see a command prompt, try pressing enter.
/ # ETCDCTL_API=3 etcdctl --endpoints http://etcd-cluster-client.default:2379 put foo bar
OK
/ # ETCDCTL_API=3 etcdctl --endpoints http://etcd-cluster-client.default:2379 get foo
foo
bar
/ # ETCDCTL_API=3 etcdctl --endpoints http://etcd-cluster-client.default:2379 del foo
1
```

> Note:
> - It's possible use the [etcd-operator helm chart](https://github.com/kubernetes/charts/tree/master/stable/etcd-operator) too, to deploy it. 

## <a name="deploy_coredns"></a> Deploy CoreDNS

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
$ kubectl get svc coredns
NAME        CLUSTER-IP     EXTERNAL-IP   PORT(S)                     AGE
coredns     10.108.3.198   <nodes>       53:30053/UDP,53:30053/TCP   1m
```
<!-- 
############3

kubectl exec -ti busybox -- nslookup  hello-lsd.default.federation.svc.fr.europe.example.com


cannot update endpoints in the namespace "kube-system"

kubectl run -it --rm --restart=Never --image=infoblox/dnstools:latest dnstools
host hello-lsd.default.federation.svc.fr.europe.example.com
hello-lsd.default.federation.svc.fr.europe.example.com has address 10.210.1.68
########## -->


> Note:
> - It's possible use the [coredns helm chart](https://github.com/kubernetes/charts/tree/master/stable/coredns) too, to deploy it. 
<!-- > - The CoreDNS [service](/coredns/service.yaml#L18) can be created with `LoadBalancer` type because the [keepalived-cloud-provider](#deploy_keepalived) is running.
> - The CoreDNS [service](/coredns/service.yaml#14), by default is deployed to listen on both "TCP" and "UDP", but we can't create this service with `type: LoadBalancer`, so it is preferred to use either "TCP" or "UDP". -->


## <a name="initialize_fcp"></a> Initilizing the Federation Control Plane

This setup requires the kubefed, the main tool to initialize your Federation Control Plane and join/unjoin the clusters to Federation. 

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
$ cp kubernetes/client/bin/kubefed /usr/local/bin
$ chmod +x /usr/local/bin/kubefed
$ cp kubernetes/client/bin/kubectl /usr/local/bin
$ chmod +x /usr/local/bin/kubectl
```

Create the **coredns-provider.conf** file with the following content:

```bash
cat <<EOF > $HOME/coredns-provider.conf
[Global]
etcd-endpoints = http://etcd-cluster-client.fed-dns:2379
zones = example.com.
coredns-endpoints = 10.210.1.65:53
EOF
```
> Notes:
> - The `zones` field must be the same value of CoreDNS [configuration](coredns/configmap.yaml#L10)
> - `coredns-endpoints` is the endpoint to access coredns server. This is an optional parameter introduced from v1.7 onwards.

Now, initialize the Federation Control Plane (Use `kubefed init --help` for more information about the parameters)

```bash
kubefed init federation \
    --host-cluster-context="fcp" \
    --dns-provider="coredns" \
    --dns-zone-name="example.com." \
    --api-server-advertise-address=10.11.4.82 \
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

## <a name="join_clusters"></a> Joining Clusters

To join clustes to federation we should use the federation context

```bash
$ kubectl config use-context federation
Switched to context "federation".

# We don't have federatad clusters yet 
$ kubectl config get clusters
No resources found.
```

Now, we can join our clusters to federation

```bash
$ kubefed join fcp --host-cluster-context=fcp
cluster "fcp" created
$ kubefed join usa --host-cluster-context=fcp
cluster "usa" created
$ kubefed join chn --host-cluster-context=fcp
cluster "chn" created

$ kubectl get clusters
NAME        STATUS    AGE
usa         Ready     2m
fcp         Ready     3m
chn         Ready     3m
```

<!--TODO CREATE NS DEFAULT AND DEPLOY/SERVICES EXAMPLES-->

## Limitations:

IN PROGRESS
<!-- TODO(Arthur/Walter) Explain the fail in service discovery -->


## Contact

Feel free to submit a PR if you want
Any questions or suggestions, contact us at artmr@lsd.ufcg.edu.br