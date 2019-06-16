> Testing Rancher solutions such as Rancher, k3s, k3d, rio...

# k3d

## Get k3d

```bash
curl -s https://raw.githubusercontent.com/rancher/k3d/master/install.sh | bash
```

## Create cluster (2 workers)

```bash
k3d create --api-port 6550 --publish 8081:80 --workers 2
```

## Configure kubctl

```bash
export KUBECONFIG="$(k3d get-kubeconfig --name='k3s-default')"
kubectl cluster-info
alias k=kubectl

k cluster-info
```

will give something like:

```
Kubernetes master is running at https://localhost:6550
CoreDNS is running at https://localhost:6550/api/v1/namespaces/kube-system/services/kube-dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

## Create a deployment/service/ingress on it

```bash
k create deployment nginx --image=nginx
k create service clusterip nginx --tcp=80:80

k apply -f yaml/k3d/ingress.yaml
```

test the service:

```bash
curl localhost:8081/
```


### clean up
```bash
k delete deployment nginx
k delete service nginx
k delete ingress nginx
```


# rio

## install

```bash
curl -sfL https://get.rio.io | sh -
rio install
```

```bash
rio info
```

will output something like:
```bash
Rio Version: v0.1.1 (271d9196)
Rio CLI Version: v0.1.1 (271d9196)
Cluster Domain: 4g4n4g.on-rio.io
Cluster Domain IPs: 172.19.0.2,172.19.0.3,172.19.0.4
System Namespace: rio-system

System Components:
Autoscaler status: running
BuildController status: running
BuildKit status: running
CertManager status: running
Grafana status: running
IstioCitadel status: running
IstioPilot status: running
IstioTelemetry status: running
Kiali status: running
Prometheus status: running
Registry status: running
Webhook status: running
```

## Create a first app on rio
```bash
rio run https://github.com/rancher/rio-demo
```

### check it's allocated endpoint
```bash
rio ps
```

you'll get something like:

```bash
Name                          CREATED         ENDPOINT                                                    REVISIONS   SCALE     WEIGHT    DETAIL
default/admiring-northcutt7   6 minutes ago   https://admiring-northcutt7-default.4g4n4g.on-rio.io:9443   v0          1         100%      
```

### curl it

```bash
curl https://admiring-northcutt7-default.4g4n4g.on-rio.io:9443
Hi there, I'm running in Rio
```

## Staging / blue / green

```bash
rio run ibuildthecloud/demo:blue
```

### check blue endpoint

```bash
rio ps
```

```bash
rio ps
Name                          CREATED          ENDPOINT                                                    REVISIONS   SCALE     WEIGHT    DETAIL
default/epic-almeida5         13 minutes ago   https://epic-almeida5-default.4g4n4g.on-rio.io:9443         v0          1         100%      
default/admiring-northcutt7   2 hours ago      https://admiring-northcutt7-default.4g4n4g.on-rio.io:9443   v0          1         100%      

```

You get another endpoint now:

open in browser `https://epic-almeida5-default.4g4n4g.on-rio.io:9443`


### check revisions

```bash
rio revision
```


```bash
Name                             IMAGE                                                                  CREATED          SCALE     ENDPOINT                                                       WEIGHT    DETAIL
default/epic-almeida5:v0         ibuildthecloud/demo:blue                                               16 minutes ago   1         https://epic-almeida5-v0-default.4g4n4g.on-rio.io:9443         100       
default/admiring-northcutt7:v0   default/admiring-northcutt7:786b366d5d44de6b547939f51d467437e45c5ee1   2 hours ago      1         https://admiring-northcutt7-v0-default.4g4n4g.on-rio.io:9443   100   
```

### stage green version


Let's say we want to update the image now, to be served by green cows:

```bash
rio stage --image ibuildthecloud/demo:green default/epic-almeida5:v1
```

```bash
rio revision 
Name                             IMAGE                                                                  CREATED          SCALE     ENDPOINT                                                       WEIGHT    DETAIL
default/epic-almeida5:v1         ibuildthecloud/demo:green                                              16 seconds ago   1         https://epic-almeida5-v1-default.4g4n4g.on-rio.io:9443         0         
default/epic-almeida5:v0         ibuildthecloud/demo:blue                                               20 minutes ago   1         https://epic-almeida5-v0-default.4g4n4g.on-rio.io:9443         100       
default/admiring-northcutt7:v0   default/admiring-northcutt7:786b366d5d44de6b547939f51d467437e45c5ee1   2 hours ago      1         https://admiring-northcutt7-v0-default.4g4n4g.on-rio.io:9443   100       

```

Open the new endpoint and watch a green cow there:

https://epic-almeida5-v1-default.4g4n4g.on-rio.io:9443

rio ps shows that the revision v0 is still serving the traffic.

```bash
rio ps
Name                          CREATED          ENDPOINT                                                    REVISIONS   SCALE     WEIGHT    DETAIL
default/epic-almeida5         24 minutes ago   https://epic-almeida5-default.4g4n4g.on-rio.io:9443         v0          1         100%      
default/admiring-northcutt7   2 hours ago      https://admiring-northcutt7-default.4g4n4g.on-rio.io:9443   v0          1         100%      
```

## promoting staged revision

gradual traffic shifting will happen when we promote the revision to serve the traffic:

```bash
rio promote default/epic-almeida5:v1
```
