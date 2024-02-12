# Install with manifests

Follow the instructions below to deploy the CDCS in a Kubernetes cluster
using YAML manifests.

## Prerequisites
* [Kubernetes Cluster](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)
* [Ingress Nginx](https://kubernetes.github.io/ingress-nginx/deploy/)

In single node deployment, allow deployment on the main node by typing:
```shell
kubectl taint nodes --all node-role.kubernetes.io/control-plane:NoSchedule-
```

To make use of the NFS volumes provided with this repository, make sure to have an NFS 
server available with two separate mounts (one for MongoDB, the other one for PostgreSQL).

## Deployment overview

The CDCS application is packaged in a container exposing port 8000 to the pod. In the 
same pod, a Nginx sidecar is used to distribute the web application and all static 
files (images, scripts, etc.). At this stage, using an HTTPS certificate is not 
necessary, thus the Nginx sidecar only exposes port 80.

To make the service available within the cluster, a ClusterIP service is used and make 
the CDCS deployment available at port 8080.

A Nginx Ingress service, serving as a load balancer, protects the whole stack with an 
HTTPS certificate and links the ClusterIP service on port 8080 to port 443 (default 
HTTPS port).

In a bare metal deployment, a NodePort service is then used to forward a port between 
30000 and 32767 of the cluster nodes to port 443 of the Ingress Nginx. See 
[Ingress Nginx NodePort](#ingress-nginx-nodeport-configurations-optional-bare-metal-only) for more information.

## Environment variables
### Create Secrets
#### From files
Create secrets files, by copying `*-secrets-example` files from the `config` folder in 
new files without the `-example` suffix. Then run:
```shell
kubectl create secret generic mongodb --from-env-file=./config/mongo-secrets
kubectl create secret generic redis --from-env-file=./config/redis-secrets
kubectl create secret generic postgres --from-env-file=./config/postgres-secrets
kubectl create secret generic cdcs --from-env-file=./config/cdcs-secrets
```

#### From literal
PostgreSQL Example:
```shell
kubectl create secret generic postgres \
    --from-literal="POSTGRES_USER=${PG_USER}" \
    --from-literal="POSTGRES_PASSWORD=${PG_PASS}" \
    --from-literal="POSTGRES_DB=${PG_DB}"
```

To verify that the secrets have been properly created, the following command can be used:
```shell
kubectl get secrets
```

### Create ConfigMap
#### From file
Create config files, by copying `*-config-example` files from the `config` folder in new 
files without the `-example` suffix. Then run:
```shell
kubectl create configmap cdcs --from-env-file=./config/cdcs-config
```

### Description of environment variables

Below is the list of environment variables to set in the `*-secrets` and `*-config` 
files and their description.

#### mongo-secrets

| Variable                   | Description                                                              |
|----------------------------|--------------------------------------------------------------------------|
| MONGO_INITDB_ROOT_USERNAME | Admin user for MongoDB (should be different from `MONGO_USER`)           |
| MONGO_INITDB_ROOT_PASSWORD | Admin password for MongoDB                                               |
| MONGO_USER                 | User for MongoDB (should be different from `MONGO_INITDB_ROOT_USERNAME`) |
| MONGO_PASS                 | User password for MongoDB                                                |
| MONGO_INITDB_DATABASE      | Name of the Mongo database (e.g. cdcs)                                   |

#### postgres-secrets

| Variable          | Description                                 |
|-------------------|---------------------------------------------|
| POSTGRES_USER     | User for PostgreSQL                         |
| POSTGRES_PASSWORD | Password for PostgreSQL                     |
| POSTGRES_DB       | Name of the PostgreSQL database (e.g. cdcs) |

#### redis-secrets

| Variable   | Description        |
|------------|--------------------|
| REDIS_PASS | Password for Redis |

#### cdcs-secrets

| Variable          | Description                                                                                                                              |
|-------------------|------------------------------------------------------------------------------------------------------------------------------------------|
| MONGO_USER        | User for MongoDB                                                                                                                         |
| MONGO_PASS        | Password for MongoDB                                                                                                                     |
| POSTGRES_USER     | User for PostgreSQL                                                                                                                      |
| POSTGRES_PASS     | Password for PostgreSQL                                                                                                                  |
| REDIS_PASS        | Password for Redis                                                                                                                       |
| DJANGO_SECRET_KEY | [Secret Key](https://docs.djangoproject.com/en/4.2/howto/deployment/checklist/#secret-key) for Django (should be a "large random value") |

#### cdcs-config

| Variable              | Description                                                                                                                                                    |
|-----------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------|
| PROJECT_NAME          | Name of the CDCS/Django project to build (e.g. mdcs, nmrr)                                                                                                     |
| MONGO_HOST            | Hostname for MongoDB (name of MongoDB service)                                                                                                                 |
| MONGO_DB              | Name of the Mongo database (e.g. cdcs)                                                                                                                         |
| POSTGRES_HOST         | Hostname for PostgreSQL (name of PostgreSQL service)                                                                                                           |
| POSTGRES_DB           | Name of the PostgreSQL database (e.g. cdcs)                                                                                                                    |
| REDIS_HOST            | Hostname for Redis (name of Redis service)                                                                                                                     |
| SERVER_URI            | URI of server                                                                                                                                                  |
| SERVER_NAME           | Name of the server, used to distinguish instances in federated queries (e.g. {INSTITUTION} or {INSTITUTION}-{CUSTOM-INSTANCE-NAME})                            |
| ALLOWED_HOSTS         | Comma-separated list of hosts (e.g. ALLOWED_HOSTS=127.0.0.1,localhost), see [Allowed hosts](https://docs.djangoproject.com/en/4.2/ref/settings/#allowed-hosts) |
| SETTINGS              | Settings file to use during deployment, default value is `settings` ([more info in the Settings section](#settings))                                           |
| MONITORING_SERVER_URI | (optional) URI of an APM server for monitoring                                                                                                                 |
| PROCESSES             | (optional) Number of Gunicorn workers to start (default `workers=cpu_count() * 2 + 1`)                                                                         |
| THREADS               | (optional) Number of Gunicorn threads per process/worker (default 8)                                                                                           |

## Configure Ingress

In `k8s/django-ingress.yaml`:
- Replace `CDCS_HOSTNAME` by the hostname to be used
- Replace `CDCS_TLS` by name of the secret containing the TLS certificate and private key
  (see https://kubernetes.io/docs/concepts/services-networking/ingress/#tls)

## Configure volumes
The volumes for PostgreSQL, MongoDB and the Django media folder need to be sized and then
deployed. To deploy the necessary volumes, choose only one of the method below: HostPath
or NFS. See https://kubernetes.io/docs/concepts/storage/volumes/ for more information
and options about volumes.

### Size configurations
Each of the volumes will need to be appropriately sized according to the datasets to be
hosted and deployment environments. The following commands will configure the three 
volumes to have a size of 10Gi.
```shell
sed -i -e "s;MEDIA_STORAGE_SIZE;10Gi;g" \
  ./volumes/**/*.yaml
sed -i -e "s;MONGO_STORAGE_SIZE;10Gi;g" \
  ./volumes/**/*.yaml
sed -i -e "s;POSTGRES_STORAGE_SIZE;10Gi;g" \
  ./volumes/**/*.yaml
sed -i -e "s;REDIS_STORAGE_SIZE;10Gi;g" \
  ./volumes/**/*.yaml
```

### Using HostPath (single node deployments ONLY!)
For single node deployment, a HostPath deployment, although not recommended, can be 
enough. First, you must specify the desired path to the volume, using:
```shell
sed -i -e "s;MONGO_VOLUME_PATH;/path/to/my/mongo_data;g" \
  ./volumes/local/mongo-volume-claim.yaml
sed -i -e "s;POSTGRES_VOLUME_PATH;/path/to/my/postgres_data;g" \
  ./volumes/local/postgres-volume-claim.yaml
sed -i -e "s;MEDIA_VOLUME_PATH;/path/to/my/media_data;g" \
  ./volumes/local/media-volume-claim.yaml
sed -i -e "s;REDIS_VOLUME_PATH;/path/to/my/redis_data;g" \
  ./volumes/local/redis-volume-claim.yaml
```

To deploy the local volumes for the database, enter the following command:
```shell
kubectl apply -f ./volumes/local/
```

**WARNING:** Using HostPath in a multi-node deployment can lead to data loss and 
uncontrolled behavior!

### Using NFS
For multi-nodes deployment, several options make volumes available to the cluster. As an 
example, NFS claims have been implemented in this repository. Five (5) variables need 
to be set up in this configuration: `MONGO_VOLUME_PATH`, `POSTGRES_VOLUME_PATH`, `REDIS_VOLUME_PATH` 
`MEDIA_VOLUME_PATH` and `NFS_SERVER_IP`. Here are the commands to configure them:
```shell
# Change the path to the Mongo and PostgreSQL NFS shared folders.
sed -i -e "s;MONGO_VOLUME_PATH;/path/to/my/mongo_data;g" \
  ./volumes/nfs/mongo-volume-claim.yaml
sed -i -e "s;POSTGRES_VOLUME_PATH;/path/to/my/postgres_data;g" \
  ./volumes/nfs/postgres-volume-claim.yaml
sed -i -e "s;MEDIA_VOLUME_PATH;/path/to/my/media_data;g" \
  ./volumes/nfs/media-volume-claim.yaml
sed -i -e "s;REDIS_VOLUME_PATH;/path/to/my/redis_data;g" \
  ./volumes/nfs/redis-volume-claim.yaml
# Configure the NFS server IP for both file.
sed -i -e "s;NFS_SERVER_IP;1.2.3.4;g" ./volumes/nfs/*-volume-claim.yaml
```

To deploy the configured volumes on the cluster, type the following command: 
```shell
kubectl apply -f ./volumes/nfs/
```

### Checking volume creation

Ensure there are no errors with the volume creation by typing:
```shell
kubectl get pvc | grep -i -v bound
```
Volume claims named `cdcs-pvc-media`, `cdcs-pvc-mongo`, `cdcs-pvc-redis` 
and `cdcs-pvc-postgres` should not appear in the list returned by the command.

## Deployment

### Generate self-signed SSL certificate for the Ingress (optional)
A certificate is necessary for the Nginx Ingress to work properly. If you don't have one
available, generate a self-signed one (not secure!) using this script:
```shell
# Script parameters
HOST="<host.example.com>" 
KEY_FILE="<key-file.pem>" 
CERT_FILE="<cert-file.pem>"

# Generate certificate
openssl req -x509 -nodes -days 365 \
  -newkey rsa:2048 \
  -keyout ${KEY_FILE} \
  -out ${CERT_FILE} \
  -subj "/CN=${HOST}/O=${HOST}" \
  -addext "subjectAltName = DNS:${HOST}"
```

Once the certificate is available, the following command will register a secret that will
be used by the Nginx Ingress.
```shell
# Add the certificate as a secret
kubectl create secret tls cdcs-cert --key ${KEY_FILE} --cert ${CERT_FILE}
```

### Ingress Nginx NodePort configurations (optional, bare metal only).

When installing on bare metal, additional configurations of the Ingress Nginx
are necessary. To know where the application will be available, a port between 30000 and
32767 needs to be chosen. Type the following commands to configure the application to be
available on port 32000:

```shell
sed -i -e "s;HTTPS_PORT;32000;g" \
  ./k8s-baremetal/django-ingress.yaml
kubectl apply -f ./k8s-baremetal
```

### Deploy the stack
In `k8s/django-deployment.yaml` and `init/create-superuser.yaml`:
- Replace `CDCS_IMAGE` by the CDCS container image to be used

The following command will deploy the entire stack:
```shell
kubectl apply -f ./k8s
```

Before continuing, make sure the stack is properly deployed. The following command can 
help diagnose any problem from the deployment
```shell
# Check that all deployments are healthy.
kubectl get deploy
# Check that all pods are healthy.
kubectl get pods
# Check logs from a pod that might be unhealthy.
kubectl logs -f ${problem_pod}
```

### Create a superuser
The superuser is the first user that will be added to the CDCS. This is the
main administrator on the platform. Once it has been created, more users
can be added using the web interface. Wait for the CDCS server to start, then:

Create secrets file `superuser-secrets`, by copying `superuser-secrets-example` file 
from the `init` folder without the `-example` suffix.

| Variable           | Description                            |
|--------------------|----------------------------------------|
| SUPERUSER_USERNAME | Username for superuser                 |
| SUPERUSER_PASSWORD | User password for superuser            |
| SUPERUSER_EMAIL    | Email address for superuser (optional) |

To create the secret, run:
```shell
kubectl create secret generic cdcs-superuser --from-env-file=./init/superuser-secrets
```

The superuser can then be created using:
```shell
kubectl apply -f init/create-superuser.yaml
```

## Settings
Starting from MDCS/NMRR 2.14, repositories of these two projects will
have settings ready for deployment (not production).

The deployment can be further customized by mounting additional settings
to the deployed containers:
- **Option 1 (default):** Use settings from the image. 
    - set the `SETTINGS` variable to `settings`.
- **Option 2**: Use default settings from the CDCS image and customize them. Custom 
settings can be used to override default settings or add additional settings. For 
example:
    - Create a config map containing a `custom_settings.py` entry for the custom settings,
    - Update the `django-deployment.yml` file and create a volume for the config map that 
    will
        mount the settings at the following location:
        ```
        /srv/curator/nmrr/custom_settings.py
        ```
    - set the `SETTINGS` variable to `custom_settings` to use the custom settings

For more information about production deployment of a Django project,
please check the [Deployment Checklist](https://docs.djangoproject.com/en/4.2/howto/deployment/checklist/#deployment-checklist)

## SAML2 authentication

For SAML-based authentication:
- add SAML2 environment variables from the [CDCS Docker SAML2 documentation](https://github.com/usnistgov/cdcs-docker/tree/master#saml2)
in the file `config/cdcs-config`.
- a sample configuration for Keycloak is provided in the `cdcs-config-example` file.

## Scale deployment
If the performance of the CDCS needs to be improved, it is possible to scale up the 
number of CDCS django pods replicas.

To scale the deployment run:
```shell
kubectl scale --replicas=${replica_count} deployment/cdcs-django-deployment
```

Note: `${replica_count}` should be higher than 1 and lower or equal to the number of 
available nodes. If performance issues persist, please contact the CDCS development team. 

## Remove deployment
To remove the CDCS stack from the cluster, the following commands can be used:
```shell
# Delete CDCS stack.
kubectl delete -f ./k8s
# Delete the volumes (replace `local` by `nfs` if an NFS server is used)
kubectl delete -f ./volumes/local
```
