![Parse Evolution](images/logo.jpg)

# Parse Evolution

Steps to change any docker-compose file to helm chart:
# Table and Content
  - [Play with docker-compose](../master/doc/docker-compose.md)
  - [Use kompose](../master/doc/kompose.md)
  - [Helming](../master/doc/helming.md)
  - [Ingress](../master/doc/Ingress.md)
  - [GitHub Pages](../master/doc/GitHubPages.md)
  - [Configmaps](../master/doc/Configmaps.md)
  - [Secrets](../master/doc/Secrets.md)
  - [PV,PVC](../master/doc/pv-pvc.md)
  - [Helm Install options](../master/doc/Helm-Set.md)
  - [Postgres as a dependency](../master/doc/Postgress-Dep.md)
  - [TLS](../master/doc/TLS.md)


# Quick Start
Take a look to values file:
```bash 
git clone https://github.com/rfinland/ParseChart.git
cd ParseChart/charts/parse
nano values.sample.yaml
```
As default:
```bash
appId: 'MyParseApp'          #Parse appID 
masterkey: 'adminadmin'      #Parse masterkey 
url: '{YOUR Parse url}'      #YOUR Parse url 


postgresql:          
  username: postgres         #postgresql username
  password: postgres1        #postgresql password
  database: postgres         #postgresql database
  data_dir: '/ParesDatabase' #postgresql data directory mounted to default data dir


dashboard: 
  enabled: true               #True if you need dashboard 
  username: user              #dashboard username if you enabled dashboard 
  password: user              #dashboard password if you enabled dashboard 
  url: {YOUR Dashboard url}   #YOUR Dashboard url


certmanager:                  #If you want to use cert-manager as tls connected to letsencrypt
  enabled: false              #If true certs and issuer will create as your urls for parse and dashboard
  
```
# Helming!

```bash
helm install parse . -f values.sample.yaml 
```
If you have your own certs(server.crt,server.key) based on your urls:
```bash
kubectl create secret generic {YOUR Parse url} --from-file=tls.crt=./server.crt --from-file=tls.key=./server.key 
kubectl create secret generic {YOUR Dashboard url} --from-file=tls.crt=./server.crt --from-file=tls.key=./server.key 
```
You have to set name of secret inside values file.
If you want to use cert-manager and letsencrypt just enable certmanager inside values yaml file:
```bash
certmanager:          
  enabled: true
``` 

Here you go.
