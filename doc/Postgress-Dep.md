
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