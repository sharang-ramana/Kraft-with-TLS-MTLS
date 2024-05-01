# Kraft with TLS MTLS [WIP]
## Security setup

In this workflow scenario, you'll set up a Confluent Platform cluster with the following security:
- Full TLS network encryption with user provided certificates
- mTLS authentication

## Prerequisites

Before continuing with the scenario, ensure that you have set up the prerequisites

* A Kubernetes cluster - any CNCF conformant version
* Helm 3 installed on your local machine
* Kubectl installed on your local machine
* A namespace created in the Kubernetes cluster - `confluent`
* Kubectl configured to target the `confluent` namespace:
  ```
  kubectl config set-context --current --namespace=confluent
  ```
* This repo cloned to your workstation:
  ```
  git@github.com:sharang-ramana/Kraft-with-TLS-MTLS.git
  ```

## Set the current tutorial directory

Set the tutorial directory for this tutorial under the directory you downloaded the tutorial files:

```
export TUTORIAL_HOME=<Tutorial directory>
```
  
## Deploy Confluent for Kubernetes

Set up the Helm Chart:

```
helm repo add confluentinc https://packages.confluent.io/helm
```

Install Confluent For Kubernetes using Helm:

```
helm upgrade --install confluent-operator confluentinc/confluent-for-kubernetes --namespace confluent --set kRaftEnabled=true
```
  
Check that the Confluent For Kubernetes pod comes up and is running:

```
kubectl get pods --namespace confluent
```

## Create TLS certificates

In this scenario, you'll configure authentication using the mTLS mechanism. With mTLS, Confluent components and clients use TLS certificates for authentication. The certificate has a CN that identifies the principal name.

Each Confluent component service should have it's own TLS certificate. In this scenario, you'll
generate a server certificate for each Confluent component service. Follow [these instructions](./assets/certs/component-certs/README.md) to generate these certificates.

These TLS certificates include the following principal names for each component in the certificate Common Name:
- Kraft: `kraft`
- Kafka: `kafka`
- Schema Registry: `sr`
- Kafka Connect: `connect`
- Kafka Rest Proxy: `krp`
- ksqlDB: `ksql`
- Control Center: `controlcenter`
     
## Deploy configuration secrets

you'll use Kubernetes secrets to provide credential configurations.

With Kubernetes secrets, credential management (defining, configuring, updating)
can be done outside of the Confluent For Kubernetes. You define the configuration
secret, and then tell Confluent For Kubernetes where to find the configuration.

To support the above deployment scenario, you need to provide the following
credentials:

* Component TLS Certificates

* Authentication credentials for Kafka, Control Center, remaining CP components

### Provide component TLS certificates

Set the tutorial directory for this tutorial under the directory you downloaded the tutorial files:

```
export TUTORIAL_HOME=<Tutorial directory>
```

In this step, you will create secrets for each Confluent component TLS certificates.

```
kubectl create secret generic tls-kraft \
  --from-file=fullchain.pem=$TUTORIAL_HOME/assets/certs/component-certs/generated/kraft-server.pem \
  --from-file=cacerts.pem=$TUTORIAL_HOME/assets/certs/component-certs/generated/cacerts.pem \
  --from-file=privkey.pem=$TUTORIAL_HOME/assets/certs/component-certs/generated/kraft-server-key.pem \
  --namespace confluent

kubectl create secret generic tls-kafka \
  --from-file=fullchain.pem=$TUTORIAL_HOME/assets/certs/component-certs/generated/kafka-server.pem \
  --from-file=cacerts.pem=$TUTORIAL_HOME/assets/certs/component-certs/generated/cacerts.pem \
  --from-file=privkey.pem=$TUTORIAL_HOME/assets/certs/component-certs/generated/kafka-server-key.pem \
  --namespace confluent

kubectl create secret generic tls-controlcenter \
  --from-file=fullchain.pem=$TUTORIAL_HOME/assets/certs/component-certs/generated/controlcenter-server.pem \
  --from-file=cacerts.pem=$TUTORIAL_HOME/assets/certs/component-certs/generated/cacerts.pem \
  --from-file=privkey.pem=$TUTORIAL_HOME/assets/certs/component-certs/generated/controlcenter-server-key.pem \
  --namespace confluent

kubectl create secret generic tls-schemaregistry \
  --from-file=fullchain.pem=$TUTORIAL_HOME/assets/certs/component-certs/generated/schemaregistry-server.pem \
  --from-file=cacerts.pem=$TUTORIAL_HOME/assets/certs/component-certs/generated/cacerts.pem \
  --from-file=privkey.pem=$TUTORIAL_HOME/assets/certs/component-certs/generated/schemaregistry-server-key.pem \
  --namespace confluent

kubectl create secret generic tls-connect \
  --from-file=fullchain.pem=$TUTORIAL_HOME/assets/certs/component-certs/generated/connect-server.pem \
  --from-file=cacerts.pem=$TUTORIAL_HOME/assets/certs/component-certs/generated/cacerts.pem \
  --from-file=privkey.pem=$TUTORIAL_HOME/assets/certs/component-certs/generated/connect-server-key.pem \
  --namespace confluent

kubectl create secret generic tls-ksqldb \
  --from-file=fullchain.pem=$TUTORIAL_HOME/assets/certs/component-certs/generated/ksqldb-server.pem \
  --from-file=cacerts.pem=$TUTORIAL_HOME/assets/certs/component-certs/generated/cacerts.pem \
  --from-file=privkey.pem=$TUTORIAL_HOME/assets/certs/component-certs/generated/ksqldb-server-key.pem \
  --namespace confluent

kubectl create secret generic tls-kafkarestproxy \
  --from-file=fullchain.pem=$TUTORIAL_HOME/assets/certs/component-certs/generated/kafkarestproxy-server.pem \
  --from-file=cacerts.pem=$TUTORIAL_HOME/assets/certs/component-certs/generated/cacerts.pem \
  --from-file=privkey.pem=$TUTORIAL_HOME/assets/certs/component-certs/generated/kafkarestproxy-server-key.pem \
  --namespace confluent
```

### Provide authentication credentials

Create a Kubernetes secret object for Control Center authentication credentials.

This secret object contains file based properties. These files are in the
format that each respective Confluent component requires for authentication
credentials.

```   
kubectl create secret generic credential \
  --from-file=basic.txt=$TUTORIAL_HOME/creds-control-center-users.txt \
  --namespace confluent
```

## Deploy Confluent Platform

Deploy Confluent Platform:

```
kubectl apply -f $TUTORIAL_HOME/confluent-platform.yaml --namespace confluent
```

Check that all Confluent Platform resources are deployed:

```
kubectl get pods --namespace confluent
```

If any component does not deploy, it could be due to missing configuration information in secrets.
The Kubernetes events will tell you if there are any issues with secrets. For example:  

If you see a **`CrashLoopBackOff`** on any of the components like Schema Registry / Connect or Control Center, this is **expected** as the ACLs were not yet created, continue to the next steps to add ACLs.  

```
kubectl get events --namespace confluent
Warning  KeyInSecretRefIssue  kafka/kafka  required key [ldap.txt] missing in secretRef [credential] for auth type [ldap_simple]
```
## Validate

### Validate in Control Center

Use Control Center to monitor the Confluent Platform, and see the created topic
and data. You can visit the external URL you set up for Control Center, or visit the URL
through a local port forwarding like below:

Set up port forwarding to Control Center web UI from local machine:

```
kubectl port-forward controlcenter-0 9021:9021 --namespace confluent
```

Browse to Control Center by portforwarding.

```
https://localhost:9021
```
or try to use the external url

## Tear down

```  
kubectl delete -f $TUTORIAL_HOME/confluent-platform-mtls-acls.yaml --namespace confluent

kubectl delete secret \
tls-kafka  tls-controlcenter tls-schemaregistry tls-connect tls-ksqldb credential \
--namespace confluent

helm delete operator --namespace confluent
```

## Appendix: Troubleshooting

### Gather data to troubleshoot

```
# Check for any error messages in events
kubectl get events --namespace confluent

# Check for any pod failures
kubectl get pods --namespace confluent

# For CP component pod failures, check pod logs
kubectl logs <pod-name> --namespace confluent
```