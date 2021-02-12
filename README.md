# CDCS K8s

## FIXME
- cdcs image is hardcoded in the django-deployment: `wipp-registry:k8s`
- django-ingress has `xxx.xxx.xxx.xxx` instead of the local machine's ip
- mongo-init-config-map has hardcoded values USERNAME/PASSWORD
- explore unable to reach server URI on MacOS with Docker for Mac


## Installation Notes

### Prerequisites
- Docker for Mac with Kubernetes Cluster enabled
- Ingress Nginx: https://kubernetes.github.io/ingress-nginx/deploy/#docker-for-mac

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

### Deploy

```shell
kubectl apply -f k8s/
```

## TODO

- look into namespaces for several CDCS instances on the same cluster
- use envFrom secret for mongo/postgres instead of valueFrom



