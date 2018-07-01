# Kubernetes nginx-ingress Example

This repo is an example Helm chart for tying together:

 *  [nginx-ingress](https://github.com/kubernetes/ingress-nginx): route external traffic in the cluster.
 *  [cert-manager](https://github.com/jetstack/cert-manager): automatically provision TLS certificates for the routed
    domains.
 *  [external-dns](https://github.com/kubernetes-incubator/external-dns): automatically create DNS records for the
    routed domains in an external DNS provider.

using [Let's Encrypt](https://letsencrypt.org/) for TLS certs and [Google Cloud DNS](https://cloud.google.com/dns/) for
DNS hosting.


## nginx-ingress

To have the ingress controller service use a specific IP, configure `nginx-ingress.controller.service.loadBalancerIP` in
`values.yaml`.


## cert-manager

`cert-manager` watches Ingress resources and creates certs based on the content of the `spec.tls` section. **Without
that section, no cert is created.** The `secretName` property in that section is where the key and cert will be stored.

`templates/cluster-issuer.yaml` configures `cert-manager` to use Let's Encrypt (as opposed to other cert issuers). It is
configured to use the HTTP01 challenge, which `cert-manager` will handle.

In `values.yaml`, add the email for your Let's Encrypt account to `letsencrypt.email`.

Add your Let's Encrypt private key to a Secret called `letsencrypt-private-key`. E.g. if you had previously used Lego:

    kubectl create secret generic letsencrypt-private-key \
      --from-file=tls.key="~/.lego/accounts/acme-v02.api.letsencrypt.org/you@example.org/keys/you@example.org.key"


# external-dns

In `values.yaml`, `external-dns.txtOwnerId` is used as a check to ensure `external-dns` doesn't change records it's not
supposed to.


# Hello World service

This sets up a 'Hello World' web server using the `nginxdemos/hello` image.

Update `hello-world/templates/ingress.yaml` with a real domain name for the `tls` and `rules` parts to try it out.


## Cluster Setup

The cluster's node pool requires the `clouddns` scope so that ExternalDNS can update DNS records. The Google load
balancing controller should also be disabled, so that Ingress resources in the cluster don't create load balancing
resources within GCP (`nginx-ingress` will handle this traffic).

An existing cluster can be modified by creating a new node pool with the required scopes, and updating the cluster to
disable the `HttpLoadBalancing` addon.

### Creating a new cluster

 1. [Create the cluster](https://cloud.google.com/container-engine/docs/clusters/operations#creating_a_container_cluster):

        gcloud container clusters create mycluster \
          --zone us-central1-a \
          --machine-type g1-small \
          --num-nodes 1 \
          --disk-size 20 \
          --scopes="https://www.googleapis.com/auth/devstorage.read_only,\
https://www.googleapis.com/auth/logging.write,\
https://www.googleapis.com/auth/monitoring,\
https://www.googleapis.com/auth/ndev.clouddns.readwrite,\
https://www.googleapis.com/auth/service.management.readonly,\
https://www.googleapis.com/auth/servicecontrol,\
https://www.googleapis.com/auth/trace.append" \
          --addons=HttpLoadBalancing=DISABLED

 2. [Install Helm and Tiller](https://github.com/kubernetes/helm/blob/master/docs/install.md)

 3. Set up Tiller RBAC:

        kubectl apply -f ./tiller/rbac.yaml

 4. Install Tiller resources:

        helm install --name mycluster .

## Installing the example

    helm dep up
    helm upgrade --install mycluster .

### apiVersion is not available

If you receive the error:

    Error: apiVersion "certmanager.k8s.io/v1alpha1" in templates/cluster-issuer.yaml is not available

Then you're seeing an error about Kubernetes not able to determine which order things get installed in. To work around
this, move that file out of the pay, perform the install/upgrade, then put it back and upgrade again to create it.
