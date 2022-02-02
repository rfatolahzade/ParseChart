![Parse Evolution](images/logo.jpg)

# Parse Evolution
Check docker-compose config:
```bash
git clone https://github.com/rfinland/ParseChart.git
cd ParseChart
docker-compose config 
```
Notice: Application-Id values are selective with user MASTER_KEY; you can set values for these envs as you want.
In our case APP_ID=MyParseApp and MASTER_KEY=adminadmin in parse 
Up the app:
```bash
docker-compose up -d
#OR If already exist:
docker-compose up --force-recreate -d
```
#notice: If you want to create kube via docker-compose yaml file you have to set your envs as static in your yaml file
#Check Parse status
```bash
curl http://localhost:1337/parse/health
#Result: {"status":"ok"}
```
# POST 
Post a record:
You have to set your X-Parse-Application-Id as you set in environment section on your docker-compose file (in our case:APP_ID=MyParseApp)
```bash
curl -X POST \
-H "X-Parse-Application-Id: MyParseApp" \
-H "Content-Type: application/json" \
-d '{"score":1000,"playerName":"Rain Man","cheatMode":false}' \
http://localhost:1337/parse/classes/GameScore
```
You should get a response similar to this:
```bash
{
"objectId":"ngaYmcDv1m",
"createdAt":"2021-12-06T06:56:09.068Z"
}
```
# GET
Get the record:
```bash
curl -X GET \
  -H "X-Parse-Application-Id: MyParseApp" \
  http://localhost:1337/parse/classes/GameScore/ngaYmcDv1m
```
Get all records that you defined:
```bash
curl -X GET \
  -H "X-Parse-Application-Id: MyParseApp" \
  http://localhost:1337/parse/classes/GameScore
```
Also you can see the result in your browser (you just need set X-Parse-Application-Id: MyParseApp )
# Recommended add-ons
##### JSON Formatter: 
Chrome extension for printing JSON and JSONP nicely when you visit it 'directly' in a browser tab.
[JSON Formatter](https://github.com/callumlocke/json-formatter)

##### Modify Header:
Add, modify or remove a header for any request on desired domains.
[Modify Header](https://mybrowseraddon.com/modify-header-value.html)
##### RestMan:
RESTMan is a browser extension to work on http requests.
[RestMan](https://chrome.google.com/webstore/detail/restman/ihgpcfpkpmdcghlnaofdmjkoemnlijdi)

#DELETE Records reamin
#How to delete records that we created.
Lets do these steps with Kubernetes:
```bash
docker-compose down
```
kompose Installation:
# Linux
```bash
curl -L https://github.com/kubernetes/kompose/releases/download/v1.25.0/kompose-linux-amd64 -o kompose
chmod +x kompose
sudo mv ./kompose /usr/local/bin/kompose
#OR
brew install kompose
```
# Convert time
just run :
```bash
mkdir postgres
cd postgres
cp ../docker-compose.yaml .
kompose convert
rm docker-compose.yaml
```
And Then Apply yaml files:
```bash
kubectl apply -f .
```

OR If you ran "kompose convert -o parse.yaml" then just run:
```bash
kubectl apply -f parse.yaml
```

Two way to find out why your pod not ready (if it happens):
```bash
#First one:
kubectl logs pod/server-74cd8bb94d-bhtwt   #your pod name
#Second one:
kdes pod/server-74cd8bb94d-bhtwt   #describe of your not ready pod
```
To access to your Parse:
```bash
#The CLUSTER-IP  of your service/server
curl <CLUSTER-IP>:1337/parse/health
curl 10.43.33.149:1337/parse/health
```
It returns:
```bash
{"status":"ok"}
```

Now lets expose this port via nodePort service:
```bash
mkdir nodePort
cd nodePort
cp ../postgres/*  .

```
You have to add "nodePort: 30001" & type: NodePort under spec section of server-service:
```bash
spec:
  ports:
    - name: "1337"
      protocol: "TCP"
      port: 1337
      targetPort: 1337
      nodePort: 30001
  selector:
    io.kompose.service: server
  type: NodePort
status:
  loadBalancer: {}

```
and then apply changes:
```bash
kubectl apply -f .
```
Now lets POST a sample record:
```bash
curl -X POST \
-H "X-Parse-Application-Id: MyParseApp" \
-H "Content-Type: application/json" \
-d '{"score":1000,"playerName":"Rain Man","cheatMode":false}' \
http://localhost:30001/parse/classes/GameScore
```
You should get a response similar to this:
```bash
{
"objectId":"kJx7buQPDW",
"createdAt":"2021-12-08T09:17:08.682Z"
}
Get the record (Notice the objectId at least of the url):
```bash
curl -X GET \
  -H "X-Parse-Application-Id: MyParseApp" \
  http://localhost:30001/parse/classes/GameScore/kJx7buQPDW  
```
Get all records that you defined:
```bash
curl -X GET \
  -H "X-Parse-Application-Id: MyParseApp" \
  http://localhost:30001/parse/classes/GameScore
```

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

# Ingress 
Let's add ingress to project:
```bash
cd charts/Parse
touch templates/ingress.yaml
nano templates/ingress.yaml
```
Paste :
```bash
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
spec:
  rules:
  - host: rfinland.net
    http:
      paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: server
              port:
                number: 1337
```
I set "- host: rfinland.net" this is what I wrote in /etc/hosts for my  host IP:
```bash
ip -br -c a    #eth0 ip address in my case
```
then copy the host IP in my case "192.168.1.209" and then set it in /etc/hosts
192.168.1.209 rfinland.net
```
Apply changes:
```bash
helm upgrade parse . -f values.sample.yaml
kubectl get ingress
```
```bash
#Test On your CLUSTER-IP of(service/server) first :
curl 10.43.27.73:1337/parse/health
#On our Ingress:
curl http://rfinland.net/parse/health
```

# GitHub Pages
Create a new branch for github pages:
```bash
#Create a new branch:
git checkout --orphan gh-pages
#We have to make it empty:
git rm -rf .
#Commit & Push the new branch:
git commit --allow-empty -m "root commit"
git push origin gh-pages
#Chech the current branch:
git branch
#Switch to the branch:
git checkout gh-pages
#Switch to master branch:
git checkout master
#Delete branch: When your current branch is main
git branch -d gh-pages
#Delete branch remotely
git push origin --delete gh-pages
```

Lets create an index.html file on our new branch:
```bash
git checkout gh-pages
touch index.html
echo "Hello!" > index.html
git add .
git commit -m "A index html to GitHub page"
git push --set-upstream origin gh-pages
```

And then visit:
```bash
https://rfinland.github.io/ParseChart/index.html
#https://YOURGITHUBUSER.github.io/YOURPROJECT/index.html
#OR just visit:
https://rfinland.github.io/ParseChart/
```
You should able to see what's happen in github actions on your repo
Also you can set theme for your page and add a README.md
For open source projects, GitHub Pages is a great choice to host Helm repositories. 
We’re using the gh-pages branch to store and serve the packaged charts in this part of article. 
After each release we undergo a manual process of packaging and pushing the new chart version to the gh-pages branch.

# Create Helm package
## With cr (chart-releaser)
Install the cr:
```bash
brew tap helm/tap
brew install chart-releaser
```
And then clone the repo:
```bash
git clone https://github.com/rfinland/ParseChart.git
```
Run this command to create package of your helm chart:
```bash
cd ParseChart
helm package charts/Parse --destination .deploy
```
Upload Helm chart packages to GitHub Releases:
```bash
cr upload -o rfinland -r ParseChart -p .deploy -t <YOURTOKEN>
git checkout gh-pages
```
Update a Helm chart repository index.yaml file based on a the given GitHub repository's releases:
```bash
cr index -i ./index.yaml -p .deploy --owner rfinland --charts-repo https://rfinland.github.io/ParseChart --git-repo ParseChart
```
My first package link:
```bash
https://github.com/rfinland/ParseChart/releases/download/Parse-0.1.0/Parse-0.1.0.tgz
```
Push created index.yaml to gh-pages branch:
```bash
git checkout gh-pages
git add index.yaml
git commit -m "Added index yaml"
git push
```
The index.yaml file avaiable on:
```bash
https://rfinland.github.io/ParseChart/index.yaml
```
At last:
```bash
helm repo add Parse https://rfinland.github.io/ParseChart
helm repo update
helm repo list
helm search repo Parse
```
#### Install the package:

##### via The absolute URL: 
```bash
helm install parse https://github.com/rfinland/ParseChart/releases/download/Parse-0.1.0/Parse-0.1.0.tgz
```
##### via The package:
 ```bash
helm install parse parse/parse
```

## With GitHub action
Now I'm going to create the package via GitHub action:
 ```bash
mkdir .github
mkdir .github/workflows
touch .github/workflows/release.yaml 
nano .github/workflows/release.yaml 
 ```
Fill it as below:
 ```bash
name: Release Charts

on:
  push:
    branches:
      - main

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v2
        with:
          fetch-depth: 0

      - name: Configure Git
        run: |
          git config user.name "$GITHUB_ACTOR"
          git config user.email "$GITHUB_ACTOR@users.noreply.github.com"
      - name: Run chart-releaser
        uses: helm/chart-releaser-action@v1.1.0
        env:
          CR_TOKEN: "${{ secrets.GITHUB_TOKEN }}"
		  
 ```
Make a fake release to test the action:
in the ParseChart\charts\Parse\Chart.yaml
I changed:
 ```bash
version: 0.1.0 to version: 0.1.1
 ```
save the change and push our changes
 ```bash
git checkout main
git add .
git commit -m "make an action"
git push
 ```
Visit the action tab  and watch what's going on, As you can see after whole procces of the action done, new release "Parse-0.1.1" published as well.
You can update your helm repo and see the new release.

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


# call from values to configmap and secret then to env
#### secret
Our values.sample.yaml file has been changed to:
```bash
server:
 appId: 'MyParseApp' 
 masterkey: 'adminadmin'
 database: 'postgres://postgres:postgres@postgres/postgres'
```

our secret.yaml file (I set default value and call from values.sample.yaml):
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
helm install  parse . -f values.sample.yaml
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
and if you didn't call the values.sample.yaml result of "k edit secrets secret-parse" will be diffrent.


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

# Helm Install options
#### secret
```bash
helm install parse charts/parse/ --set masterkey=master
echo -n 'master' | base64 
```
Result of "echo -n 'master' | base64 " is bWFzdGVy
If you edit the secret-parse:
```bash
kubectl edit secret secret-parse
```
The value will be "bWFzdGVy"

```bash
helm install parse charts/parse/
```
The value will be :
```bash
randAlphaNum 10 | b64enc
```
Call masterkey from values.sample.yaml:
```bash
helm install parse charts/parse/   -f charts/parse/values.sample.yaml
```
The value will be "YWRtaW5hZG1pbg==":
```bash
echo -n 'adminadmin' | base64 
```
If I edit my secret:
```bash
kubectl edit secret secret-parse
```
I can see my value:"YWRtaW5hZG1pbg=="
#### ingress
If you set host inner values.sample.yaml Parse ingress will be install too.
```bash
helm install parse charts/parse/   -f charts/parse/values.sample.yaml
```
Set it with set:
```bash
helm install parse charts/parse/ --set hostname=rtl.net
```
And then you have to write your host in /etc/hosts
```bash
curl http://rtl.net/parse/health
```

# Add postgres as a dependency
Change the chart.yaml as below:

```bash
apiVersion: v2
name: parse
description: Parse platform
type: application
version: 0.1.3
appVersion: "1.16.0"
dependencies:
- name: postgresql
  repository: https://charts.bitnami.com/bitnami
  version: 10.x.x

```
Delete deployment and service files for postgres in  templates directory.
Add POSTGRES_PASS as env inner server-deployment:
```bash
             - name: POSTGRES_PASS
               valueFrom:
                 secretKeyRef:
                   name: parse-postgresql
                   key: postgresql-password
             - name: PARSE_SERVER_DATABASE_URI
               value: postgres://postgres:$(POSTGRES_PASS)@parse-postgresql/postgres
```
And run this command to add your dependency to chart:

```bash
mkdir ParseChart/charts/parse/charts
cd ParseChart/charts/parse
helm dependency update
```
inside the ParseChart/charts/parse/charts you able to see .tgz file of you dependency.
inner the values yml file(values.sample.yaml) add these lines to set password for postgresql (your dependency):
```bash
postgresql:
  postgresqlPassword: 'mypass'
```
and run the app:
```bash
helm install parse . --set hostname=rtl.net -f values.sample.yaml
```
Your app will be up and test it
Test Health:
```bash
curl http://rtl.net/parse/health
```
POST a records:
```bash
curl -X POST \
-H "X-Parse-Application-Id: MyParseApp" \
-H "Content-Type: application/json" \
-d '{"score":1000,"playerName":"Rain Man","cheatMode":false}' \
http://rtl.net/parse/classes/GameScore
```

GET records:
```bash

curl -X GET \
  -H "X-Parse-Application-Id: MyParseApp" \
  http://rtl.net/parse/classes/GameScore

```
#### tls
Self-signed certificates:
```bash
openssl req -nodes -new -x509 -keyout server.key -out server.crt -days 365 \
-subj "/C=FI/ST=Finland/L=Helsinki/O=MyApa/CN=rtl.net" \
-addext "subjectAltName = DNS:parse.rtl.net,DNS:dashboard.rtl.net"
```
Create a secret:
```bash
kubectl create secret generic parse-tls --from-file=tls.crt=./server.crt --from-file=tls.key=./server.key 
```
Run traefik dashboard:
```bash
kubectl port-forward -n kube-system "$(kubectl get pods -n kube-system| grep '^traefik-' | awk '{print $1}')" 9000:9000
```

# Kubernetes Traefik Ingress LetsEncrypt
Add an issuer:
```bash
kubectl apply -f letsencrypt-issuer.yml
```
We have deployed letsEncrypt issuer which issues certificates.
You have to change email address.



