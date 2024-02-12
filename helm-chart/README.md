# Install with Helm

## Prerequisites
* [Kubernetes Cluster](https://kubernetes.io/docs/setup/)
* [Helm](https://helm.sh/docs/intro/install/)
* [Ingress Nginx](https://kubernetes.github.io/ingress-nginx/deploy/)
* PV provisioner support in the underlying infrastructure (or see the [Volumes](#volumes) section to use existing claims)

## Get CDCS dependencies

Add the bitnami repository to pull the charts of CDCS dependencies:
```commandline
helm repo add bitnami https://charts.bitnami.com/bitnami
```

Then pull the charts:
```commandline
helm dependency build
```

## Quick Start

To install the CDCS and its dependencies, run the command below. By default, it will use the 
basic configuration from the `values.yaml` file provided with this repository. See options below to 
override the basic configuration with your own.  

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

Once the chart is installed, you should get a message telling you how to access the web server,
such as:

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

## MongoDB Secrets

Passwords for MongoDB can be stored in a secret. To do so, MongoDB requires specific keys and
changes to the `values.yaml` that will be described in this section.

### Create a secret
First, to create a secret for MongoDB, copy the file `mongo-secrets-example` to a new 
file called `mongo-secrets` and set the two variables `mongodb-root-password` and `mongodb-passwords`:

Example:
```
mongodb-root-password=mongo_admin_pass
mongodb-passwords=mongo_pass
```

Create a secret from the environment file with the following command:
```commandline
kubectl create secret generic [SECRET] --from-env-file=./config/mongo-secrets --namespace [NAMESPACE] 
```

Example:
```commandline
kubectl create secret generic mongodb --from-env-file=./config/mongo-secrets --namespace mdcs-test 
```

### Edit values.yaml

Then edit the `values.yaml` or your own override file, to use the secret:
- Comment `auth.passwords` and `auth.rootPassword`
- Uncomment `existingSecret` and put the name of the secret previously created.

Example:
```yml
auth:
#    passwords:
#      - mongo_pass
#    rootPassword: mongo_admin_pass
    # Set the name of the secret created by the previous command
    existingSecret: "mongodb"
```

## Volumes

To use existing PVC, first create the PVC by following these [instructions](../manifests/README.md#configure-volumes). Existing claims need to be created in
the same namespace as the helm chart release. You can follow the
[instructions to create a namespace](https://kubernetes.io/docs/tasks/administer-cluster/namespaces-walkthrough/#create-new-namespaces) 
prior to creating the PVC.

## Ingress TLS Secret

Instructions to create a TLS secret for ingress can be found at this [link](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_create/kubectl_create_secret_tls/).
Once the TLS secret has been created, it can be set using the following field of the `values.yml` file:

Example:
```yml
cdcs:
  ingress:
    tls:
      - secretName: "cdcs-cert"
```

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

### PostgreSQL

Read replicas can be enabled for the PostgreSQL databases by setting
 `postgresql.architecture=replication` and by configuring the `postgresql.readReplicas` fields:

The Django server needs to be properly configured to use the replicas. See the 
[Django database routing](https://docs.djangoproject.com/en/4.2/topics/db/multi-db/#automatic-database-routing)
documentation for more information.

Example:
```commandline
helm install -n mdcs-test --create-namespace mdcs-helm-test . --set postgresql.architecture=replication --set postgresql.readReplicas.replicaCount=1
```

## Uninstall the chart

To uninstall the chart, type the following command:
```commandline
helm uninstall -n [NAMESPACE] [CHART] 
```

Example:
```commandline
helm uninstall -n mdcs-test mdcs-helm-test 
```

## Troubleshooting

### MongoDB chart on ARM architectures

The bitnami helm chart for MongoDB is currently not supported
on ARM-based chips (e.g. Apple M1).

### PostgreSQL and Redis charts conflicts

The version of the bitnami chart for Redis required by the CDCS is not compatible with
the version of the bitnami chart for PostgreSQL and needs to be downgraded
(See comment in [Chart.yml](Chart.yaml)).