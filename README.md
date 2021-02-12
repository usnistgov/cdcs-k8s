# CDCS K8s

## FIXME
- cdcs image is hardcoded in the django-deployment: `wipp-registry:k8s`
- django-ingress has `xxx.xxx.xxx.xxx` instead of the local machine's ip
- mongo-init-config-map has hardcoded values USERNAME/PASSWORD


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

## TODO

- look into namespaces for several CDCS instances on the same cluster
- use envFrom secret for mongo/postgres instead of valueFrom



