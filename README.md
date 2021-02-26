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

### Configure Ingress

In `k8s/django-ingress.yaml`:
- Replace `CDCS_HOSTNAME` by the hostname to be used
- Replace `CDCS_TLS` by name of the secret containing the TLS certificate and private key (see https://kubernetes.io/docs/concepts/services-networking/ingress/#tls)

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



