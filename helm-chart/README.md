# Install with Helm

## Prerequisites

* [Kubernetes Cluster](https://kubernetes.io/docs/setup/)
* [Helm](https://helm.sh/docs/intro/install/)
* [Ingress Nginx](https://kubernetes.github.io/ingress-nginx/deploy/)
* PV provisioner support in the underlying infrastructure (or see the [Volumes](#volumes)
  section to use existing claims)

## Get CDCS dependencies

:warning: Changes to Bitnami catalog starting from 08/28/2025: https://github.com/bitnami/charts/issues/35164

Deploy the CDCS dependencies (PostgresSQL, Redis and optionally MongoDB): 
1) using the manifests from this repository,
2) using available helm charts or operators.

## Quick Start

To deploy the CDCS, run the command below. By default, it will use
the basic configuration from the `values.yaml` file provided with this repository. See
options below to override the basic configuration with your own.

Install the Chart:

```commandline
helm install -n [NAMESPACE] --create-namespace [RELEASE] [CHART]
```

Example:

```commandline
helm install -n mdcs-test --create-namespace mdcs-helm-test .
```

Override the default `values.yaml` by linking to a new values file using the `-f` option:

```commandline
helm install -n mdcs-test --create-namespace -f ./values.test.yaml mdcs-helm-test .
```

Override only specific fields of the `values.yaml` file, with the `--set` option:

```commandline
helm install -n mdcs-test --create-namespace mdcs-helm-test . --set cdcs.imagePullPolicy=IfNotPresent
```

The CDCS container can be configured using environment variables.
The complete list of environment variables can be found in the
[customized deployment](https://github.com/usnistgov/cdcs-docker?tab=readme-ov-file#1-customize-the-deployment)
section of the CDCS Docker documentation.
These environment variables can be set in the file `values.yaml`.

Example:

```yaml
cdcs:
  envs:
    - name: PROJECT_NAME
      value: "mdcs"
```

Once the chart is installed, you should get a message telling you how to access the web
server, such as:

```commandline
NAME: mdcs-helm-test
LAST DEPLOYED: [DATE]
NAMESPACE: mdcs-test
STATUS: deployed
REVISION: 1
NOTES:
1. The system has been deployed at the following URL:
  http://localhost
```

Once all the pods are properly started, go to the provided URL to access the system.

## Create a Superuser

The superuser is the first user that will be added to the CDCS. This is the
main administrator on the platform. Once it has been created, more users
can be added using the web interface. Wait for the CDCS server to start, then run
the following and enter the desired username, password and optional email:

```commandline
HELM_RELEASE_NAMESPACE="[NAMESPACE]" &&\
HELM_RELEASE_NAME="[RELEASE]" &&\
CDCS_POD_NAME=$(kubectl get pod --namespace $HELM_RELEASE_NAMESPACE -l component=[COMPONENT] -o jsonpath='{.items[0].metadata.name}') &&\
kubectl exec -it --namespace $HELM_RELEASE_NAMESPACE $CDCS_POD_NAME -c [CONTAINER] -- python manage.py createsuperuser
```

Example:

```commandline
HELM_RELEASE_NAMESPACE="mdcs-test" &&\
HELM_RELEASE_NAME="mdcs-helm-test" &&\
CDCS_POD_NAME=$(kubectl get pod --namespace $HELM_RELEASE_NAMESPACE -l component=cdcs-django -o jsonpath='{.items[0].metadata.name}') &&\
kubectl exec -it --namespace $HELM_RELEASE_NAMESPACE $CDCS_POD_NAME -c cdcs -- python manage.py createsuperuser
```

## Volumes

To use existing PVC, first create the PVC by following
these [instructions](../manifests/README.md#configure-volumes). Existing claims need to
be created in the same namespace as the helm chart release. You can follow the
[instructions to create a namespace](https://kubernetes.io/docs/tasks/administer-cluster/namespaces-walkthrough/#create-new-namespaces)
prior to creating the PVC.

## Ingress TLS Secret

Instructions to create a TLS secret for ingress can be found at
this [link](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_create/kubectl_create_secret_tls/).
Once the TLS secret has been created, it can be set using the following field of
the `values.yml` file:

Example:

```yml
cdcs:
  ingress:
    tls:
      - secretName: "cdcs-cert"
```

## Ingress Controller NodePort configurations (optional, bare metal only).

On bare metal configuration, follow
these [instructions](../manifests/README.md#ingress-controller-nodeport-configurations-optional-bare-metal-only)
to setup a specific port to access the application.

## Replication

### Django

Replication can be enabled for the Django web server by setting
the `cdcs.replicas` field:

```commandline
helm install -n [NAMESPACE] --create-namespace [RELEAE] [CHART] --set cdcs.replicas=[REPLICAS] 
```

Example:

```commandline
helm install -n mdcs-test --create-namespace mdcs-helm-test . --set cdcs.replicas=2
```

## Uninstall the chart

To uninstall the chart, type the following command:

```commandline
helm uninstall -n [NAMESPACE] [RELEASE] 
```

Example:

```commandline
helm uninstall -n mdcs-test mdcs-helm-test 
```