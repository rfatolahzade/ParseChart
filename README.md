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