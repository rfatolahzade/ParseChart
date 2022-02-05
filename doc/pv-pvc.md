# PV,PVC
Create pv.yaml:
```bash
touch /ParseChart/charts/parse/templates/pv.yaml
nano /ParseChart/charts/parse/templates/pv.yaml
```
Paste:
```bash
apiVersion: v1
kind: PersistentVolume
metadata:
  name: parse-pv-hostpath
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 100Mi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: 
    {{ if .Values.postgresdatadir }}
      {{ .Values.postgresdatadir | quote }}
    {{ else }}
      {{ "/postgresdata" }}
    {{ end }}
```
	
I created /postgresdata directory as my pv hostPath named parse-pv-hostpath

and then create pvc.yaml:
```bash
touch /ParseChart/charts/parse/templates/pvc.yaml
nano /ParseChart/charts/parse/templates/pvc.yaml
```
Paste:
```bash
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: parse-pvc-hostpath
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Mi
```

Inner postgres-deployment.yaml define the pvc as below:
```bash
    spec:
      containers:
        - env:
            - name: POSTGRES_DB
              value: postgres
            - name: POSTGRES_PASSWORD
              value: postgres
            - name: POSTGRES_USER
              value: postgres
          image: postgres
          name: postgres
          ports:
            - containerPort: 5432
          volumeMounts:
          - name: host-volume
            mountPath: /var/lib/postgresql/data
          resources: {}
      volumes:
      - name: host-volume
        persistentVolumeClaim:
         claimName: parse-pvc-hostpath
      restartPolicy: Always
status: {}
```
The mountPath: /var/lib/postgresql/data as postgres data directory.

Let's intall the helm:
```bash
cd /ParseChart/charts/Parse
helm install parse . -f values.sample.yaml
```

Test Health:
```bash
curl http://rfinland.net/parse/health
```
Insert 2records:
```bash
curl -X POST \
-H "X-Parse-Application-Id: MyParseApp" \
-H "Content-Type: application/json" \
-d '{"score":1000,"playerName":"Rain Man","cheatMode":false}' \
http://rfinland.net/parse/classes/GameScore

curl -X POST \
-H "X-Parse-Application-Id: MyParseApp" \
-H "Content-Type: application/json" \
-d '{"score":1001,"playerName":"FSM","cheatMode":false}' \
http://rfinland.net/parse/classes/GameScore
```

Insert 2records in other table:
```bash
curl -X POST \
-H "X-Parse-Application-Id: MyParseApp" \
-H "Content-Type: application/json" \
-d '{"ident":1,"name":"Rain Man","location":"FI"}' \
http://rfinland.net/parse/classes/UserList

curl -X POST \
-H "X-Parse-Application-Id: MyParseApp" \
-H "Content-Type: application/json" \
-d '{"ident":2,"Name":"FSM","Location":"NL"}' \
http://rfinland.net/parse/classes/UserList

```
GET records:
```bash

curl -X GET \
  -H "X-Parse-Application-Id: MyParseApp" \
  http://rfinland.net/parse/classes/GameScore
  
curl -X GET \
  -H "X-Parse-Application-Id: MyParseApp" \
  http://rfinland.net/parse/classes/UserList
 ``` 
  
Now lets uninstall helm chart:
```bash
helm uninstall parse
```
and then reInstall the chart:
```bash
helm install parse . -f values.sample.yaml
```
Get old records as well:
```bash
curl -X GET \
-H "X-Parse-Application-Id: MyParseApp" \
http://rfinland.net/parse/classes/GameScore
  
curl -X GET \
-H "X-Parse-Application-Id: MyParseApp" \
http://rfinland.net/parse/classes/UserList
  
  ```