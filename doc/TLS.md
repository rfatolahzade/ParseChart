
#### tls
Self-signed certificates:
```bash
openssl req -nodes -new -x509 -keyout server.key -out server.crt -days 365 \
-subj "/C=FI/ST=Finland/L=Helsinki/O=MyApa/CN=rtl.net" \
-addext "subjectAltName = DNS:parse.rtl.net,DNS:dashboard.rtl.net"
```
Create secrets for dashboard and parse:
```bash
kubectl create secret generic parse-tls --from-file=tls.crt=./server.crt --from-file=tls.key=./server.key 
```
Run traefik dashboard:
```bash
kubectl port-forward -n kube-system "$(kubectl get pods -n kube-system| grep '^traefik-' | awk '{print $1}')" 9000:9000
```

# Kubernetes Traefik Ingress LetsEncrypt
First of all you have to install cert-manager:
```bash
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true
```
Add an issuer:
```bash
nano  letsencrypt-issuer.yml
```
change as shown below:
```bash
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
  namespace: default
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: r.finland88@gmail.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: traefik
```
You have to change email address.
Apply Issuer:
```bash
kubectl apply -f letsencrypt-issuer.yml
```
We have deployed letsEncrypt issuer which issues certificates.
```bash
kubectl get ClusterIssuer
```
# Creating Traefik Ingress Let’s Encrypt TLS Certificate
Now lets create Traefik Ingress LetsEncrypt TLS certificate for your microservice (parse and dashboard):

For parse:
```bash
nano letsencrypt-cert-parse.yaml
```
change as shown below:
```bash
{{- if .Values.certmanager.enabled -}}

apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: {{ .Values.url | quote }}
  namespace: default
spec:
  secretName: {{ .Values.url | quote }}
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  commonName: {{ .Values.url | quote }}
  dnsNames:
  - {{ .Values.url | quote }}
 
{{- end -}}

  
```
For Dashboard:
```bash
nano letsencrypt-cert-dashboard.yaml
```
change as shown below:
```bash
{{- if .Values.certmanager.enabled -}}

apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: {{ .Values.dashboard.url | quote }}
  namespace: default
spec:
  secretName: {{ .Values.dashboard.url | quote }}
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  commonName: {{ .Values.dashboard.url | quote }}
  dnsNames:
  - {{ .Values.dashboard.url | quote }}
 
 
{{- end -}}

  
```

And apply:

```bash
kubectl apply -f letsencrypt-cert-parse.yaml
kubectl apply -f letsencrypt-cert-dashboard.yaml
```
List of Certificate:
```bash
kubectl get Certificate
```
List of Secrets:

```bash
kubectl get secrets
```
IF you set:
```bash
certmanager:
  enabled: true
```
letsencrypt-cert-dashboard,letsencrypt-cert-parse and letsencrypt-issuer will prepare k8s to use cert-manager as tls
# Accessing Traefik Ingress Resources using Let’s Encrypt
Just visit your link on browser.
