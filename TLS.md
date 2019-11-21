To provide TLS to ESP projects and web clients, we can use [cert-manager](https://docs.cert-manager.io). 

The following is an example how to deploy [Kuard](https://github.com/kubernetes-up-and-running/kuard), a demo app for Kubernetes Up and Running book. 

Note that the original documentation can be confusing, so I summerize the steps here. The goal is configure ingress to provide TLS support for Kuard -- when an HTTPS 
request is submitted to Kuard, the ingress will automatically request the certificate from a configured issuer and Kuard knows nothing about this.

**Install cert-manager**
```
kubectl create namespace cert-manager
kubectl label namespace cert-manager certmanager.k8s.io/disable-validation=true
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v0.10.1/cert-manager.yaml
```
You should see 3 pods running:
```
kubectl get pods -n cert-manager
NAME                                       READY   STATUS    RESTARTS   AGE
cert-manager-84fc69dbdf-p4z2t              1/1     Running   0          4h3m
cert-manager-cainjector-869bb969f6-mxsrn   1/1     Running   0          4h3m
cert-manager-webhook-7b4fb887bc-vbpgc      1/1     Running   0          4h3m
```

Cert-manager uses 2 CRDs to configure and control its operation: [Issuers (or ClusterIssuer)](https://docs.cert-manager.io/en/latest/reference/issuers.html) and [Certificates](https://docs.cert-manager.io/en/latest/reference/certificates.html). 

**Deploy an issuer**

This demo uses [ACME](https://en.wikipedia.org/wiki/Automated_Certificate_Management_Environment) protocol and [letsencrypt](https://letsencrypt.org/how-it-works/) as the issuer. Other issuers can also be configured. 
Note that letsencrypt has a very strict rate limit for producation certificates, so for test purpose we use a [staging issuer](https://gitlab.sas.com/shhuan/esp-k8s-operator/blob/master/oauth2/cert-manager/staging-issuer.yaml) and once everything works we will switch to 
[production issuer](https://gitlab.sas.com/shhuan/esp-k8s-operator/blob/master/oauth2/cert-manager/production-issuer.yaml).
```
kubectl apply -f staging-issuer.yaml -n shhuan
```
If cert-manager acts correctly:
```
kubectl get issuer -n shhuan
NAME                  AGE
letsencrypt-staging   106m
```

**Deploy Kuard**

Assume we added a record in DNS (e.g. esp-foo.sas.com) that resolves to the IP of the Nginx ingress controller's external interface. Now we are ready to deploy Kuard. 
```
kubectl apply -f kuard.yaml -n shhuan
```
The interesting part of [kuard.yaml](https://gitlab.sas.com/shhuan/esp-k8s-operator/blob/master/oauth2/cert-manager/kuard.yaml) is in the Ingress.

First, the annontations:
```
  annotations:
    kubernetes.io/ingress.class: "nginx"
    certmanager.k8s.io/issuer: "letsencrypt-staging"
```
This tells Nginx to use issuer "letsencrypt-staging", that just got created.

Then, the spec:
```
spec:
  tls:
  - hosts:
    - esp-foo.sas.com
    secretName: quickstart-example-tls
  rules:
  - host: esp-foo.sas.com
    http:
      paths:
      - path: /
        backend:
          serviceName: kuard
          servicePort: 80
```
Cert-manager will create a certificate if everything works correctly:
```
kubectl get certificate -n shhuan
NAME                     READY   SECRET                   AGE
quickstart-example-tls   False   quickstart-example-tls   108m
```
Note this is a staging issuer, as a result the certificate is temporary. That's why you see "False" under "READY". Now the secret should exist:
```
kubectl get secret quickstart-example-tls -n shhuan -o yaml
apiVersion: v1
data:
  ca.crt: ""
  tls.crt: ...
  tls.key: ...
 kind: Secret
...
```

Now in your browser enter [https://esp-foo.sas.com](https://esp-foo.sas.com), the broswer should warn you about the certificate because it's temporary, but you can 
proceed and see Kuard is running.