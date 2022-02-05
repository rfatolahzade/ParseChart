

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
