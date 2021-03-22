# Kubernetes Installation for WIPP-Registry
This repository contains Kubernetes deployment instructions for the [WIPP-Registry project](https://github.com/usnistgov/WIPP-Registry.git)

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
| MONGO_USER            | User for MongoDB (should be different from `MONGO_INITDB_ROOT_USERNAME`) |
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
| MONGO_USER            | User for MongoDB |
| MONGO_PASS            | User password for MongoDB |
| MONGO_DB              | Name of the Mongo database (e.g. cdcs) |
| POSTGRES_HOST         | Hostname for Postgres (name of Postgres service) |
| POSTGRES_USER         | User for Postgres |
| POSTGRES_PASS         | User password for Postgres |
| POSTGRES_DB           | Name of the Postgres database (e.g. cdcs) |
| REDIS_HOST            | Hostname for Redis (name of Redis service) |
| REDIS_PASS            | Password for Redis |
| SERVER_URI            | URI of server |
| SERVER_NAME           | Name of the server, used to distinguish instances in federated queries (e.g. {INSTITUTION}-WIPP or {INSTITUTION}-{CUSTOM-WIPP-NAME}) |
| ALLOWED_HOSTS         | Comma-separated list of hosts (e.g. ALLOWED_HOSTS=127.0.0.1,localhost), see [Allowed hosts](https://docs.djangoproject.com/en/2.2/ref/settings/#allowed-hosts) |
| DJANGO_SECRET_KEY     | [Secret Key](https://docs.djangoproject.com/en/2.2/howto/deployment/checklist/#secret-key) for Django (should be a "large random value") |
| SETTINGS              | Settings file to use during deployment, default value is `settings` ([more info in the Settings section](#settings))|
| MONITORING_SERVER_URI | (optional) URI of an APM server for monitoring |
| SAML_METADATA_CONF_URL| (optional) URI of a SAML metadata configuration |
| SAML_CREATE_USER      | (optional) Determines if a new Django user should be created for new users |
| SAML_ATTRIBUTES_MAP_EMAIL| (optional) Mapping of Django user attributes to SAML2 user attribute - email |
| SAML_ATTRIBUTES_MAP_USERNAME| (optional) Mapping of Django user attributes to SAML2 user attribute - username |
| SAML_ATTRIBUTES_MAP_FIRSTNAME| (optional) Mapping of Django user attributes to SAML2 user attribute - firstname |
| SAML_ATTRIBUTES_MAP_LASTNAME| (optional) Mapping of Django user attributes to SAML2 user attribute - lastname |
| SAML_ASSERTION_URL| (optional) A URL to validate incoming SAML responses against |
| SAML_ENTITY_ID| (optional) The optional entity ID string to be passed in the 'Issuer' element of authn request, if required by the IDP |
| SAML_NAME_ID_FORMAT| (optional) Set to the string 'None', to exclude sending the 'Format' property of the 'NameIDPolicy' element in authn requests. Default value if not specified is 'urn:oasis:names:tc:SAML:2.0:nameid-format:transient' |
| SAML_USE_JWT| (optional) JWT authentication - False |
| SAML_CLIENT_SETTINGS| (optional) Client settings - False |

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
Create secrets file `superuser-secrets`, by copying `superuser-secrets-example` file from the `init` folder without the `-example` suffix.

| Variable | Description |
| ----------- | ----------- |
| SUPERUSER_USERNAME        | Username for superuser |
| SUPERUSER_PASSWORD        | User password for superuser |
| SUPERUSER_EMAIL           | Email address for superuser (optional) |

Then run:
```shell
kubectl create secret generic cdcs-superuser --from-env-file=superuser-secrets
```

#### Run superuser creation Job

```shell
kubectl apply -f init/create-superuser.yaml
```

#### Settings

Starting from MDCS/NMRR 2.14, repositories of these two projects will
have settings ready for deployment (not production).

The deployment can be further customized by mounting additional settings
to the deployed containers:
- **Option 1 (default):** Use settings from the image. 
    - set the `SETTINGS` variable to `settings`.
- **Option 2**: Use default settings from the CDCS image and customize
them. Custom settings can be used to override default settings or add additional settngs. For example:
    - Create a config map containing a `custom_settings.py` entry for the custom settings,
    - Update the `django-deployment.yml` file and create a volume for the config map that will
        mount the settings at the following location:
        ```
        /srv/curator/nmrr/custom_settings.py
        ```
    - set the `SETTINGS` variable to `custom_settings` to use the custom settings

For more information about production deployment of a Django project,
please check the [Deployment Checklist](https://docs.djangoproject.com/en/2.2/howto/deployment/checklist/#deployment-checklist)

#### SAML2 authentication

For SAML-based authentication:
- uncomment and set `SAML_*` variables in `cdcs-secret` file
- change the image in `django-deployment.yaml` from `wipp/wipp-registry:{version}` to `wipp/wipp-registry:{version}-saml` (e.g. `wipp/wipp-registry:1.1.0-saml`)

More information about the SMAL2 configuration can be found in the [django-saml2-auth module](https://github.com/fangli/django-saml2-auth), sample configuration for Keycloak is provided in the `cdcs-secret-example` file.

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

## Disclaimer

[NIST Disclaimer](LICENSE.md)

