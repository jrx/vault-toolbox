# Vault - Login MFA with Userpass Fallback

Links:

- [Multi-Factor Authentication (MFA) for Login - Auth Methods | Vault | HashiCorp Developer](https://developer.hashicorp.com/vault/docs/auth/login-mfa)

Tested with Rocky Linux 9

## Start LDAP server

```sh
sudo su

yum install -y podman openldap-clients

podman run \
  --name vault-openldap \
  --env LDAP_ORGANISATION="example" \
  --env LDAP_DOMAIN="example.org" \
  --env LDAP_ADMIN_PASSWORD="admin" \
  -p 389:389 \
  -p 636:636 \
  --rm \
  docker.io/osixia/openldap:1.4.0

# verify
sudo su
podman ps -f name=vault-openldap --format "table {{.Names}}\t{{.Status}}"

cat > /tmp/base.ldif <<EOF
dn: ou=people,dc=example,dc=org
objectClass: organizationalUnit
ou: people

dn: ou=groups,dc=example,dc=org
objectClass: organizationalUnit
ou: groups
EOF

ldapadd -x -D cn=admin,dc=example,dc=org -W -f /tmp/base.ldif
# password: admin
```

## Configure LDAP Group and Users

```sh
# add user

cat > /tmp/user.ldif <<EOF
dn: cn=vault-users,ou=groups,dc=example,dc=org
objectClass: groupofnames
objectClass: top
description: testing group for vault-users
cn: vault-users
member: uid=jan,ou=people,dc=example,dc=org
member: uid=peter,ou=people,dc=example,dc=org

dn: cn=apps-admin,ou=groups,dc=example,dc=org
objectClass: groupofnames
objectClass: top
description: testing group for apps-admin
cn: apps-admin
member: uid=jan,ou=people,dc=example,dc=org
member: uid=peter,ou=people,dc=example,dc=org

dn: uid=jan,ou=people,dc=example,dc=org
objectClass: inetOrgPerson
objectClass: top
cn: jan
sn: jan
uid: jan
description: Test User for Jan
memberOf: cn=vault-users,ou=groups,dc=example,dc=org
memberOf: cn=apps-admin,ou=groups,dc=example,dc=org
userPassword: foo

dn: uid=peter,ou=people,dc=example,dc=org
objectClass: inetOrgPerson
objectClass: top
cn: peter
sn: peter
uid: peter
description: Test User for peter
memberOf: cn=vault-users,ou=groups,dc=example,dc=org
memberOf: cn=apps-admin,ou=groups,dc=example,dc=org
userPassword: foo
EOF

ldapadd -x -D cn=admin,dc=example,dc=org -W -f /tmp/user.ldif
```

## Configure Vault with OpenLDAP

```sh
# local
export VAULT_ADDR=https://127.0.0.1:8200
export VAULT_TOKEN=hvs....
export VAULT_SKIP_VERIFY=true
unset VAULT_NAMESPACE

vault auth enable ldap

vault write auth/ldap/config \
    url="ldap://localhost" \
    userattr=uid \
    userdn="ou=people,dc=example,dc=org" \
    groupdn="ou=people,dc=example,dc=org" \
    groupfilter="(&(objectClass=inetOrgPerson)(uid={{.Username}}))" \
    groupattr="memberOf" \
    binddn="cn=admin,dc=example,dc=org" \
    bindpass='admin' \
    insecure_tls=true \
    starttls=false

vault auth tune -listing-visibility='unauth' ldap
```

## Map LDAP Groups

### vault-users

```sh
vault write -format=json identity/group \
  name="vault-users" \
  policies="vault-users" \
  type="external" | jq -r ".data.id" > /tmp/group_vault_users_id.txt

vault auth list -format=json \
  | jq -r '.["ldap/"].accessor' \
  > /tmp/accessor_ldap.txt

vault write identity/group-alias \
  name="vault-users" \
  mount_accessor=$(cat /tmp/accessor_ldap.txt) \
  canonical_id="$(cat /tmp/group_vault_users_id.txt)"
```

### apps-admin

```sh
vault write -format=json identity/group \
  name="apps-admin" \
  policies="apps-admin" \
  type="external" | jq -r ".data.id" > /tmp/group_apps_admin_id.txt

vault auth list -format=json \
  | jq -r '.["ldap/"].accessor' \
  > /tmp/accessor_ldap.txt

vault write identity/group-alias \
  name="apps-admin" \
  mount_accessor=$(cat /tmp/accessor_ldap.txt) \
  canonical_id="$(cat /tmp/group_apps_admin_id.txt)"
```

## Pre-create Entities

### Entity: jan

```sh

vault write -format=json identity/entity \
  name="jan" | jq -r ".data.id" > /tmp/entity_id_jan.txt

vault auth list -format=json \
  | jq -r '.["ldap/"].accessor' \
  > /tmp/accessor_ldap.txt

vault write identity/entity-alias name="jan" \
     canonical_id=$(cat /tmp/entity_id_jan.txt) \
     mount_accessor=$(cat /tmp/accessor_ldap.txt)
```

### Entity: peter

```sh

vault write -format=json identity/entity \
  name="peter" | jq -r ".data.id" > /tmp/entity_id_peter.txt

vault auth list -format=json \
  | jq -r '.["ldap/"].accessor' \
  > /tmp/accessor_ldap.txt

vault write identity/entity-alias name="peter" \
     canonical_id=$(cat /tmp/entity_id_peter.txt) \
     mount_accessor=$(cat /tmp/accessor_ldap.txt)
```

## Configure MFA

### Enable login MFA method

```sh
vault write identity/mfa/method/totp \
    -format=json \
    method_name="my_totp" \
    generate=true \
    issuer=Vault \
    period=30 \
    key_size=30 \
    algorithm=SHA256 \
    digits=6 | jq -r '.data.method_id' \
    > /tmp/totp_method_id.txt

cat /tmp/totp_method_id.txt

vault read identity/mfa/method/totp/$(cat /tmp/totp_method_id.txt)
```

### Admin enforces MFA for the LDAP group

```sh
# Enforce MFA
vault write identity/mfa/login-enforcement/totp-ldap \
  mfa_method_ids="$(cat /tmp/totp_method_id.txt)" \
  identity_group_ids="$(cat /tmp/group_vault_users_id.txt)"

vault read identity/mfa/login-enforcement/totp-ldap
```

## Configure Userpass

```sh
vault auth enable userpass
vault auth tune -listing-visibility='unauth' userpass

vault auth list -format=json \
  | jq -r '.["userpass/"].accessor' \
  > /tmp/accessor_userpass.txt
  
tee /tmp/userpass-policy.hcl <<EOF
path "/identity/mfa/method/totp/generate" {
  capabilities = ["create", "read", "list", "update"]
  required_parameters = ["method_id"]
}
path "sys/auth" {  
  capabilities = [ "read" ]  
}
path "auth/userpass/users" {  
  capabilities = [ "list" ]  
}
path "auth/userpass/users/{{identity.entity.aliases.$(cat /tmp/accessor_userpass.txt).name}}" {  
  capabilities = [ "read", "list", "update" ]  
}
path "auth/userpass/users/{{identity.entity.aliases.$(cat /tmp/accessor_userpass.txt).name}}/password" {  
  capabilities = [ "update" ]  
  allowed_parameters = {  
    "password" = []  
  }  
}  
EOF

vault policy write userpass-policy /tmp/userpass-policy.hcl 
```

### User: jan

```sh
vault write auth/userpass/users/jan \
  token_policies=userpass-policy \
  password=bar

vault auth list -format=json \
  | jq -r '.["userpass/"].accessor' \
  > /tmp/accessor_userpass.txt

vault write identity/entity-alias name="jan" \
     canonical_id=$(cat /tmp/entity_id_jan.txt) \
     mount_accessor=$(cat /tmp/accessor_userpass.txt)
```

### User: peter

```sh
vault write auth/userpass/users/peter \
  token_policies=userpass-policy \
  password=bar

vault auth list -format=json \
  | jq -r '.["userpass/"].accessor' \
  > /tmp/accessor_userpass.txt

vault write identity/entity-alias name="peter" \
     canonical_id=$(cat /tmp/entity_id_peter.txt) \
     mount_accessor=$(cat /tmp/accessor_userpass.txt)
```

## Define Policy

### vault-users policy

```sh
vault secrets enable -path=users kv-v2

# Define the secret policy
tee /tmp/vault-users.hcl <<EOF
path "users/data/{{identity.entity.name}}/*" {
  capabilities = ["create", "update", "read", "delete", "patch"]
}

path "users/metadata/{{identity.entity.name}}/*" {
  capabilities = ["list", "read"]
}
path "users/metadata/*" {
  capabilities = ["list"]
}
EOF

vault policy write vault-users /tmp/vault-users.hcl
```

### apps-admin policy

```sh
vault secrets enable -path=apps kv-v2

# Define the secret policy
tee /tmp/apps-admin.hcl <<EOF
path "apps/*" {
  capabilities = ["sudo","read","create","update","delete","list","patch"]
}
EOF

vault policy write apps-admin /tmp/apps-admin.hcl
```

## Test Login and Create K/V 

```sh
export VAULT_ADDR=https://127.0.0.1:8200
export VAULT_SKIP_VERIFY=true
unset VAULT_TOKEN
unset VAULT_NAMESPACE
```

```sh
# Second login to Userpass
# password: bar
vault login -method=userpass username=jan

vault kv put -mount=users jan/secret1 user=moin password=consul
vault kv get -mount=users jan/secret1
vault kv list -mount=users jan

vault kv put -mount=users peter/secret3 user=hi password=nomad
vault kv get -mount=users peter/secret3
vault kv list -mount=users peter

vault kv put -mount=apps segment1/app1/secret5 user=hello password=vault
vault kv get -mount=apps segment1/app1/secret5
vault kv list -mount=apps segment1/app1/
```

### User generates OTP

```sh
vault write -field=barcode \
    identity/mfa/method/totp/generate \
    method_id=$(cat /tmp/totp_method_id.txt) \
    | base64 -d > /tmp/qr-code.png

open /tmp/qr-code.png
```

```sh
# First login to LDAP
# password: foo
vault login -method=ldap username=jan

vault kv put -mount=users jan/secret1 user=moin password=consul
vault kv get -mount=users jan/secret1
vault kv list -mount=users jan

vault kv put -mount=apps segment1/app1/secret5 user=hello password=vault
vault kv get -mount=apps segment1/app1/secret5
vault kv list -mount=apps segment1/app1/
```

### User changes password

```sh
# Create an example password policy
vault write sys/policies/password/example policy=-<<EOF
  length=20
  rule "charset" {
    charset = "abcdefghij0123456789"
    min-chars = 1
  }
  rule "charset" {
    charset = "!@#$%^&*STUVWXYZ"
    min-chars = 1
  }
EOF

# Setup mount point for K/V version 2
vault secrets enable -path=secret kv-v2

# Create the initial secret
vault kv put -mount=secret app/app_name \
  user=hello \
  password=$(vault read -field password sys/policies/password/example/generate)
# Test
vault kv get -mount=secret app/app_name
```

```sh
vault login -method=userpass username=jan
vault write auth/userpass/users/jan/password password=bar
```

## Admin resets the MFA for a User

### Option 1) Admin regenerates QR code

```sh

# Destroy MFA Secret
vault write identity/mfa/method/totp/admin-destroy \
  method_id=$(cat /tmp/totp_method_id.txt) \
  entity_id=$(cat /tmp/entity_id_jan.txt)

vault read -format=json identity/entity/id/$(cat /tmp/entity_id_jan.txt) | jq -r .data.mfa_secrets

# Generate authenticator app QR code
vault write -field=barcode \
  /identity/mfa/method/totp/admin-generate \
  method_id=$(cat /tmp/totp_method_id.txt) \
  entity_id=$(cat /tmp/entity_id_jan.txt) \
  | base64 -d > /tmp/qr-code.png

open /tmp/qr-code.png
```

### Option 2) Admin resets the entity to allow user to re-onboard

```sh
# Alternatively, you can recreate the entity, which increases the client count

# Delete entity
vault delete /identity/entity/id/$(cat /tmp/entity_id_jan.txt)

# Recreate entity
vault write -format=json identity/entity \
  name="jan" | jq -r ".data.id" > /tmp/entity_id_jan.txt

vault write identity/entity-alias name="jan" \
     canonical_id=$(cat /tmp/entity_id_jan.txt) \
     mount_accessor=$(cat /tmp/accessor_userpass.txt)

vault write identity/entity-alias name="jan" \
     canonical_id=$(cat /tmp/entity_id_jan.txt) \
     mount_accessor=$(cat /tmp/accessor_ldap.txt)
```
