# Traefik Webinar

## Setup and Install

### Setup your cluster

I use [github/clemenko/ucp](https://github.com/clemenko/ucp) for building Docker Enterprise. Honestly any k8s distribution should work. All my hosting is done on [DigitalOcean](http://digitlocean.com). They Rock!

### Label ingress nodes

The idea is to label the specific ingress nodes you want to use. This will allow for an easier setup of the external load balancer and create the fastest path to the pod.

```bash
kubectl label nodes ddc-bc41 ddc-a12c ddc-8393 traefik=ingress
```

### Deploy Traefik

deploy :

```bash
git clone https://github.com/clemenko/traefik_webinar
cd traefik_webinar
kubectl apply -f traefik_ingress_controller.yml
```

example :

```bash
clemenko@clemenko traefik_webinar % kubectl apply -f traefik_ingress_controller.yml
namespace/ingress-traefik created
serviceaccount/ingress-traefik created
clusterrole.rbac.authorization.k8s.io/ingress-traefik created
clusterrolebinding.rbac.authorization.k8s.io/ingress-traefik created
deployment.apps/traefik-ingress-controller created
service/traefik-ingress-service created
ingress.networking.k8s.io/traefik-ingress created
```

We can take a second to verify that the pods are on the nodes with the labels. Run `kubectl get pods` and double check the nodes.

```bash
kubectl get pods -n ingress-traefik -o wide
```

Get dashboard port for traefik and navigate there.

```bash
kubectl get svc -n ingress-traefik traefik-ingress-service -o=jsonpath='{.spec.ports[?(@.port==8080)].nodePort}'; echo ""
```

The output was port 33981, so navigate to any node in the cluster on port 33981.

![dashboard](imgs/dashboard.jpg)

### Setup load balancer

For the webinar I am going to use [DigitalOcean](http://digitlocean.com) again.

Another option is to setup a node to run nginx as a load balancer. Ideally use your cloud providers' lb serivce. Note the IPs in the conf. They point to the three managers nodes of the cluster. Make sure you change the port for the backend servers to the correct NodePort for Traefik. You can find it with the following command.

```bash
kubectl get svc -n ingress-traefik traefik-ingress-service -o=jsonpath='{.spec.ports[?(@.port==80)].nodePort}'; echo ""
```

And then the setup itself for Nginx.

```bash
yum install -y yum-utils vim
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
yum install docker-ce -y
systemctl start docker
systemctl enable docker

cat << EOF >> stream.conf
user  nginx;
worker_processes  auto;

error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;

events {
    worker_connections  1024;
}

stream {
    upstream stream_backend {
        server ucp.dockr.life:XXXX max_fails=3 fail_timeout=2s;
        server ucp2.dockr.life:XXXX max_fails=3 fail_timeout=2s;
        server ucp3.dockr.life:XXXX max_fails=3 fail_timeout=2s;
    }

    server {
        listen        80;
        proxy_pass    stream_backend;
        proxy_timeout 3s;
        proxy_connect_timeout 1s;
    }
}
EOF

docker run --rm -d -p 80:80 -v /root/stream.conf:/etc/nginx/nginx.conf:ro nginx:alpine
```

### Deploy Prometheus and Grafana

Deploy

```bash
kubectl apply -f prometheus/.
```

example :

```bash
clemenko@clemenko traefik_webinar % kubectl apply -f prometheus/. 
clusterrolebinding.rbac.authorization.k8s.io/kube-state-metrics created
clusterrole.rbac.authorization.k8s.io/kube-state-metrics created
deployment.apps/kube-state-metrics created
rolebinding.rbac.authorization.k8s.io/kube-state-metrics created
role.rbac.authorization.k8s.io/kube-state-metrics-resizer created
serviceaccount/kube-state-metrics created
service/kube-state-metrics created
namespace/monitoring created
clusterrole.rbac.authorization.k8s.io/prometheus created
clusterrolebinding.rbac.authorization.k8s.io/prometheus created
serviceaccount/prometheus created
daemonset.extensions/node-exporter created
daemonset.extensions/cadvisor created
service/alertmanager created
configmap/alertmanager-config created
configmap/alertmanager-templates created
deployment.extensions/alertmanager created
configmap/prometheus-configmap created
service/prometheus-svc created
deployment.extensions/prometheus-deployment created
configmap/grafana-datasource created
configmap/grafana-dashboard-providers created
service/grafana created
deployment.extensions/grafana created
configmap/blackbox-configmap created
deployment.extensions/blackbox-exporter created
service/blackbox-exporter created
ingress.extensions/grafana-ingress created
ingress.extensions/prom-ingress created
configmap/grafana-dashboards created
```

Let's look at the grafana dashboard at [grafana.dockr.life](grafana.dockr.life).

The login for grafana is `admin / Pa22word`.

Out of the box I have included a few handy dashboards. Check out the `Kube - Cluster Health` dashboard.

![grafana](imgs/grafana.jpg)

The prometheus dashboard can be found at [prom.dockr.life](prom.dockr.life).

You can also check the `routers` that Traefik knows about by exploring the `Routers` page.

![routers](imgs/routers.jpg)

### Deploy a few apps

```bash
kubectl apply -f k8s_all_the_things.yml
kubectl apply -f whoami.yml
```

Now check the Traefik `Router`.

## Check our StackRox

## Caveats

For this demo I am not using state-full storage. Typically you will want to use the best storage solution from your hosting provider.
