# [WIP] Kubernetes Federation for on-premises clusters

If you don't know the basics of Kubernetes Federation, you can take a look at this [doc](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/federation.md).

## Scenario

This setup is for on-premise clusters, i.e., not attached to any cloud provider.
This is not suitable for production yet.

|   Clusters   |  API Server      |  Region    | Zone   |
|:------------:|:----------------:|:----------:|:------:|
| fcp-cluster  | 10.11.12.1:6443  | europe     | fr     |
| usa-cluster  | 10.11.12.2:6443  | america    | us     |
| chn-cluster  | 10.11.12.3:6443  | asia       | cn     |

The fields Region and Zone will be used later. For now, just be aware they represent that clusters can be geo distributed.

We created three clusters created using [kubeadm](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/) tool 

- Kubernetes version: 1.7.3
- OS (cluster's nodes): Ubuntu 16.04.2 LTS
- Authorization Mode: RBAC

## Objectives

* Deploy a Federation Control Plane (FCP) on `fcp-cluster` and join the other two clusters to Federation.
* Test cross-cluster service discovery

Steps to do it:
1. [Get the credentials of the clusters](#get_credentials)
1. [Add the `region` and `zone` labels for each nodes](#label_nodes)
1. [Deploy etcd-operator as backend of CoreDNS](#deploy_etcd_operator)
1. [Deploy the CoreDNS as Federation DNS provider](#deploy_coredns)
1. [Initialize your FCP](#initialize_fcp)
1. [Join the clusters](#join_clusters)
1. [Deploy keepalived-cloud-provider to exopose services with LoadBalancer type](#deploy_keepalived)

## <a name="get_credentials"></a> Get the credentials

As you have multiple clusters, you must be able to [authenticate in them with kubeconfig](https://kubernetes.io/docs/tasks/access-application-cluster/authenticate-across-clusters-kubeconfig/)

Basically, each cluster has a kubeconfig file (by default `$HOME/.kube/config` on master node) and you should merge these files to access your clusters from a single point.

A pair of information composed by "Where your API server is running" and the auth info to access this API is called "context".
See `kubectl config --help`.

```bash
kubectl config get-contexts
CURRENT   NAME      CLUSTER       AUTHINFO    NAMESPACE
          usa       usa-cluster   usa-admin   
*         fcp       fcp-cluster   fcp-admin  
          chn       chn-cluster   chn-admin
```

As an example, this scenario has the following kubeconfig:

```yaml
kubectl config view
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

## <a name="label_nodes"></a> Label the nodes

You need to add `failure-domain.beta.kubernetes.io/region=<region>` and
`failure-domain.beta.kubernetes.io/zone=<zone>` labels for all nodes in each cluster.

This step is needed to have service discovery based on the region and zone. 

```bash
# Adding region "america" and zone "us" labels in all nodes of usa-cluster
kubectl label --all nodes failure-domain.beta.kubernetes.io/region=america --context usa
kubectl label --all nodes failure-domain.beta.kubernetes.io/zone=us --context usa

# Adding region "asia" and zone "cn" labels in all nodes of chn-cluster
kubectl label --all nodes failure-domain.beta.kubernetes.io/region=asia --context chn
kubectl label --all nodes failure-domain.beta.kubernetes.io/zone=cn --context chn

# Adding region "europe" and zone "fr" labels in all nodes of fcp-cluster
kubectl label --all nodes failure-domain.beta.kubernetes.io/region=europe --context fcp
kubectl label --all nodes failure-domain.beta.kubernetes.io/zone=fr --context fcp
```

## <a name="deploy_etcd_operator"></a> Deploy etcd-operator

The Federation Control Plane will be deployed in the `fcp-cluster`, which means that the `etcd-operator` and `CoreDNS` will be deployed in `fcp` context.

```bash
# RBAC
kubectl apply -f etcd-operator/rbac.yaml 
serviceaccount "etcd-operator" created
clusterrole "etcd-operator" created
clusterrolebinding "etcd-operator" created

# Deployment
kubectl apply -f etcd-operator/deployment.yaml 
deployment "etcd-operator" created

# Wait the Pod status of etcd-operator to be running before creating the etcd-cluster
kubectl apply -f etcd-operator/cluster.yaml 
etcdcluster "etcd-cluster" created
```

Now you should see the `etcd-cluster-client` service running on 2379 port.
```bash
kubectl get svc etcd-cluster-client
NAME                  CLUSTER-IP     EXTERNAL-IP   PORT(S)             AGE
etcd-cluster-client   10.100.41.37   <none>        2379/TCP            1m
```

You can run the commands below to test your `etcd-cluster-client` endpoint
```bash
kubectl run --rm -i --tty fun --image quay.io/coreos/etcd --restart=Never -- /bin/sh
/ # ETCDCTL_API=3 etcdctl --endpoints http://etcd-cluster-client.default:2379 put foo bar
OK
/ # ETCDCTL_API=3 etcdctl --endpoints http://etcd-cluster-client.default:2379 get foo
foo
bar
/ # ETCDCTL_API=3 etcdctl --endpoints http://etcd-cluster-client.default:2379 del foo
1
```

> Note:
> - It is possible to use the [etcd-operator helm chart](https://github.com/kubernetes/charts/tree/master/stable/etcd-operator) to deploy it, however, we have found find some issues in this chart, such as [outdated cluster template](https://github.com/kubernetes/charts/issues/1795). We chose to create the resources directly. 

## <a name="deploy_coredns"></a> Deploy CoreDNS

We use the [CoreDNS helm charts](https://github.com/kubernetes/charts/tree/master/stable/coredns) to deploy the CoreDNS as DNS provider of federation. 

Getting Helm. See the [install docs](https://docs.helm.sh/using_helm/#installing-helm).

```bash
curl https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get > get_helm.sh
chmod 700 get_helm.sh
./get_helm.sh
```

Grant superuser access to Helm.

```bash
kubectl -n kube-system create sa tiller
kubectl create clusterrolebinding tiller --clusterrole cluster-admin --serviceaccount=kube-system:tiller
helm init --service-account tiller
```

The CoreDNS default configuration should be customized to suit the federation. Below is the coredns-chart-config.yaml, which overrides the default configuration parameters on the CoreDNS chart.

```bash
cat <<EOF >  $HOME/coredns-chart-values.yaml
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

The above configuration file needs some explanation:

* **isClusterService** specifies whether CoreDNS should be deployed as a cluster-service, which is the default. You need to set it to false, so that CoreDNS is deployed as a Kubernetes application service.
* **serviceType** specifies the type of Kubernetes service to be created for CoreDNS. You need to choose either “LoadBalancer” or “NodePort” to make the CoreDNS service accessible outside the Kubernetes cluster.
* Disable **middleware.kubernetes**, which is enabled by default by setting **middleware.kubernetes.enabled** to false.
* Enable **middleware.etcd** by setting **middleware.etcd.enabled** to true.
* Configure the DNS zone (federation domain) for which CoreDNS is authoritative by setting **middleware.etcd.zones** as shown above.
* Configure the etcd endpoint which was deployed earlier by setting **middleware.etcd.endpoint**

Deploy CoreDNS
```bash
helm install --name coredns -f  $HOME/coredns-chart-values.yaml  stable/coredns

# Now you should see the `coredns-coredns` service running
kubectl get svc coredns-coredns
NAME                  CLUSTER-IP      EXTERNAL-IP   PORT(S)                     AGE
coredns-coredns       10.103.252.94   <nodes>       53:32609/UDP,53:32609/TCP   1m
```

## <a name="initialize_fcp"></a> Initilizing the Federation Control Plane

This setup requires kubefed, the main tool to initialize your Federation Control Plane and join/unjoin the clusters to Federation.

```bash
# Getting Kubefed
# Linux
curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/kubernetes-client-linux-amd64.tar.gz
tar -xzvf kubernetes-client-linux-amd64.tar.gz

# OS X
curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/kubernetes-client-darwin-amd64.tar.gz
tar -xzvf kubernetes-client-darwin-amd64.tar.gz

# Windows
curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/kubernetes-client-windows-amd64.tar.gz
tar -xzvf kubernetes-client-windows-amd64.tar.gz
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

Create the **coredns-provider.conf** file with the following command:

```bash
cat <<EOF > $HOME/coredns-provider.conf
[Global]
etcd-endpoints = http://etcd-cluster-client.default:2379
zones = example.com.
coredns-endpoints = 10.11.4.82:32609
EOF
```

> Notes:
> - The `zones` field must have the same value of CoreDNS chart config 
> - `coredns-endpoints` is the endpoint to access coredns server. CoreDNS was deployed with `NodePort` type, so the endpoints is composed by `<ip-of-nodes>:<port-of-service>` This is an optional parameter introduced from v1.7 onwards.

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
> - You must configure [dynamic persistent volume](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#dynamic) to ensure correct operation across in case of your federation control plane restarts and set`--etcd-persistent-storage=true`

After the FCP is running, a new context is created:

```bash
kubectl config get-contexts
CURRENT   NAME         CLUSTER      AUTHINFO     NAMESPACE
          usa          usa          usa          
*         fcp          fcp          fcp          
          chn          chn          chn      
          federation   federation   federation
```

## <a name="join_clusters"></a> Joining Clusters

To join clustes to federation we should use the federation context

```bash
kubectl config use-context federation
Switched to context "federation".

# We don't have federatad clusters yet 
kubectl config get clusters
No resources found.
```

Now, we can join our clusters to federation

```bash
kubefed join fcp --host-cluster-context=fcp
cluster "fcp" created
kubefed join usa --host-cluster-context=fcp
cluster "usa" created
kubefed join chn --host-cluster-context=fcp
cluster "chn" created

kubectl get clusters
NAME        STATUS    AGE
usa         Ready     2m
fcp         Ready     3m
chn         Ready     3m
```

## <a name="deploy_keepalived"></a> Deploy keepalived-cloud-provider

We are investigating some ways to create services with `LoadBalancer` type as it's currently necessary for service discovery in federation. One of them is called [keepalived-cloud-provider](https://github.com/munnerz/keepalived-cloud-provider).

We don't have RBAC resources for keepalived-cloud-provider, for now, we need create the service account called `keepalived` and grant superuser access to it. This will be updated soon.

```
kubectl create sa keepalived -n kube-system --context fcp
kubectl create sa keepalived -n kube-system --context usa
kubectl create sa keepalived -n kube-system --context chn

kubectl create clusterrolebinding keepalived \
  --clusterrole=cluster-admin \
  --serviceaccount=kube-system:keepalived \
  --context=fcp
kubectl create clusterrolebinding keepalived \
  --clusterrole=cluster-admin \
  --serviceaccount=kube-system:keepalived \
  --context=usa
kubectl create clusterrolebinding keepalived \
  --clusterrole=cluster-admin \
  --serviceaccount=kube-system:keepalived \
  --context=chn
```

Now, it's possible to create the `kube-keepalive-vip` and `keepalive-cloud-provider`.
The `keepalived-cloud-provider` has a [CIDR param](/keepalived-cloud-provider/deployment.yaml#35). Changing this value is indicated if you want to have different CIDRs for our clusters, consequently one IP for each service.

```bash
# Creating kube-keealived-vip for each context
kubectl apply -f keepalived-vip/ --context fcp
kubectl apply -f keepalived-vip/ --context chn
kubectl apply -f keepalived-vip/ --context usa
# Creating keealived-cloud-provider for each context
kubectl apply -f keepalived-cloud-provider/ --context fcp
# Change CIDR to 10.210.2.100/26
kubectl apply -f keepalived-cloud-provider/ --context chn
# Change CIDR to 10.210.3.100/26
kubectl apply -f keepalived-cloud-provider/ --context usa
```

### Configure the kube-controller-manager.

This step is needed **for each cluster**. 

If you deployed your cluster with kubeadm tool, you should edit `/etc/kubernetes/manifests/kube-controller-manager.yaml` file, adding `--cloud-provider=external` to the command section. 

If you are using `kubeadm` file, then the following fragment will enable the external cloud provider. 

```yaml
controllerManagerExtraArgs:
  cloud-provider: external
```

Now, it's possibe to create services with `LoadBalancer` type in the on-premisses clusters.
> Note:
> - This `keepalived-cloud-provider` is in alpha state. 

## Cross-cluster service discovery

You need to add the CoreDNS server to the pod’s nameserver `resolv.conf` chain in all the federated clusters as this self hosted CoreDNS server is not publicly discoverable. This can be achieved by adding the below line to dnsmasq container’s arg in kube-dns deployment.
`--server=/example.com./<CoreDNS endpoint>`

```bash 
# Configure fcp-cluster and do it the same for each cluster 
kubectl config use-context fcp

# In our case we add --server=/example.com./10.11.4.82:32609 arg
kubectl edit deploy kube-dns -n kube-system
```

### Create a federated service

```bash
# The namespace default is not created in federation context, so create it.
kubectl create ns default
# Create a `hello-world` as a federated deployment/service 
kubectl create -f hello-world/

# Each service has a own IP
kubectl get svc hello-world --context fcp
NAME          CLUSTER-IP       EXTERNAL-IP   PORT(S)         AGE
hello-world   10.105.48.221    10.210.1.68   8080:30497/TCP  12s

kubectl get svc hello-world --context chn
NAME          CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
hello-world   10.96.134.187   10.210.2.65   8080:32739/TCP   13s

kubectl get svc hello-world --context usa
NAME          CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
hello-world   10.105.126.36   10.210.3.65   8080:32112/TCP   13s

# The federated service has the IPs of each service
kubectl get svc hello-world
NAME          CLUSTER-IP   EXTERNAL-IP        PORT(S)    AGE
hello-world                10.210.1.68,1...   8080/TCP   2m
```

### Test the cross-cluster service discovery
```
kubectl config use-context fcp
kubectl run -i --tty busybox --restart=Never --image=busybox -- sh
If you don't see a command prompt, try pressing enter.
/ # nslookup  hello-world.default.federation.svc.fr.europe.example.com
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      hello-world.default.federation.svc.fr.europe.example.com
Address 1: 10.210.1.68
/ # nslookup  hello-world.default.federation.svc.us.america.example.com
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      hello-world.default.federation.svc.us.america.example.com
Address 1: 10.210.3.65
/ # nslookup  hello-world.default.federation.svc.cn.asia.example.com
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      hello-world.default.federation.svc.cn.asia.example.com
Address 1: 10.210.2.65
```

## References:

* [Set up Cluster Federation with Kubefed](https://kubernetes.io/docs/tasks/federation/set-up-cluster-federation-kubefed/)
* [Set up CoreDNS as DNS provider for Cluster Federation](https://kubernetes.io/docs/tasks/federation/set-up-coredns-provider-federation/)
* [Keepalived Cloud provider](https://github.com/munnerz/keepalived-cloud-provider)
* [Core Etcd Operator](https://github.com/coreos/etcd-operator) 
* [Experimenting with Cross Cloud Kubernetes Cluster Federation](https://medium.com/google-cloud/experimenting-with-cross-cloud-kubernetes-cluster-federation-dfa99f913d54)
* [Cross-cluster Service Discovery using Federated Services](https://kubernetes.io/docs/tasks/federation/federation-service-discovery/)

## Limitations

* We tried to deploy the `etcd-operator` and `CoreDNS` in a different namespace than `default`, but this setup fails.
* We tried to create the `CoreDNS` service with `LoadBalancer` type, creating the keepalived-cloud-provider before setup the Federation, but this setup fails.
* Our CoreDNS server is not publicly discoverable.

Obs: We are investigating these limitations and soon we expected to report a more descriptive issue and contribute to solving them.

## Contact

Feel free to submit a PR if you want

Any questions or suggestions, contact us at artmr@lsd.ufcg.edu.br
