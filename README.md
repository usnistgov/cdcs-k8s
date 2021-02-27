# CDCS K8s

## Installation Notes

### Prerequisites
- Kubernetes Cluster
- Ingress Nginx: https://kubernetes.github.io/ingress-nginx/deploy/

### Create Secrets
#### From files
Create secrets files, by copying files from the `secrets` folder in new files without the `-example` suffix.
Then run:
```shell
kubectl create secret generic mongodb --from-env-file=mongo-secrets
kubectl create secret generic redis --from-env-file=redis-secrets
kubectl create secret generic postgres --from-env-file=postgres-secrets
kubectl create secret generic cdcs --from-env-file=cdcs-secrets
```
#### From literal
Postgres Example:
```shell
kubectl create secret generic postgres \
--from-literal="POSTGRES_USER=<user>" \
--from-literal="POSTGRES_PASSWORD=<password>" \
--from-literal="POSTGRES_DB=<db>"
```

#### Description of environment variables

Below is the list of environment variables to set in the `*-secrets` files and their description.

##### mongo-secrets

| Variable | Description |
| ----------- | ----------- |
| MONGO_INITDB_ROOT_USERNAME      | Admin user for MongoDB (should be different from `MONGO_USER`) |
| MONGO_INITDB_ROOT_PASSWORD      | Admin password for MongoDB |
| MONGO_USER            | User for MongoDB (should be different from `MONGO_ADMIN_USER`) |
| MONGO_PASS            | User password for MongoDB |
| MONGO_INITDB_DATABASE              | Name of the Mongo database (e.g. cdcs) |

##### postgres-secrets

| Variable | Description |
| ----------- | ----------- |
| POSTGRES_USER         | User for Postgres |
| POSTGRES_PASSWORD     | User password for Postgres |
| POSTGRES_DB           | Name of the Postgres database (e.g. cdcs) |

##### redis-secrets

| Variable | Description |
| ----------- | ----------- |
| REDIS_PASS            | Password for Redis |

##### cdcs-secrets

| Variable | Description |
| ----------- | ----------- |
| MONGO_HOST            | Hostname for MongoDB (name of MongoDB service) |
| MONGO_ADMIN_USER      | Admin user for MongoDB (should be different from `MONGO_USER`) |
| MONGO_ADMIN_PASS      | Admin password for MongoDB |
| MONGO_USER            | User for MongoDB (should be different from `MONGO_ADMIN_USER`) |
| MONGO_PASS            | User password for MongoDB |
| MONGO_DB              | Name of the Mongo database (e.g. cdcs) |
| POSTGRES_HOST         | Hostname for Postgres (name of Postgres service) |
| POSTGRES_USER         | User for Postgres |
| POSTGRES_PASS         | User password for Postgres |
| POSTGRES_DB           | Name of the Postgres database (e.g. cdcs) |
| REDIS_HOST            | Hostname for Redis (name of Redis service) |
| REDIS_PASS            | Password for Redis |
| SERVER_URI            | URI of server |
| SERVER_NAME           | Name of the server (e.g. MDCS, WIPP-Registry, ...) |
| ALLOWED_HOSTS         | [Allowed hosts](https://docs.djangoproject.com/en/2.2/ref/settings/#allowed-hosts) |
| DJANGO_SECRET_KEY     | [Secret Key](https://docs.djangoproject.com/en/2.2/howto/deployment/checklist/#secret-key) for Django (should be a "large random value") |

### Configure Ingress

In `k8s/django-ingress.yaml`:
- Replace `CDCS_HOSTNAME` by the hostname to be used
- Replace `CDCS_TLS` by name of the secret containing the TLS certificate and private key (see https://kubernetes.io/docs/concepts/services-networking/ingress/#tls)

### Deploy

```shell
kubectl apply -f k8s/
```

### Create a superuser

The superuser is the first user that will be added to the CDCS. This is the
main administrator on the platform. Once it has been created, more users
can be added using the web interface. Wait for the CDCS server to start, then:

#### Create cdcs-superuser secret
Create secrets file `cdcs-superuser`, by copying `cdcs-superuser-example` file from the `init` folder without the `-example` suffix.

| Variable | Description |
| ----------- | ----------- |
| SUPERUSER_USERNAME        | Username for superuser |
| SUPERUSER_PASSWORD        | User password for superuser |
| SUPERUSER_EMAIL           | Email address for superuser (optional) |

Then run:
```shell
kubectl create secret generic cdcs-superuser --from-env-file=cdcs-superuser
```

#### Run superuser creation Job

```shell
kubectl apply -f init/create-superuser.yaml
```

### Troubleshoot

#### Local deployment with HTTP

Update `django-cluster-ip-service.yaml`, and change the `type` to:
```yaml
type: NodePort
```
After deployment, run:
```shell
kubectl get services/django-cluster-ip-service -o go-template='{{(index .spec.ports 0).nodePort}}'
```

This command will return a port number: DJANGO_PORT.
You can then access the instance using your IP address, and the port (e.g. http://xxx.xxx.xxx.xxx:DJANGO_PORT).

#### HTTPS deployment with self-signed certificate

If you are using a self-signed certificate or certificate bundle that contains a self-signed certificate in the chain, the certificate needs to be mounted inside of the cdcs container and the `REQUESTS_CA_BUNDLE` environment variable needs to be set.

Create configmap from certificate bundle:
```
kubectl create configmap cdcs-capemstore --from-file=cdcs.pem 
```

Modify `k8s/django-deployment.yaml` to mount the certificate and set the `REQUESTS_CA_BUNDLE` environment variable:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cdcs-django-deployment
spec:
  ...
  template:
    ...
    spec:
      containers:
      - image: wipp-registry:k8s
        name: cdcs
        ...
        volumeMounts:
        - name: ca-pemstore
          mountPath: /etc/ssl/certs/cdcs.pem
          subPath: cdcs.pem
          readOnly: false
        env:
        - name: REQUESTS_CA_BUNDLE
          value: /etc/ssl/certs/cdcs.pem
        ...
      ...
      volumes:
      - name: ca-pemstore
        configMap:
          name: cdcs-capemstore
```



