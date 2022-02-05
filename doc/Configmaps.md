
# Add configmap:
First,I do these steps commandly, Let's create a configmap:
```bash
k create configmap parse \
--from-literal=PARSE_SERVER_APPLICATION_ID=MyParseApp \
--from-literal=PARSE_SERVER_MASTER_KEY=adminadmin \
--from-literal=PARSE_SERVER_DATABASE_URI=postgres://postgres:postgres@postgres/postgres
```
Let's see our configmap:
```bash
kubectl get cm
kubectl describe cm parse
```
Then, we have to set env to use our new configmap, so let's do it:
```bash
nano charts/parse/templates/server-deployment.yaml 
```
Change:
```bash
    spec:
      containers:
        - env:
            {{- range .Values.server }}
            - name: {{ .name }}
              value: {{ .value }}
            {{- end }} 
          image: parseplatform/parse-server
```
to:
```bash
    spec:
      containers:
        - env:
             - name: PARSE_SERVER_APPLICATION_ID
               valueFrom:
                 configMapKeyRef:
                   name: parse
                   key: PARSE_SERVER_APPLICATION_ID
             - name: PARSE_SERVER_MASTER_KEY
               valueFrom:
                 configMapKeyRef:
                   name: parse
                   key: PARSE_SERVER_MASTER_KEY
             - name: PARSE_SERVER_DATABASE_URI
               valueFrom:
                 configMapKeyRef:
                   name: parse
                   key: PARSE_SERVER_DATABASE_URI 
          image: parseplatform/parse-server
```
Save the changes and run:
```bash
helm install  parse ./charts/parse/
#helm upgrade  parse ./charts/parse/
```

Test the Parse, If you run my project, ingress.yaml it will exist in the templates directory: 
```bash
curl http://localhost:1337/parse/health
#in ingress mode:
curl http://rfinland.net/parse/health
```
Chacke the status and post a record:
```bash
curl -X POST \
-H "X-Parse-Application-Id: MyParseApp" \
-H "Content-Type: application/json" \
-d '{"score":1000,"playerName":"Rain Man","cheatMode":false}' \
http://localhost:1337/parse/classes/GameScore

#in ingress mode:
curl -X POST \
-H "X-Parse-Application-Id: MyParseApp" \
-H "Content-Type: application/json" \
-d '{"score":1000,"playerName":"Rain Man","cheatMode":false}' \
http://rfinland.net/parse/classes/GameScore

```
Get the record:
```bash
curl -X GET \
  -H "X-Parse-Application-Id: MyParseApp" \
  http://localhost:1337/parse/classes/GameScore
#in ingress mode:
curl -X GET \
  -H "X-Parse-Application-Id: MyParseApp" \
  http://rfinland.net/parse/classes/GameScore
```
#### change value inner configmap env
Now you can delete values.sample.yaml, and let's edit cm:
```bash
k edit cm parse
```
then change the name via "kubectl edit cm parse" and then change the value or:

```bash
kubectl get cm parse -o yaml | \
  sed -e 's|PARSE_SERVER_APPLICATION_ID: MyParseApp|PARSE_SERVER_APPLICATION_ID: MyParseApps|' | \
  kubectl apply -f -
```
see your change:
```bash
kubectl get cm parse -o yaml
#history of change saved as well if you ran your change via command
```
Now we have to restart deployment/pod:
```bash
kubectl rollout restart deployment server 
#OR
k exec -it pod/server-57b5f759b9-qkrtm sh
reboot
```

After reboot/restart lets take a look inner NEW pod:
```bash
k exec -it pod/server-66df6c6cd8-xr6v4 sh
env | grep My
```

Your new value is here "PARSE_SERVER_APPLICATION_ID=MyParseApps".
And test the new value:
```bash
curl -X POST \
-H "X-Parse-Application-Id: MyParseApps" \
-H "Content-Type: application/json" \
-d '{"score":1001,"playerName":"Rain Man","cheatMode":true}' \
http://localhost:1337/parse/classes/GameScore

#in ingress mode:
curl -X POST \
-H "X-Parse-Application-Id: MyParseApps" \
-H "Content-Type: application/json" \
-d '{"score":1001,"playerName":"Rain Man","cheatMode":true}' \
http://rfinland.net/parse/classes/GameScore
```

Get record(s):
```bash
curl -X GET \
  -H "X-Parse-Application-Id: MyParseApps" \
  http://localhost:1337/parse/classes/GameScore
#in ingress mode:
curl -X GET \
  -H "X-Parse-Application-Id: MyParseApps" \
  http://rfinland.net/parse/classes/GameScore
```
As you can see old records also exist.



# configmap file
Create a config map as file in helm chart. In this step you can use your cm that is already created in your cluster or create a file as configmap in charts/parse/templates:
If you want to create the file your script will be:
```bash
touch charts/parse/templates/configmap.yaml
nano charts/parse/templates/configmap.yaml
```
and paste:
```bash
apiVersion: v1
data:
  PARSE_SERVER_APPLICATION_ID: MyParseApp
  PARSE_SERVER_DATABASE_URI: postgres://postgres:postgres@postgres/postgres
kind: ConfigMap
metadata:
  name: parse
  namespace: default
```
If you want to create this file from your cm that ran with command, It just need to run:
```bash
kubectl edit cm parse
```
and then copy script into a file in charts/parse/templates directory and remove unnecessary lines like creationTimestamp, resourceVersion, uid and your configmap file will be same as I did in the configmap.yaml

Delete the cm parse and upgrade the chart:

```bash
kubectl delete cm parse
```
then  run:
```bash
kubectl rollout restart deployment server
```
to see whats happen when configmap not ready (your pod fails) Now lets fix this by applying configmap file to chart:

```bash
helm upgrade parse charts/parse/ 
kubectl get cm 
kubectl get all -o wide 
```
Test the Parse, If you run my project, ingress.yaml it will exist in the templates directory:
```bash
curl http://localhost:1337/parse/health
#in ingress mode:
curl http://rfinland.net/parse/health
```

# call from values to configmap then to env
#### configmap
our configmap.yaml file (I set default value and call from values.sample.yaml):
 
 ```bash
apiVersion: v1
data:
  app-id: 
   {{ if .Values.appId }}
    {{ .Values.appId | quote }}
   {{ else }}
    {{ "MyParse" }}
   {{ end }} 
  database-url: "postgres://postgres:postgres@postgres/postgres"

kind: ConfigMap
metadata:
  name: parse
  namespace: default
```

In the serve.deployment.yaml:
```bash
        - env:
             - name: PARSE_SERVER_APPLICATION_ID
               valueFrom:
                 configMapKeyRef:
                   name: parse
                   key: app-id
             - name: PARSE_SERVER_MASTER_KEY
               valueFrom:
                 secretKeyRef:
                   name: secret-parse
                   key: master-key
             - name: PARSE_SERVER_DATABASE_URI
               valueFrom:
                 configMapKeyRef:
                   name: parse
                   key: database-url 
          image: parseplatform/parse-server
```
Install the chart:
```bash
helm install  parse . -f values.sample.yaml
#OR default
helm install  parse .
```



