# Background

[code-server](https://github.com/cdr/code-server) implements an instance of the [VS Code IDE](https://github.com/Microsoft/vscode) that can be served from the cloud, accessible via a browser. This repository demonstrates a secure, persistent and multi-environment deployment of code-server onto Google Cloud Platform. 

Some key features of this deployment are:

- **Persistence**
  - [StatefulSet](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/) (single replica) configured per environment to ensure a stable and persistent mapping of underlying state storage on [Persistent Disk](https://cloud.google.com/persistent-disk).
  - This ensures that password, workspace configuration, extensions, and any other state will survive a restart of the underlying container.
- **Multi-Environment Support**
  - Since code-server _currently_ supports only a single-user, there is a need to provision distinct StatefulSet, Managed Certificate and Ingress rule configuration per user environment.
  - This is facilitated by the use of [Jinja2](https://jinja.palletsprojects.com/en/2.11.x/) templating, to provide an automated and declarative approach to rolling-out user environments.
- **Security**
  - Security is enforced with two layers of authentication at both perimeter and container.
  - Ingress layer: Google Identity Aware Proxy ([IAP](https://cloud.google.com/iap)) is utilized to restrict access to environments at the perimeter, based on user identity (OAuth). This prevents malicious actors from attempting to compromise the code-server container.
  - Container layer: HTTP Basic Authentication with automatically generated strong password that persists container restarts.
    



# Prerequisites

- Access to a a freshly created GCP project with Owner/Editor permissions (these ["primitive"](https://cloud.google.com/iam/docs/understanding-roles#primitive_role_definitions) roles are required to enable APIs)
- Admin access to a DNS domain (i.e. ability to add resource records to this domain)
- Installation of [Google Cloud SDK](https://cloud.google.com/sdk/install) (including kubectl component) on your workstation
- Python 3.7

# Deployment

## Configure Python Environment

Templating requires pyyaml and jinja2-cli python packages. To install these into a clean virtual environment, ensure you are in project root and run:

```bash
pip install pipenv
pipenv --python 3.7
pipenv install
```

## Enable APIs

```bash
gcloud services enable \
  compute.googleapis.com \
  container.googleapis.com \
  iap.googleapis.com
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

Note: the machine type "n1-standard-4" is used for demonstration purposes. In production deployments, select an appropriately sized machine type. This will depend on the a) number of user environments and b) the expected cpu/memory resource requirements of each environment. 

## Apply K8S Manifests

The first step is to specify the variables required for the templating engine to generate the K8S manifests. An example of this file can be found [here](demo.yaml).

Example:

```
domain: yoleus.com
static_ip_name: code-server
home_drive_size: 1Gi
environments:
- djrut-sandbox
- hal-sandbox
```
Generate and apply the manifests as follows, with the $CONFIG variable pointing your configuration yaml:

```bash
export CONFIG=demo.yaml
for template in $(ls templates); do                                                                                                                
  pipenv run jinja2 templates/$template $CONFIG | kubectl apply -f -
done
```

**Note:** It is normal for the ingress to take up to 20 minutes to fully provision, so this is a great opportunity to go grab some coffee!

## Troubleshooting

* Confirm Pods have "Running" status, e.g.:

```bash
$ kubectl get po                                                                                                                    [master]
NAME              READY   STATUS    RESTARTS   AGE
djrut-sandbox-0   2/2     Running   0          3m37s
hal-sandbox-0     2/2     Running   0          3m37s
```
* Confirm ManagedCertificates has status "Active", e.g.:

```bash
$ kubectl describe managedcertificate|egrep 'Domain:|Status'                                                                             
Status:
  Certificate Status:  Active
  Domain Status:
    Domain:     djrut-sandbox.yoleus.com
    Status:     Active
Status:
  Certificate Status:  Active
  Domain Status:
    Domain:     hal-sandbox.yoleus.com
    Status:     Active
```

* Confirm backends are healthy

```bash
$ kubectl describe ingress|grep ingress.kubernetes.io/backends:                                                                     
  ingress.kubernetes.io/backends:                    {"k8s-be-30759--ea814a4473ad1246":"HEALTHY","k8s-be-32338--ea814a4473ad1246":"HEALTHY","k8s-be-32558--ea814a4473ad1246":"HEALTHY"}
```

* Confirm ingress is healthy (address field is populated):

```bash
$ kubectl get ingress                                                                                                               [master]
NAME          HOSTS                                             ADDRESS         PORTS   AGE
code-server   djrut-sandbox.yoleus.com,hal-sandbox.yoleus.com   34.120.27.101   80      17m
```

## Increase backend service timeout

By default, GCLB terminates idle connections after 30s, which is problematic for applications that use long-lived web-sockets. It is recommended to increase this timeout to 24hrs, which can be done programmatically as follows:

```bash
declare -a backends=($(gcloud compute backend-services list --format="value(name)"))
for backend in $backends; do                                                                                                                       
  gcloud compute backend-services update $backend --global --timeout=86400 # Set timeout to 24hrs
done
```

## Obtain code-server password

The first time a new statefulset is provisioned, code-server automatically generates a password and stores this in the config.yaml file. This password persists container restarts.

Obtain the password from each environment like this:

```bash
export ENVIRONMENT=hal-sandbox                                                                                                    
kubectl exec -it $ENVIRONMENT-0 -c $ENVIRONMENT cat /home/coder/.config/code-server/config.yaml|grep ^password|cut -d: -f2
``` 

## Secure perimeter with IAP

Follow the steps [here](https://cloud.google.com/iap/docs/enabling-kubernetes-howto) to configure and enable IAP.


When reaching the "Configuring BackendConfig" section, note that when applying the example yaml for the BackendConfig as specified, this currently results in the error: 

```
error: unable to recognize "iap/backend_config.yaml": no matches for kind "BackendConfig" in version "cloud.google.com/v1"
```
The fix is to replace the apiVersion with "cloud.google.com/v1beta1". The example manifest in this repo [iap/backend_config.yaml](iap/backend_config.yaml) has already been patched to this effect.

Apply as follows:

```bash
kubectl apply -f iap/backend_config.yaml
```

When reaching the step "associate Service ports with your BackendConfig", there is patch file already provided in [iap/iap_patch.yaml](iap/iap_patch.yaml)

Apply as follows, replacing the service name as needed:

```bash
kubectl patch service djrut-sandbox --patch "$(cat iap/iap_patch.yaml)"
```












