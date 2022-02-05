
# Add secret:

I deleted my old cm (you can ignore this step)
And then recreate new configmap without "PARSE_SERVER_MASTER_KEY" I'll set it as a secret:
```bash
kubectl create configmap parse \
--from-literal=PARSE_SERVER_APPLICATION_ID=MyParseApp \
--from-literal=PARSE_SERVER_DATABASE_URI=postgres://postgres:postgres@postgres/postgres
```
Let's create a secret and upgrad the helm chart:
```bash
k create secret generic secret-parse --from-literal=PARSE_SERVER_MASTER_KEY=adminadmin
k get secret secret-parse -o yaml
```
In the "server-deployment.yaml" you have to change "configMapKeyRef" to "secretKeyRef" and change the name of it (if you had, changed the name):

```bash

             - name: PARSE_SERVER_MASTER_KEY
               valueFrom:
                 configMapKeyRef:
                   name: parse
                   key: PARSE_SERVER_MASTER_KEY
             - name: PARSE_SERVER_DATABASE_URI
               valueFrom:
```
to:
```bash
             - name: PARSE_SERVER_MASTER_KEY
               valueFrom:
                 secretKeyRef:
                   name: secret-parse
                   key: PARSE_SERVER_MASTER_KEY
             - name: PARSE_SERVER_DATABASE_URI
               valueFrom:
```

After save run:
```bash
helm upgrade  parse ./charts/parse/
```

Test the Parse, If you run my project, ingress.yaml it will exist in the templates directory:
```bash
curl http://localhost:1337/parse/health
#in ingress mode:
curl http://rfinland.net/parse/health
```
Now kubernetes call "PARSE_SERVER_APPLICATION_ID" and "PARSE_SERVER_DATABASE_URI" from configmap and call "PARSE_SERVER_MASTER_KEY" from secrets.

# secret file
I'll do the same steps for secret as I did for configmap:
```bash
touch charts/parse/templates/secret.yaml
nano charts/parse/templates/secret.yaml
```
First you have to create base64 of your value:

echo -n 'adminadmin' | base64
the result "YWRtaW5hZG1pbg=="
and set it in your value as below:
 

and paste:
```bash
apiVersion: v1
data:
  PARSE_SERVER_MASTER_KEY: YWRtaW5hZG1pbg==
kind: Secret
metadata:
  name: secret-parse
  namespace: default
type: Opaque
```
Also you can do it with:
```bash
kubectl edit secret secret-parse
```
and copy and paste in your secret yaml file and remove unnecessary lines .
Delete the secret secret-parse and upgrade the chart:

```bash
kubectl delete secret secret-parse
```

then  run:
```bash
kubectl rollout restart deployment server
```

to see whats happen when secret state change (your pod stuck into CreateContainerConfigError) Now lets fix this by applying configmap file to chart:

```bash
helm upgrade parse charts/parse/ 
kubectl get secret 
kubectl get all -o wide 
```
Test the Parse, If you run my project, ingress.yaml it will exist in the templates directory:
```bash
curl http://localhost:1337/parse/health
#in ingress mode:
curl http://rfinland.net/parse/health

```

# call from values to secret then to env
#### secret
Our values.yaml file has been changed to:
```bash
server:
 appId: 'MyParseApp' 
 masterkey: 'adminadmin'
 database: 'postgres://postgres:postgres@postgres/postgres'
```

our secret.yaml file (I set default value and call from values.yaml):
```bash
apiVersion: v1
kind: Secret
metadata:
  name: secret-parse
type: Opaque
data:
  master-key: 
    {{ if .Values.masterkey }}
      {{ .Values.masterkey | quote }}
     {{ else }}
      {{ randAlphaNum 10 | quote }}
    {{ end }}
 
  
```  

and in server-deployment.yaml:
```bash
        - env:
             .
			 .
             - name: PARSE_SERVER_MASTER_KEY
               valueFrom:
                 secretKeyRef:
                   name: secret-parse
                   key: master-key
```
Install the chart:
```bash
helm install  parse . -f values.yaml
#OR default
helm install  parse .
```
When you use default you can see diffrent inner the secret:
```bash
k edit secrets secret-parse
```
If you run:
```bash
echo -n 'adminadmin' | base64 
```
the result "YWRtaW5hZG1pbg=="
and if you didn't call the values.yaml result of "k edit secrets secret-parse" will be diffrent.
