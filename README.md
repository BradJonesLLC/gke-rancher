# Rancher 2.x stand-up scripts for GKE

This repository contains my preferred method of standing-up
a GKE Kubernetes cluster, with Rancher 2.x as an orchestration
layer, and a cluster-wide Traefik ingress behind a GCP Layer 4
load balancer.

This may not represent the best practice for your organization
or business requirements and no warranty is made regarding
suitability. Use at your own risk and with due diligence as to
the contents of the configuration included.

# Overview

The goal is to install Rancher 2.x on a GKE cluster, configure
Traefik to handle all the cluster's HTTP/HTTPS termination
(including for Rancher itself) and take advantage of GKE's hosted
K8s master while Rancher stores its configuration in the cluster's
etcd store.

## Deploy steps

1. Create a GKE cluster in the GCP web console or CLI. If you
    wish to employ network policies on Calico, GKE's network
    driver, select the Network Policy option in the advanced
    settings for the cluster. It is not enabled by default.
1. Download and install the GCP `gcloud` utility and `kubectl`,
    if you don't already have them.
1. Deploy the required cluster role bindings in `cluster-admin.yml`.
    This will require your GCP account to either have container
    service admin permissions, OR use admin credentials.
    As such:
    ```bash
    gcloud container clusters describe <cluster name> --zone <zone> | grep password
    kubectl config set-credentials <credentialname> --username=admin --password=<password from above>
    kubectl config set-context <context name> --user=<credentialname> --cluster=<full GKE cluster name>
    kubectl --context=<context name> apply -f k8s/cluster-admin.yml
    ```
1. Create the etcd operator (supervisor) and some basic cluster
    configuration: `kubectl apply -f k8s/cluster.yml`.
1. Create the [etcd cluster](https://docs.traefik.io/user-guide/kv-config/)
    for Traefik: `kubectl apply -f k8s/etcd.yml`.
    You may want to consider an etcd cluster backup or persistence
    strategy, which is out of scope, here.
1. Store the traefik config (replacing your hostmaster address
    for the default in the ACME config):
    `kubectl apply -f k8s/storeconfig.yml`.
1. Retrieve the L4 load balancer IP: `kubectl -n cluster-services get service`.
    It will be listed as the `EXTERNAL-IP`. Optionally reserve it
    as a static IP in your VPC network configuration in the cloud
    console.
1. Set a DNS record for rancher, on the load balancer IP. Place
    this hostname for `CATTLE_SERVER_URL` in `k8s/cattle.yml`,
    as well as in the ingress config in `k8s/traefik-controller.yml`.
1. Start Traefik: `kubectl apply -f k8s/traefik-controller.yml`.
1. Start Rancher: `kubectl apply -f k8s/cattle.yml`.
1. Log in to Rancher (which should now have a Let's Encrypt TLS
    certificate) and set your admin password.
1. Validate in the Google Cloud console that all your workloads
    (including system objects) are in an OK status.
1. If you are wishing to use Rancher's network policy features,
    thee is [a bug](https://github.com/rancher/rancher/issues/14085)
    relating to Rancher's assumptions around node annotations it
    expects to find from Flannel. GKE uses Calico. The ticket is
    currently scheduled for resolution on v2.0.8 but has been
    postponed before. To work around this issue, set a fake annotation
    on your cluster's nodes for this missing flannel annotation:
    ```bash
    kubectl annotate node --all flannel.alpha.coreos.com/public-ip=8.8.8.8
    ```
1. If you wish to use Google's cloud SQL with a proxy container,
    consult `k8s/mysql.yml` and [this documentation](https://cloud.google.com/sql/docs/mysql/connect-kubernetes-engine).
    You will need to create a JSON snippet of credentials and store
    them in a secret.
    
### Copyright, License and Contributions

Contributions are welcome in the form of pull requests.

Copyright 2018 Brad Jones LLC. MIT license.
