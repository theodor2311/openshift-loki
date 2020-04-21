# Install Loki Stack on OpenShift
*Disclaimer: This is just for testing loki stack in Openshift, not a good practice in security view*
# Setting up Environment Variables for Installation
```bash
# Tested with OpenShift 4.3.9
HELM_VERSION='v3.2.0-rc.1' #Tested with v3.2.0-rc.1
LOKI_NAMESPACE='openshift-loki'
LOKI_STACK='loki-stack' #Tested with Chart: loki-stack-0.36.0, App Version v1.4.1
LOKI_GRAFANA='loki-grafana' #Tested with Chart: grafana-5.0.13, App Version: 6.7.1
GRAFANA_PASSWD='redhat'
```
# Install and configure Helm3
```bash
curl -s "https://get.helm.sh/helm-${HELM_VERSION}-linux-amd64.tar.gz" | tar xz
mv linux-amd64/helm /usr/bin/helm
rm -rf linux-amd64/
helm repo add loki https://grafana.github.io/loki/charts
helm repo add stable https://kubernetes-charts.storage.googleapis.com
# helm repo update
```
# Create Namespace for Loki Stack
```bash
oc create -f - <<EOF
apiVersion: v1
kind: Namespace
metadata:
  annotations:
    openshift.io/sa.scc.supplemental-groups: 10001/10000
    openshift.io/sa.scc.uid-range: 10001/10000
  name: $LOKI_NAMESPACE
EOF
```
# Adding privileged scc to service accounts
```bash
oc adm policy add-scc-to-user privileged -z $LOKI_STACK-promtail -n $LOKI_NAMESPACE
oc adm policy add-scc-to-user privileged -z $LOKI_STACK -n $LOKI_NAMESPACE
oc adm policy add-scc-to-user privileged -z $LOKI_STACK-fluent-bit-loki -n $LOKI_NAMESPACE
oc adm policy add-scc-to-user privileged -z $LOKI_GRAFANA -n $LOKI_NAMESPACE
```
# Install Loki Stack
Reference https://hub.helm.sh/charts/loki/loki-stack for more parameters
```bash
helm upgrade --install $LOKI_STACK loki/loki-stack \
--set fluent-bit.enabled=false,promtail.enabled=true,loki.persistence.enabled=true,loki.persistence.size=100Gi \
--namespace=$LOKI_NAMESPACE
# Adding privileged securityContext for promtail is required in OpenShift
oc patch ds $LOKI_STACK-promtail  -p '{"spec": {"template": {"spec": {"containers": [{"name": "promtail","securityContext": {"privileged": true}}]}}}}' -n $LOKI_NAMESPACE
```
# Install Grafana
Reference https://hub.helm.sh/charts/stable/grafana for more parameters
```bash
helm install $LOKI_GRAFANA stable/grafana \
--set persistence.enabled=true,persistence.type=pvc,persistence.size=10Gi \
--namespace=$LOKI_NAMESPACE
# Create OpenShift route for Grafana
oc -n $LOKI_NAMESPACE create route edge $LOKI_GRAFANA --service=$LOKI_GRAFANA --port=service --insecure-policy=Redirect
```
# Login to Grafana
After Loki Stack and Grafana deployments running, we can get the cookies for further processing with following scripts
```bash
GRAFANA_OLDPASSWD=$(oc get secret --namespace $LOKI_NAMESPACE $LOKI_GRAFANA -o jsonpath="{.data.admin-password}" | base64 --decode)
GRAFANA_ROUTE="https://$(oc get route $LOKI_GRAFANA -o jsonpath='{.spec.host}' -n $LOKI_NAMESPACE)"
curl -X POST -d '{"user": "admin", "password": "'$GRAFANA_OLDPASSWD'", "email": ""}' -H "Content-Type: application/json" -k $GRAFANA_ROUTE/login -c /tmp/grafana_cookies.txt
```
# Changing Grafana password
```bash
curl -X PUT -d '{"oldPassword":"'$GRAFANA_OLDPASSWD'","newPassword":"'$GRAFANA_PASSWD'","confirmNew":"'$GRAFANA_PASSWD'"}' -H "Content-Type: application/json" -k $GRAFANA_ROUTE/api/user/password -b /tmp/grafana_cookies.txt
```
# Adding Loki datasource
```bash
curl -X POST -d '{"name":"Loki","type":"loki","access":"proxy","isDefault":true}' -H "Content-Type: application/json" -k $GRAFANA_ROUTE/api/datasources -b /tmp/grafana_cookies.txt
curl -X PUT -d '{"id":1,"orgId":1,"name":"Loki","type":"loki","typeLogoUrl":"","access":"proxy","url":"http://'$LOKI_STACK':3100","password":"","user":"","database":"","basicAuth":false,"basicAuthUser":"","basicAuthPassword":"","withCredentials":false,"isDefault":true,"jsonData":{},"secureJsonFields":{},"version":1,"readOnly":false}' -H "Content-Type: application/json" -k $GRAFANA_ROUTE/api/datasources/1 -b /tmp/grafana_cookies.txt
```
# Remove Grafana temp cookies
```bash
rm -f /tmp/grafana_cookies.txt
```
# Access Grafana UI
```bash
# Access with following route
echo $GRAFANA_ROUTE
# Login: admin
# Password: $GRAFANA_PASSWD (default: redhat)
```
# Reference
- https://grafana.com/oss/loki/
