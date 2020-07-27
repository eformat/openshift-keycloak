# Keycloak OpenShift User Setup with GroupSync Operator
Quick steps to setup:

- Keycloak as an IDP for OCP4.5+
- Group Sync Operator

## Install Keycloak Operator
As cluster-admin:
```bash
oc new-project keycloak

cat <<EOF | oc apply -f -
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  generateName: keycloak-
  name: keycloak-og
spec:
  targetNamespaces:
  - keycloak
EOF

cat <<EOF | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: keycloak-operator
  namespace: keycloak
spec:
  channel: alpha
  installPlanApproval: Automatic
  name: keycloak-operator
  source: community-operators
  sourceNamespace: openshift-marketplace
  startingCSV: keycloak-operator.v10.0.0
EOF

cat <<EOF | oc apply -f -
apiVersion: keycloak.org/v1alpha1
kind: Keycloak
metadata:
  name: keycloak
  labels:
    app: sso
  namespace: keycloak
spec:
  extensions:
    - >-
      https://github.com/aerogear/keycloak-metrics-spi/releases/download/1.0.4/keycloak-metrics-spi-1.0.4.jar
  externalAccess:
    enabled: false
  instances: 1
  podDisruptionBudget:
    enabled: true
EOF
```

Create a reencrypt route (this requires you to have real certs for cluster e.g. LetsEncrypt in this example)
```bash
API=api.example.com

oc apply -n keycloak -f - <<EOF
apiVersion: v1
kind: Route
metadata:
  annotations:
    haproxy.router.openshift.io/balance: source
  labels:
    app: keycloak
  name: keycloak
  namespace: keycloak
spec:
  host: keycloak-keycloak.apps.example.com
  to:
    kind: Service
    name: keycloak
  tls:
    insecureEdgeTerminationPolicy: Redirect
    termination: reencrypt
    key: |-
$(sed 's/^/      /' /directory-to/.acme.sh/${API}/${API}.key)
    certificate: |-
$(sed 's/^/      /' /directory-to/.acme.sh/${API}/${API}.cer)
    caCertificate: |-
$(sed 's/^/      /' /directory-to/.acme.sh/${API}/fullchain.cer)
EOF
```

## Create a realm and client from Keycloak UI
Login to keycloak as admin using creds:
```
echo -e $(oc get secret credential-keycloak -n keycloak -o jsonpath='{.data.ADMIN_USERNAME}' | base64 -d)
echo -e $(oc get secret credential-keycloak -n keycloak -o jsonpath='{.data.ADMIN_PASSWORD}' | base64 -d)
```

Create an realm called `openshift`

Configure an OpenID-Connect Client
``` bash
name:     ocp-console
protocol: openid-connect
type:     confidential
redirect uris: https://oauth-openshift.apps.example.com/*, https://console-openshift-console.apps.example.com/
```

## Configure OpenShift OAUTH
From keycloak `ocp-console` client grab the credential secret and create a kube secret with it:
```bash
oc create secret generic idp-secret --from-literal=clientSecret=d015980b-32f1-40b8-9474-bcef2ff2b62c -n openshift-config
```
Create CA cert from the Router CA, we need `router-certs` secret from `openshift-ingress` namespace:
```bash
echo -e $(oc get secret router-certs -n openshift-ingress --template='{{index .data "tls.crt"}}') | base64 -d > /tmp/ca.crt
oc delete configmap ca-config-map -n openshift-config
oc create configmap ca-config-map --from-file=ca.crt=/tmp/ca.crt -n openshift-config
```

Grab the issuer from well known openid realm:
```
https://keycloak-keycloak.apps.example.com/auth/realms/openshift/.well-known/openid-configuration
```

Append your new Keycloak IDP provider (this assumes we already have htpasswd) to OAuth
```bash
cat <<EOF | oc apply -n openshift-config -f -
apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  name: cluster
spec:
  identityProviders:
  - name: htpassidp
    type: HTPasswd
    htpasswd:
      fileData:
        name: htpasswdidp-secret
  - mappingMethod: claim
    name: keycloak
    openID:
      ca:
        name: ca-config-map
      claims:
        email:
        - email
        name:
        - name
        preferredUsername:
        - preferred_username
      clientID: ocp-console
      clientSecret:
        name: idp-secret
      extraScopes: []
      issuer: https://keycloak-keycloak.apps.example.com/auth/realms/openshift
    type: OpenID
EOF
```

Watch progress
```bash
watch oc get clusteroperators authentication
```

Configure logout correctly
```bash
oc patch console.config.openshift.io cluster --type=merge -p '{"spec":{"authentication":{"logoutRedirect":"https://keycloak-keycloak.apps.example.com/auth/realms/openshift/protocol/openid-connect/logout?redirect_uri=https://console-openshift-console.apps.example.com/"}}}'
```

Watch progress
```bash
watch oc get clusteroperators console
```

## Add users and groups to Keycloak
From Keycloak UI, add users and groups locally (of course we can federate from all sorts of backends into Keycloak)
```bash
-- users
fred (group member of ocp-admins)
-- groups
ocp-admins
ocp-users
```

## Deploy Group Sync Operator

Deploy GroupSync operator:
```bash
oc new-project group-sync-operator

git clone https://github.com/redhat-cop/group-sync-operator.git
cd group-sync-operator
oc apply -f deploy/crds/redhatcop.redhat.io_groupsyncs_crd.yaml
oc apply -n group-sync-operator -f deploy/service_account.yaml
oc apply -n group-sync-operator -f deploy/clusterrole.yaml
oc apply -n group-sync-operator -f deploy/clusterrole_binding.yaml
oc apply -n group-sync-operator -f deploy/role.yaml
oc apply -n group-sync-operator -f deploy/role_binding.yaml
oc apply -n group-sync-operator -f deploy/operator.yaml
```

Create login credentials for keycloak (see above for user/pwd)
```bash
oc create secret generic keycloak-group-sync --from-literal=username=admin --from-literal=password=<password>
```

Deploy GroupSync configuration, including sync every minute cron:
```bash
cat <<EOF | oc apply -n group-sync-operator -f -
apiVersion: redhatcop.redhat.io/v1alpha1
kind: GroupSync
metadata:
  name: keycloak-groupsync
spec:
  schedule: "0-59 * * * *"
  providers:
  - name: keycloak
    keycloak:
      realm: openshift
      credentialsSecret:
        name: keycloak-group-sync
        namespace: group-sync-operator
      url: https://keycloak-keycloak.apps.example.com
EOF
```

Bind OpenShift roles to groups:
```bash
oc adm policy add-cluster-role-to-group cluster-admin ocp-admins
oc adm policy add-role-to-group admin ocp-users
```

Test Logging In and Logging Out.