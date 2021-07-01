# Overview

A quick example of using NGINX as an independent sidecar proxy in front of your APIs hosted on OpenShift

# Usage

This example requires access to Iron Bank, DoD Centralized Artifacts Repository (DCAR), which hosts a trial NGINX Plus image for the advanced usage example below.

Prerequisites:

```
git clone https://github.com/liveaverage/openshift-nginx-sidecar.git
cd openshift-nginx-sidecar/

oc new-project sidecar

# Establish a pull secret for accessing Iron Bank
oc create secret docker-registry external-registry --docker-username=<your_username> --docker-password=<your_harbor_key> --docker-server=registry1.dso.mil
oc secrets link default external-registry --for=pull
```

Initial Deployment:
```
oc create configmap nginx --from-file=default.conf=nginx-sidecar-proxy.conf
oc create configmap jwk --from-file=api_secret.jwk=nginx-sidecar-api.jwk
oc apply -f nginx-sidecar-deployment.yaml 
```

Expose the NGINX service using OpenShift Router:

```
oc expose deployment sidecar --port=8080
oc expose service sidecar
export ROUTE=`oc get route sidecar --template={{.spec.host}}`
```

Updates:

```
# Recreate the NGINX configuration
oc delete cm nginx; oc create configmap nginx --from-file=default.conf=nginx-sidecar-proxy.conf

# Redeploy httpbin and your sidecar
oc scale deployment/sidecar --replicas=0; sleep 8; oc scale deployment/sidecar --replicas=1; sleep 3; oc get pods -o wide

# Access httpbin
echo "httpbin available at: http://${ROUTE}"

# Smoke test
curl -X GET "http://${ROUTE}/get" -H  "accept: application/json" -v
```

# Advanced Usage

> NOTE: Use of [JWT Authentication](https://docs.nginx.com/nginx/admin-guide/security-controls/configuring-jwt-authentication/) requires an NGINX Plus Release R10 or higher 

Using JWT as an API key, based on the example here: https://www.nginx.com/blog/authenticating-api-clients-jwt-nginx-plus/

```
# Recreate NGINX configuration and secure location /get
oc delete cm nginx; oc create configmap nginx --from-file=default.conf=nginx-sidecar-proxy-jwt.conf

# Redeploy httpbin and your sidecar
oc scale deployment/sidecar --replicas=0; sleep 8; oc scale deployment/sidecar --replicas=1; sleep 3; oc get pods -o wide

```

Test authentication:

```
## Test without JWT; confirm response 401 denial:
curl -X GET "http://${ROUTE}/get" -H  "accept: application/json" -v

## Test with JWT; confirm response 200:
curl -X GET "http://${ROUTE}/get" -H  "accept: application/json" -H "Authorization: Bearer `cat nginx-sidecar.jwt`" -v
```
