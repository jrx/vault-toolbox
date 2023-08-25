# Demo restore a lost secret

## Setup secret engine and policies

```sh
# Setup mount point for K/V version 2
vault secrets enable -path=secret kv-v2

vault secrets list -detailed -format=json | jq '."secret/"'
vault secrets list -detailed -format=json | jq .

# Configure mount point with 
vault write secret/config max_versions=5

# Define the secret policy
tee /tmp/secret-policy.hcl <<EOF
path "secret/data/test" {
  capabilities = ["create", "update", "read", "delete", "patch"]
}
path "secret/metadata/test" {
  capabilities = ["list", "read"]
}
EOF

# Create the policy
vault policy write secret /tmp/secret-policy.hcl

# create user and assign the secret policy
vault auth enable userpass
vault write auth/userpass/users/jan \
  password="foo" \
  policies="secret"

```

## User workflow

```sh
# New terminal
unset VAULT_TOKEN

vault login -method=userpass \
  username="jan"

# Test capabilities
vault token lookup
vault token capabilities secret/data/test
vault token capabilities secret/metadata/test

# User creates a secret
vault kv put -mount=secret test user=hello password=vault
vault kv get -mount=secret test

# User rotates a secret
vault kv patch -mount=secret test password=consul
vault kv get -mount=secret test
vault kv get -field=password -mount=secret test

# User deletes a secret
vault kv delete -mount=secret test
vault kv get -mount=secret test
vault kv get -field=password -mount=secret test

# User cannot undelete a secret
vault kv undelete -versions=2 -mount=secret test
vault kv undelete -output-policy -versions=2 -mount=secret test

# User cannot destroy a secret
vault kv destroy -versions=2 -mount=secret test

# User cannot permanently delete all versions
vault kv metadata delete -mount=secret test

# User can get an older version
vault kv get -version=1 -mount=secret test

# User can rollback to an older version
vault kv rollback -version=1 -mount=secret test
vault kv get -mount=secret test
```

## Admin workflow

```sh
# Admin can list all versions
vault kv metadata get -format=json -mount=secret test \
  | jq .data.versions

# Admin can undeletes a secret version
vault kv undelete -versions=2 -mount=secret test
vault kv rollback -version=2 -mount=secret test
vault kv get -mount=secret test
```
