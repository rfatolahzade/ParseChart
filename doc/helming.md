# Helming!
```bash
echo $PWD
#/root/ParseChart
mkdir charts
cd charts
helm create Parse
```
First, delete everything under templates directory:
```bash
rm -r Parse/templates/*
rmdir Parse/charts
```
and then copy all yaml file from convert (placed postgres) and place those files under the templates directory:
```bash
cp ../postgres/* $PWD/Parse/templates/
```
Lets install our app,First delete previus deployment:
```bash
cd ../../postgres
kubectl delete -f .
cd ../charts/Parse
helm install parse .
```
Notice: in "helm install parse ." You won't be able to set an uppercase name.

##### Values.yaml
Lets create/empty values.sample.yaml to set our envs:

```bash
 cat <<EOF > values.sample.yaml
server:
 - name: PARSE_SERVER_APPLICATION_ID
   value: MyParseApp
 - name: PARSE_SERVER_MASTER_KEY
   value: adminadmin 
 - name: PARSE_SERVER_DATABASE_URI
   value: postgres://postgres:postgres@postgres/postgres
EOF
```

Now we have to call values inner server-deployment.yaml:
```bash
nano templates/server-deployment.yaml
```
then modify it as below:
```bash
containers:
        - env:
            {{- range .Values.server }}
            - name: {{ .name }}
              value: {{ .value }}
            {{- end }} 
          image: parseplatform/parse-server
          name: server
          ports:
            - containerPort: 1337
          resources: {}
      restartPolicy: Always
```
save the file and run helm install contains your values file:	  
```bash
helm upgrade parse . -f values.sample.yaml
kubectl get all -o wide 
```