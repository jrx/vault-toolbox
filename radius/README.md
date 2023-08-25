# Vault - Auth - Radius

Links:

- [Getting Started](https://wiki.freeradius.org/guide/Getting%20Started)
- [LDAP - Auth Methods | Vault | HashiCorp Developer](https://developer.hashicorp.com/vault/docs/auth/ldap)

Tested with Rocky Linux 9

## Start LDAP server

```sh
sudo su
yum install -y podman

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
podman ps -f name=vault-openldap --format "table {{.Names}}\t{{.Status}}"

cat > /tmp/base.ldif <<EOF
dn: ou=people,dc=example,dc=org
objectClass: organizationalUnit
ou: people

dn: ou=groups,dc=example,dc=org
objectClass: organizationalUnit
ou: groups
EOF

yum install -y openldap-clients
ldapadd -x -D cn=admin,dc=example,dc=org -W -f /tmp/base.ldif
# password: admin
```

## Install Radius

```sh
sudo su
yum install -y freeradius freeradius-utils freeradius-ldap

cd  /etc/raddb/certs
./bootstrap

systemctl enable --now radiusd.service
```

## Configure Radius for LDAP

```sh
ln -s /etc/raddb/mods-available/ldap ../mods-enabled/ldap
unlink /etc/raddb/mods-enabled/eap
```

```sh
# client: vault
vim /etc/raddb/clients.conf

client localhost {
    ipaddr = 127.0.0.1
    secret = testing123
}
```

```sh

```sh
vim /etc/raddb/mods-enabled/ldap

ldap {
  server = "localhost"
  basedn = "dc=example,dc=org"
  identity = 'cn=admin,dc=example,dc=org'
  password = 'admin'
    user {
        filter = "(uid=%{%{Stripped-User-Name}:-%{User-Name}})"
    }
}


unlink /etc/raddb/sites-enabled/default
unlink /etc/raddb/sites-enabled/inner-tunnel

vim /etc/raddb/sites-enabled/my_server
server my_server {
listen {
        type = auth
        ipaddr = *
        port = 1812
}
authorize {
        ldap
        if (ok || updated)  {
        update control {
        Auth-Type := ldap
        }
        }
}
authenticate {
        Auth-Type LDAP {
                ldap
        }
}
}

```

```sh
systemctl restart radiusd
tail -f /var/log/radius/radius.log
```

```sh
# radius user

cat > /tmp/user.ldif <<EOF
dn: cn=dev,ou=groups,dc=example,dc=org
objectClass: groupofnames
objectClass: top
description: testing group for dev
cn: dev
member: uid=jan,ou=people,dc=example,dc=org

dn: uid=jan,ou=people,dc=example,dc=org
objectClass: inetOrgPerson
objectClass: top
cn: jan
sn: jan
uid: jan
description: Test User
memberOf: cn=dev,ou=groups,dc=example,dc=org
userPassword: foo
EOF

ldapadd -x -D cn=admin,dc=example,dc=org -W -f /tmp/user.ldif

radtest "jan" "foo" localhost 18120 testing123
```

## Configure Vault with Radius Auth

```sh
vault auth enable radius

vault write auth/radius/config \
  host="127.0.0.1" \
  port=1812 \
  secret="testing123"


vault write auth/radius/users/jan policies=default

vault login -method=radius username=jan
#ldap password: foo
```
