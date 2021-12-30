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
