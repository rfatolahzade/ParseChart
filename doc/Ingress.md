
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
helm upgrade parse . -f values.yaml
kubectl get ingress
```
```bash
#Test On your CLUSTER-IP of(service/server) first :
curl 10.43.27.73:1337/parse/health
#On our Ingress:
curl http://rfinland.net/parse/health
```
