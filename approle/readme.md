# approle steps

## Setup shell

```sh
export policy=app1
export secret_name=hello

export VAULT_ADDR=https://$(oc get route vault-ui -n vault -o jsonpath='{.spec.host}')
export VAULT_TOKEN=$(oc get secret vault-init -n vault -o jsonpath='{.data.root_token}' | base64 -d)

vault kv put -tls-skip-verify secret/${policy}/${secret_name} foo=world excited=yes
```

## Step 1: Enable AppRole auth method

```sh

vault auth enable -tls-skip-verify approle
```

## Step 2: Create a role with policy attached

```sh
vault policy write -tls-skip-verify ${policy} - <<EOF
# Read-only permission on 'secret/${policy}/*' path
path "secret/${policy}/*" {
  capabilities = [ "read" ]
}
EOF

vault write -tls-skip-verify auth/approle/role/${policy} token_policies=${policy} token_ttl=1h token_max_ttl=4h

vault read -tls-skip-verify auth/approle/role/${policy}
```

## Step 3: Get RoleID and SecretID

```sh
export ROLE_ID=$(vault read -tls-skip-verify -field=role_id auth/approle/role/${policy}/role-id)

export SECRET_ID=$(vault write -tls-skip-verify -field=secret_id -f auth/approle/role/${policy}/secret-id)
```

## Step 4: Login with RoleID & SecretID

```sh
VAULT_TOKEN=$(vault write -field=token -tls-skip-verify auth/approle/login role_id=${ROLE_ID} secret_id=${SECRET_ID})

vault kv get -tls-skip-verify secret/${policy}/${secret_name}
```
