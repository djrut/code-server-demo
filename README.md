# Background
---

code-server implements an instance of VS Code IDE that can be served from the cloud, accessible via a browser. This repository demonstrates a secure, persistent and multi-environment capable deployment of code-server onto Google Cloud Platform. 

Some key features of this deployment are:

- **Persistence**
  - StatefulSet (single replica) configured per environment to ensure a stable and persistent mapping of underlying state storage on Persistent Disk.
  - This ensures that password, workspace configuration, extensions, and any other state will survive a restart of the underlying pod.
- **Multi-Environment Support**
  - Since code-server _currently_ supports only a single-user, there is a need to provision distinct StatefulSet, Managed Certificate and Ingress rule configuration per user environment.
  - This is facilitated by the use of Jinja2 templating, to provide an automated and declarative approach to rolling-out user environments.
- **Security**
  - Security is enforced with two layers of authentication at both perimeter and container.
  - Ingress layer: Google Identity Aware Proxy (IAP) is utilized to restrict access to environments at the perimeter, based on user identity (OAuth). This prevents malicious actors from attempting to compromise the code-server container.
  - Container layer: HTTP Basic Authentication.
    



# Prerequisites
---

- Access to a a freshly created GCP project with Owner/Editor permissions (required to enable APIs)
- Admin access to a DNS domain (i.e. ability to add resource records to this domain)
- gcloud SDK installed and configured with the project above
- kubectl
- Python 3.7
- Pipenv

# Deployment
---
## Configure Python Environment

Templating requires pyyaml and jinja2-cli python packages. To install these into a clean virtual environment, ensure you are in project root and run:

```bash
pip install pipenv
pipenv --python 3.7
pipenv install
```

## Enable APIs

```bash
gcloud services enable compute.googleapis.com container.googleapis.com
```

## Reserve static IP

```bash
gcloud compute addresses create code-server \                                                                                                      [master]
    --global \
    --ip-version ipv4
```

Make a note of the IP address (used for DNS):

```bash
gcloud compute addresses list \
    --filter="name=code-server" \
    --format="value(address)"
```

## Add DNS Resource Records

Using the address generated above, populate your domain's DNS zone file with A records for each environment, for example:

```
djrut-sandbox   IN    A    34.120.27.101
hal-sandbox     IN    A    34.120.27.101
...
```


## Provision K8S Cluster

```bash
gke clusters create code-server-demo \
    --no-enable-autoupgrade \
    --region=us-central1 \
    --scopes https://www.googleapis.com/auth/cloud-platform \
    --num-nodes=1 \
    --enable-ip-alias \
    --machine-type=n1-standard-4
```

## Apply K8S Manifests

```bash
for template in $(ls templates); do                                                                                                                
  pipenv run jinja2 templates/$template demo.yaml | kubectl apply -f -
done
```


## Increase backend service timeout

By default, GCLB terminates idle connections after 30s, which is problematic for applications that use long-lived web-sockets. It is recommended to increase this timeout to 24hrs, which can be done programmatically as follows:

```bash
declare -a backends=($(gcloud compute backend-services list|grep -e '^k8s-'|cut -d' ' -f1))
for backend in $backends; do                                                                                                                       
  gcloud compute backend-services update $backend --global --timeout=86400 # Set timeout to 24hrs
done
```

## Optional: Secure perimeter with IAP






