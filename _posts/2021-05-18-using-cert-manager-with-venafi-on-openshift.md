---
layout: post
title: "Using cert-manager with Venafi on OpenShift"
date: 2021-05-18
image: /images/certificate.jpg
---

![](/images/certificate.jpg)

-----
A couple of years ago at a meetup I talked about using cert-manager on OpenShift. At the time I used the Kubernetes Ingress object with cert-manager as it is directly supported by cert-manager. Unfortunately, OpenShift Routes are not supported by cert-manager even though someone made a PR to support them.

Fortunately for the OpenShift community this PR led to the creation of cert-utils-operator which patches OpenShift Routes with the Certificate generated from cert-manager.

ALL OF THIS because OpenShift Routes do not support TLS via k8s secrets.

I will also be using Venafi Cloud as the CA for this integration. Venafi has a pretty awesome API and it offers a wide variety of integrations which also includes a Terraform provider.

Now, with that out of the way, I'm going to write about how to configure cert-manager with Venafi for OpenShift routes with the help of the cert-utils-operator.

## Install cert-manager

The installation of cert-manager is pretty straightforward. Just do this:

```bash
oc create namespace cert-manager
oc apply -f https://github.com/jetstack/cert-manager/releases/download/v1.3.1/cert-manager.yaml
```

## Install cert-utils-operator

I installed it via Helm:

```bash
oc new-project cert-utils-operator
helm repo add cert-utils-operator https://redhat-cop.github.io/cert-utils-operator
helm repo update
helm install cert-utils-operator cert-utils-operator/cert-utils-operator
```

## Setting up Venafi Cloud for cert-manager

There's a series of easy steps that need to be done as prerequisites for the cert-manager integration. Steps for Venafi TPP are quite similar.

### Create an Issuing Template

Click on **Settings -> Issuing Templates -> Create a new Issuing Template**.

I went with the default Venafi built-in CA.

### Create an Application

Create a new Application by navigating to **Organizations -> Applications**. Choose the previously created Issuing Template when creating it.

### Get API keys

Next, grab API keys to create them with a Kubernetes secret. Go to the newly created Application and click on **API tools -> Kubernetes**. You will see the `kubectl` command auto-generated with the API token.

## Configure cert-manager

### Create the Secret

Copy the command to create the Kubernetes secret. Create it in the namespace where certs are needed. I create them in a namespace called `weather`:

```bash
kubectl create secret generic venafi-cloud-secret \
 --namespace='weather' \
 --from-literal=apikey='your_api_key'
```

### Create the Issuer

Create the Issuer for Venafi Cloud in the namespace where the certs are needed. You can also create a ClusterIssuer to make things more automated.

```yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: venafi-cloud-issuer
spec:
  venafi:
    zone: "certmanager\\cert-manager-it"
    cloud:
      apiTokenSecretRef:
        name: venafi-cloud-secret
        key: apikey
```

Apply the Issuer configuration:

```bash
kubectl apply -f issuer.yaml -n weather
```

### Create the Certificate

Create the certificate to reference the issuer:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: weather-certificate
spec:
  secretName: weather-certificate
  commonName: weather.apps.ocp.stackhack.ca
  dnsNames:
  - weather.apps.ocp.stackhack.ca
  issuerRef:
    name: venafi-cloud-issuer
```

Apply the Certificate configuration:

```bash
kubectl apply -f certificate.yaml -n weather
```

Once the certificate is created, it should automatically generate the secret containing the certificate:

```bash
kubectl get secrets -n weather | grep weather
```

Example output:

```plaintext
weather-tls          kubernetes.io/tls          3          5s
```

If you've made it thus far, everything is good to go to inject this secret into the Route.

## Create the Route

Finally, let's create the Route to automatically inject the TLS cert from the secret. Note the annotations here. This is where `cert-utils-operator` comes into play to inject the secret into the Route. This will not work without these annotations on OpenShift Routes.

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  annotations:
    cert-utils-operator.redhat-cop.io/certs-from-secret: weather-tls
    cert-utils-operator.redhat-cop.io/inject-CA: "false"
  labels:
    app: weather-app
    app.kubernetes.io/component: weather-app
    app.kubernetes.io/instance: weather-app
  name: weather
spec:
  host: weather.apps.ocp.stackhack.ca
  port:
    targetPort: 8080-tcp
  tls:
    insecureEdgeTerminationPolicy: None
    termination: edge
  to:
    kind: Service
    name: weather-app
    weight: 100
  wildcardPolicy: None
```

Apply the Route configuration:

```bash
kubectl apply -f route.yaml -n weather
```

## Verify the Certificate

Navigate to the URL and check out the cert.

Expand the cert to make sure it's the right CA.

Let me know if there are any questions!

The code used in the blog is available in this repo.