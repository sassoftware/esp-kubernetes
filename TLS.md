# TLS

To provide TLS to ESP projects and web clients, we can use [cert-manager](https://cert-manager.io/docs/).

The following is an example how to deploy [Kuard](https://github.com/kubernetes-up-and-running/kuard), a demo app for Kubernetes Up and Running book.

The steps are summarized here to supplement the Kuard documentation. The goal is to configure ingress to provide TLS support for Kuard -- when an HTTPS
request is submitted to Kuard, the ingress will automatically request the certificate from a configured issuer and Kuard knows nothing about this.

## Install cert-manager

```shell
kubectl create namespace cert-manager
kubectl label namespace cert-manager certmanager.k8s.io/disable-validation=true
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.10.2/cert-manager.yaml
```

You should see 3 pods running:

```shell
$ kubectl get pods -n cert-manager
NAME                                       READY   STATUS    RESTARTS   AGE
cert-manager-84fc69dbdf-p4z2t              1/1     Running   0          4h3m
cert-manager-cainjector-869bb969f6-mxsrn   1/1     Running   0          4h3m
cert-manager-webhook-7b4fb887bc-vbpgc      1/1     Running   0          4h3m
```

Cert-manager uses 2 CRDs to configure and control its operation: [Issuers (or ClusterIssuer)](https://cert-manager.io/v1.10-docs/reference/api-docs/#cert-manager.io/v1.ClusterIssuer) and [Certificates](https://cert-manager.io/v1.10-docs/reference/api-docs/#cert-manager.io/v1.Certificate).

## Deploy an issuer

This demo uses [ACME](https://en.wikipedia.org/wiki/Automatic_Certificate_Management_Environment) protocol and [Let's Encrypt](https://letsencrypt.org/how-it-works/) as the issuer. Other issuers can also be configured.
Note that Let's Encrypt has a very strict rate limit for production certificates.
For testing, use the Let's Encrypt staging issuer; for example:

```yaml
apiVersion: certmanager.k8s.io/v1alpha1
kind: Issuer
metadata:
    name: letsencrypt-staging
spec:
    acme:
        # The ACME server URL
        server: https://acme-staging-v02.api.letsencrypt.org/directory
        # Email address used for ACME registration
        email: jane.doe@localhost.localdomain
        # Name of a secret used to store the ACME account private key
        privateKeySecretRef:
            name: letsencrypt-staging
        # Enable the HTTP-01 challenge provider
        solvers:
            - http01:
                ingress:
                    class:  nginx
```

After everything works, switch to the Let's Encrypt production issuer; for example:

```yaml
apiVersion: certmanager.k8s.io/v1alpha1
kind: Issuer
metadata:
    name: letsencrypt-prod
spec:
    acme:
        # The ACME server URL
        server: https://acme-v02.api.letsencrypt.org/directory
        # Email address used for ACME registration
        email: jane.doe@localhost.localdomain
        # Name of a secret used to store the ACME account private key
        privateKeySecretRef:
            name: letsencrypt-prod
        # Enable the HTTP-01 challenge provider
        solvers:
            - http01:
                ingress:
                    class:  nginx
```

If cert-manager acts correctly:

```shell
$ kubectl get issuer -n "${K8S_NAMESPACE}"
NAME                  AGE
letsencrypt-staging   106m
```

## Deploy Kuard

Assume we added a record in DNS (e.g. esp-foo.example.com) that resolves to the IP of the Nginx ingress controller's external interface. Now we are ready to deploy Kuard.

```shell
kubectl apply -f - -n "${K8S_NAMESPACE}" <<\EOF
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kuard
spec:
  selector:
    matchLabels:
      app: kuard
  replicas: 1
  template:
    metadata:
      labels:
        app: kuard
    spec:
      containers:
      - image: gcr.io/kuar-demo/kuard-amd64:blue
        imagePullPolicy: Always
        name: kuard
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: kuard
spec:
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
  selector:
    app: kuard
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: kuard
  annotations:
    kubernetes.io/ingress.class: "nginx"
    certmanager.k8s.io/issuer: "letsencrypt-staging"
spec:
  tls:
  - hosts:
    - esp-foo.example.com
    secretName: quickstart-example-tls
  rules:
  - host: esp-foo.example.com
    http:
      paths:
      - path: /
        backend:
          serviceName: kuard
          servicePort: 80
EOF
```

The interesting part is in the Ingress.

First, the annotations:

```yaml
  annotations:
    kubernetes.io/ingress.class: "nginx"
    certmanager.k8s.io/issuer: "letsencrypt-staging"
```

This tells Nginx to use issuer `letsencrypt-staging`, that just got created.

Then, the spec:

```yaml
spec:
  tls:
  - hosts:
    - esp-foo.example.com
    secretName: quickstart-example-tls
  rules:
  - host: esp-foo.example.com
    http:
      paths:
      - path: /
        backend:
          serviceName: kuard
          servicePort: 80
```

Cert-manager will create a certificate if everything works correctly:

```shell
$ kubectl get certificate -n "${K8S_NAMESPACE}"
NAME                     READY   SECRET                   AGE
quickstart-example-tls   False   quickstart-example-tls   108m
```

Note this is a staging issuer, as a result the certificate is temporary. That's why you see "False" under "READY". Now the secret should exist:

```shell
$ kubectl get secret quickstart-example-tls -n "${K8S_NAMESPACE}" -o yaml
apiVersion: v1
data:
  ca.crt: ""
  tls.crt: ...
  tls.key: ...
 kind: Secret
...
```

In your browser, enter `https://esp-foo.example.com`, the browser should warn you about the certificate because it is temporary. Nonetheless, you can proceed and see Kuard is running.
