An example secure SLP deployment on Kubernetes

**Goals**:
* Deploy SLP with a secure configuration as a scalable `StatefulSet`
* Deploy an App with an OPA sidecar that utilizes the SLP as a scalable `Deployment`

**Environment**:
* Minikube
* Styra DAS - Custom System type

## Steps
### 1. Start minikube

### 2. Create a Custom system in Styra DAS

### 3. Deploy the OPA config as a Secret for use by the SLP

Download the `opa-conf.yaml` from  Settings > Install

Create Secret from the `opa-conf.yaml`
```sh
k create secret generic slp-config --from-file=slp.yaml=opa-conf.yaml
```
> We rename the file to slp.yaml as this will be used by the SLP only (not OPA)

### 4. Deploy TLS Secret for SLP HTTPS

Create CA and Cert with `certstrap`. Alternatively you could use cert-manager to auto-generate the certs.
```sh
certstrap init --common-name "MyRootCA"
certstrap request-cert --domain "slp"
certstrap sign slp --CA MyRootCA
cat out/slp.crt out/MyRootCA.crt > out/slp-fullchain.crt
```

Create the Secret
```sh
k create secret tls slp-tls --cert=out/slp-fullchain.crt --key=out/slp.key
```

### 5. Deploy Styra Local Plane
Create a Token as a Secret for use in the SLP Authz policy
```sh
k create secret generic slp-authz-token --from-literal=token=12345-same-as-my-luggage
```

Deploy the SLP
```sh
k apply -f slp.yaml
```

### 6. Deploy App with OPA sidecar
Create Configmap for MyRootCA
```sh
k create configmap my-root-ca.crt --from-file=ca.crt=out/MyRootCA.crt
```

```sh
# **Replace the system-id value on line 11 of opa.yaml with the id from your DAS system**
k apply -f app-with-opa-sidecar.yaml
```

### 7. Create and test the policy
Add a Policy in DAS for `httpapi.authz` with the following contents:
```
package httpapi.authz

# bob is alice's manager, and betty is charlie's.
subordinates := {"alice": [], "charlie": [], "bob": ["alice"], "betty": ["charlie"]}

main["allow"] := allow

default allow := false

# Allow users to get their own salaries.
allow {
    input.method == "GET"
    input.path == ["finance", "salary", input.user]
}

# Allow managers to get their subordinates' salaries.
allow {
    some username
    input.method == "GET"
    input.path = ["finance", "salary", username]
    subordinates[input.user][_] == username
}
```

Test
```
# alice is allowed to view alice
curl --user alice:password $(minikube ip):30050/finance/salary/alice
# bob is allowed to view alice
curl --user bob:password $(minikube ip):30050/finance/salary/alice
# bos is NOT allowed to view charlie
curl --user bob:password $(minikube ip):30050/finance/salary/charlie
```
