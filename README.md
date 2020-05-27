# Install Vault

This script will install Vault in HA with raft based storage and auto-unseal. The auto-unseal process will used a non-ha seed vault for the initial secret.

## Preparation

Create vault namespace

```shell
git clone https://github.com/hashicorp/vault-helm
oc new-project vault
oc adm policy add-scc-to-user anyuid -z vault -n vault
export sguid=$(oc get project vault -o jsonpath='{.metadata.annotations.openshift\.io/sa\.scc\.supplemental-groups}'|sed 's/\/.*//')
export uid=$(oc get project vault -o jsonpath='{.metadata.annotations.openshift\.io/sa\.scc\.uid-range}'|sed 's/\/.*//')
```

Deploy cert-manager

```shell
oc new-project cert-manager
oc apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v0.15.0/cert-manager.yaml
```

Deploy needed certificates

```shell
oc apply -f ./vault-certs.yaml -n vault
```

## Install the seed-vault

```shell
helm template seed-vault ./vault-helm -n vault -f ./seed-values.yaml --set server.gid=${sguid} --set server.uid=${uid} | oc apply -f -
```

## Initialize seed-vault

this is a good tutorial: https://learn.hashicorp.com/vault/operations/raft-storage

```shell
SEED_INIT_RESPONSE=$(oc exec seed-vault-0 -n vault -- vault operator init -address https://seed-vault:8200 -ca-path /etc/vault-tls/seed-vault-tls/ca.crt -format=json -key-shares=1 -key-threshold=1)

SEED_UNSEAL_KEY=$(echo "$SEED_INIT_RESPONSE" | jq -r .unseal_keys_b64[0])
SEED_VAULT_TOKEN=$(echo "$SEED_INIT_RESPONSE" | jq -r .root_token)

echo "$SEED_UNSEAL_KEY"
echo "$SEED_VAULT_TOKEN"

#here we are saving these variable in a secret, this is probably not what you should do in a production environment
oc create secret generic seed-vault-init -n vault --from-literal=unseal_key=${SEED_UNSEAL_KEY} --from-literal=root_token=${SEED_VAULT_TOKEN}

#execute only this step after a restart of the pod
oc exec seed-vault-0 -n vault -- vault operator unseal -address https://seed-vault:8200 -ca-path /etc/vault-tls/seed-vault-tls/ca.crt "$SEED_UNSEAL_KEY"

oc exec seed-vault-0 -n vault -- sh -c "VAULT_TOKEN=${SEED_VAULT_TOKEN} vault secrets enable -address https://seed-vault:8200 -ca-path /etc/vault-tls/seed-vault-tls/ca.crt transit"
oc exec seed-vault-0 -n vault -- sh -c "VAULT_TOKEN=${SEED_VAULT_TOKEN} vault write -address https://seed-vault:8200 -ca-path /etc/vault-tls/seed-vault-tls/ca.crt -f transit/keys/unseal_key"
```

## Install HA-vault

```shell
export SEED_VAULT_TOKEN
envsubst < values.yaml.template > values.yaml
helm template vault ./vault-helm -n vault -f ./values.yaml --set server.gid=${sguid} --set server.uid=${uid} --set injector.gid=${sguid} --set injector.uid=${uid} | oc apply -f -
```

## Initialize HA-vault

```shell
HA_INIT_RESPONSE=$(oc exec vault-0 -n vault -- vault operator init -address https://vault-0.vault-internal:8200 -ca-path /etc/vault-tls/vault-tls/ca.crt -format=json -recovery-shares 1 -recovery-threshold 1)

HA_UNSEAL_KEY=$(echo "$HA_INIT_RESPONSE" | jq -r .recovery_keys_b64[0])
HA_VAULT_TOKEN=$(echo "$HA_INIT_RESPONSE" | jq -r .root_token)

echo "$HA_UNSEAL_KEY"
echo "$HA_VAULT_TOKEN"

#here we are saving these variable in a secret, this is probably not what you should do in a production environment
oc create secret generic vault-init -n vault --from-literal=unseal_key=${HA_UNSEAL_KEY} --from-literal=root_token=${HA_VAULT_TOKEN}
```

## Create Raft cluster for HA-vault

```shell
oc exec vault-1 -n vault -- sh -c 'vault operator raft join -address https://vault-1.vault-internal:8200 -ca-path /etc/vault-tls/vault-tls/ca.crt -leader-ca-cert="`cat /etc/vault-tls/vault-tls/ca.crt)`" https://vault-0.vault-internal:8200'
oc exec vault-2 -n vault -- sh -c 'vault operator raft join -address https://vault-2.vault-internal:8200 -ca-path /etc/vault-tls/vault-tls/ca.crt -leader-ca-cert="`cat /etc/vault-tls/vault-tls/ca.crt)`" https://vault-0.vault-internal:8200'
```

## Verify the cluster

```shell
oc exec vault-0 -n vault -- sh -c "VAULT_TOKEN=${HA_VAULT_TOKEN} vault operator raft list-peers -address https://vault-0.vault-internal:8200 -ca-path /etc/vault-tls/vault-tls/ca.crt"
```

## Expose Vault

```shell
oc get secret vault-tls -n vault -o jsonpath='{.data.ca\.crt}' | base64 -d > /tmp/ca.crt
oc create route reencrypt --service vault-ui -n vault --port https --dest-ca-cert /tmp/ca.crt
```

## Testing vault

```shell
export VAULT_ADDR=https://$(oc get route vault-ui -n vault -o jsonpath='{.spec.host}')
export VAULT_TOKEN=${HA_VAULT_TOKEN}
vault status -tls-skip-verify
```

to access the vault ui browse here:

```shell
echo $VAULT_ADDR/ui
```
