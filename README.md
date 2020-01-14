# Collection of yamls for a Traefik Webinar

## Setup your cluster

I use (github/clemenko/ucp)[https://github.com/clemenko/ucp] for building Docker Enterprise. Honestly any k8s distro should work. 

## deploy traefik

`kubectl apply -f traefik_ingress_controller.yml`

## deploy prometheus and grafana

`kubectl apply -f prometheus/. `


