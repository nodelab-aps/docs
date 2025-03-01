---
pageTitle: Deploy TypeDB Cluster on Kubernetes
keywords: typedb, cluster, kubernetes, cloud, deployment
longTailKeywords: typedb on kubernetes
summary: Deploy TypeDB Cluster on Kubernetes
toc: false
---

## Deploying TypeDB Cluster onto Kubernetes

This guide describes how to deploy a 3-node TypeDB Cluster onto Kubernetes using [Helm](https://helm.sh/) package manager.

### Initial Setup

First, create a secret to access TypeDB Cluster image on Docker Hub:

```
kubectl create secret docker-registry private-docker-hub --docker-server=https://index.docker.io/v2/ \
--docker-username=USERNAME --docker-password='PASSWORD' --docker-email=EMAIL
```

Next, add the Vaticle Helm repo:

```
helm repo add vaticle https://repo.vaticle.com/repository/helm/
```

**Create in-flight encryption certificates (optional)**

This step is necessary if you wish to deploy TypeDB Cluster with in-flight encryption support. There are two certificates that need to be configured: external certificate (TLS) and internal certificate (Curve). The certificates need to be generated and then added to Kubernetes Secrets.

An external certificate can either be obtained from trusted third party providers such as [CloudFlare](https://www.cloudflare.com/) or [letsencrypt.org](https://letsencrypt.org/). Alternatively, it is also possible to generate it manually with [`mkcert`](https://github.com/FiloSottile/mkcert/releases):

```
$ mkcert -cert-file rpc-certificate.pem -key-file rpc-private-key.pem <server-url-address>"
```

Please note that an external certificate is always bound to URL address, not IP address.

The internal certificate can be generated using the `create-encryption-mq-key.sh` tool bundled with TypeDB Cluster distribution, which can be downloaded from [repo.vaticle.com](https://repo.vaticle.com/#browse/browse:private-artifact:vaticle_typedb_cluster):

```
$ # unpack distribution of TypeDB Cluster into `dist` folder
$ ./dist/typedb-cluster-all-<platform>-<version>/tool/create-encryption-mq-key.sh
```

Once the external and internal certificates are all generated, we can proceed to upload it to Kubernetes Secrets:

```
$ kubectl create secret generic typedb-cluster \
  --from-file rpc-private-key.pem \
  --from-file rpc-certificate.pem \
  --from-file rpc-root-ca.pem="$(mkcert -CAROOT)/rootCA.pem" \
  --from-file mq-secret-key \
  --from-file mq-public-key
```

Additionally, the secret name in Kubernetes Secret needs to be identical to the Helm release name (`typedb-cluster`) and contain exactly these keys (`rpc-private-key.pem`, `rpc-certificate.pem`, `rpc-root-ca.pem`, `mq-secret-key`, `mq-public-key`).

### Deployment Steps

Once done, there are three alternative deployment modes that you can choose from: "Private Cluster", "Public Cluster", and "Public Cluster (Minikube)". Please see the explanation of each mode below in order to decide which one works best for your requirements.

#### Deploying a Private Cluster

This deployment mode is preferred if your application is located within the same Kubernetes network as the cluster. In order to select this mode, ensure that the `exposed` flag is set to `false`.

**Deploying without in-flight encryption**

```
helm install typedb-cluster vaticle/typedb-cluster --set "exposed=false,encrypted=false"
```

Once the deployment has been completed, the servers would be accessible via the internal hostname within the Kubernetes network, ie., `typedb-cluster-0.typedb-cluster`, `typedb-cluster-1.typedb-cluster`, and `typedb-cluster-2.typedb-cluster`.

**Deploying with in-flight encryption**

Should you need to enable in-flight encryption for your private cluster, make sure the `encrypted` flag is set to `true`.

Also make sure that the external certificate is bound to `*.<helm-release-name>`. For example, for a Helm release named `typedb-cluster`, the certificate needs to be bound to `*.typedb-cluster`. 

Once done, let's perform the deployment:

```
helm install typedb-cluster vaticle/typedb-cluster --set "exposed=false,encrypted=true"
```

Once the deployment has been completed, the servers would be accessible via the internal hostname within the Kubernetes network, ie., `typedb-cluster-0.typedb-cluster`, `typedb-cluster-1.typedb-cluster`, and `typedb-cluster-2.typedb-cluster`.

#### Deploying a Public Cluster

If you need to access the cluster from outside of Kubernetes, then use this deployment mode. For example, this would be the suitable mode if you need to access the cluster from Workbase or Console running on your local machine.

Deploying a public cluster can be done by setting the `exposed` flag to `true`. 

Technically, the servers are made public by binding each one to a `LoadBalancer` instance which is assigned a public IP. The IP assignments are one automatically by the cloud provider that the Kubernetes platform is running on.

**Deploying without in-flight encryption**

```
helm install typedb-cluster vaticle/typedb-cluster --set "exposed=true"
```

Once the deployment has completed, the servers would be accessible via public IPs assigned to the Kubernetes `LoadBalancer` services. The addresses can obtained with this command:

```
kubectl get svc -l external-ip-for=typedb-cluster \
-o='custom-columns=NAME:.metadata.name,IP:.status.loadBalancer.ingress[0].ip'
```

**Deploying with in-flight encryption**

If you also want to enable in-flight encryption, there is an important requirement that must be adhered: the servers must be assigned URL addresses. This restriction comes from the fact that external certificate must be bound to a domain name, and not IP address.

Given a "domain name" and "Helm release name", The address structure of the servers will follow the specified format:

```
typedb-cluster-{0..n}.<helm-release-name>.<domain-name>
```

The format must be taken into account when generating the external certificate of all servers such that they're properly bound to the address. For example, you can generate an external certificate using wildcard, ie., `*.<helm-release-name>.<domain-name>`, that can be shared by all servers.

Once the domain name and external certificate has been configured accordingly, we can proceed to perform the deployment. Ensure that the `encrypted` flag is set to `true` and the `domain` flag set accordingly.

Once done, let's perform the deployment:

```
helm install typedb-cluster vaticle/typedb-cluster --set "exposed=true,encrypted=true,domain=<domain-name>"
```

After the deployment has been completed, we need to configure these URL addresses to correctly point to the servers. This can be done by configuring the `A record` of all the servers in your trusted DNS provider:

```
typedb-cluster-0.typedb-cluster.example.com => public IP of typedb-cluster-0 service
typedb-cluster-1.typedb-cluster.example.com => public IP of typedb-cluster-1 service
typedb-cluster-2.typedb-cluster.example.com => public IP of typedb-cluster-2 service
```

#### Deploying a Public Cluster (Minikube)

Use this deployment mode for setting up a development cluster in your local machine. However, please note that in-flight encryption *cannot* be enabled in this configuration.

First, please make sure to have [Minikube](https://minikube.sigs.k8s.io/) installed and running.

Once done, let's perform the deployment. In this example, we're adjusting various CPU and storage parameters to something smaller than the default, taking into account that resources may be more limited given that the cluster will run on a Minikube instance on your local machine.

```
helm install vaticle/typedb-cluster --generate-name \
--set "cpu=2,replicas=3,singlePodPerNode=false,storage.persistent=true,storage.size=10Gi,exposed=true"
```

Once deployment is completed, enable tunneling from another terminal:

```
minikube tunnel
```

This deployment mode is primarily inteded for development purpose. Certain adjustments will be made compared to other deployment modes:

* Minikube only has a single node, so `singlePodPerNode` needs to be set to `false`
* Minikube's node only has as much CPUs as the local machine: `kubectl get node/minikube -o=jsonpath='{.status.allocatable.cpu}'`.
  Therefore, for deploying a 3-node TypeDB Cluster to a node with 8 vCPUs, `cpu` can be set to `2` at maximum.
* Storage size probably needs to be tweaked from default value of `100Gi` (or fully disabled persistent)
  as total storage required is `storage.size` multiplied by `replicas`. In our example, total storage requirement is 30Gi.


### Configuration Reference

Configurable settings for Helm package include:

| Key                 | Default value    | Description
| :-----------------: | :---------------:| :---------------------------------------------------------------------------------------: |
| `replicas`          | `3`              | Number of TypeDB Cluster nodes to run                                                     |
| `cpu`               | `7`              | How many CPUs should be allocated for each TypeDB Cluster node                            |
| `storage.size`      | `100Gi`          | How much disk space should be allocated for each TypeDB Cluster node                      |
| `storage.persistent`| `true`           | Whether TypeDB Cluster should use a persistent volume to store data                       |
| `singlePodPerNode`  | `true`           | Whether TypeDB Cluster pods should be scheduled to different Kubernetes nodes             |
| `exposed`           | `false`          | Whether TypeDB Cluster supports connections via public IP (outside of Kubernetes network) |
| `javaopts`          | `null`           | JVM options that controls various runtime aspects of TypeDB Cluster (eg., `-Xmx`, `-Xms`) |
| `logstash.enabled`  | `false`          | Whether TypeDB Cluster pushes logs into Logstash                                          |
| `logstash.uri`      | `localhost:5044` | Hostname and port of a Logstash daemon accepting log records                              |


### Troubleshooting

These are the common error scenarios and how to troubleshoot them:

#### All pods are stuck in `ErrImagePull` or `ImagePullBackOff` state:
This means the secret to pull the image from Docker Hub has not been created. 
Make sure you've followed [Initial Setup](#initial-setup) instructions and verify that the pull secret is present by
executing `kubectl get secret/private-docker-hub`. Correct state looks like this:

```
$ kubectl get secret/private-docker-hub
NAME                 TYPE                             DATA   AGE
private-docker-hub   kubernetes.io/dockerconfigjson   1      11d
```

#### One or more pods of TypeDB Cluster are stuck in `Pending` state
This might mean pods requested more resources than available. To check if that's the case, run
`kubectl describe pod/typedb-cluster-0` on a stuck pod (e.g. `typedb-cluster-0`). Error message similar to 
`0/1 nodes are available: 1 Insufficient cpu.` or `0/1 nodes are available: 1 pod has unbound immediate PersistentVolumeClaims.`
indicates that `cpu` or `storage.size` values need to be decreased.


#### One or more pods of TypeDB Cluster are stuck in `CrashLoopBackOff` state
This might indicate any misconfiguration of TypeDB Cluster. Please obtain the logs by executing
`kubectl logs pod/typedb-cluster-0` and share them with TypeDB Cluster developers.


### Current Limitations

Deployment has several limitations which shall be resolved in the future:

* TypeDB Cluster doesn't support dynamic reconfiguration of node count without restarting all of the nodes.
