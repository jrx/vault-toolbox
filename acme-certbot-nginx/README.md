# Vault - ACME - Certbot - Nginx

Links:

- [Enable ACME with PKI secrets engine | Vault | HashiCorp Developer](https://developer.hashicorp.com/vault/tutorials/secrets-management/pki-acme-caddy)
- [User Guide - Certbot](https://eff-certbot.readthedocs.io/en/stable/using.html#changing-the-acme-server)

## Enable PKI

```sh
vault secrets enable pki
vault secrets tune -max-lease-ttl=87600h pki

vault write -field=certificate pki/root/generate/internal \
   common_name="learn.internal" \
   issuer_name="root-2023" \
   ttl=87600h > root_2023_ca.crt

vault write pki/config/cluster \
   path=https://127.0.0.1:8200/v1/pki \
   aia_path=https://127.0.0.1:8200/v1/pki

vault write pki/roles/2023-servers \
   allow_any_name=true \
   no_store=false

vault write pki/config/urls \
   issuing_certificates={{cluster_aia_path}}/issuer/{{issuer_id}}/der \
   crl_distribution_points={{cluster_aia_path}}/issuer/{{issuer_id}}/crl/der \
   ocsp_servers={{cluster_path}}/ocsp \
   enable_templating=true

vault secrets enable -path=pki_int pki
vault secrets tune -max-lease-ttl=43800h pki_int

vault write -format=json pki_int/intermediate/generate/internal \
   common_name="learn.internal Intermediate Authority" \
   issuer_name="learn-intermediate" \
   | jq -r '.data.csr' > pki_intermediate.csr

vault write -format=json pki/root/sign-intermediate \
   issuer_ref="root-2023" \
   csr=@pki_intermediate.csr \
   format=pem_bundle ttl="43800h" \
   | jq -r '.data.certificate' > intermediate.cert.pem

vault write pki_int/intermediate/set-signed certificate=@intermediate.cert.pem
vault write pki_int/config/cluster \
   path=https://127.0.0.1:8200/v1/pki_int \
   aia_path=https://127.0.0.1:8200/v1/pki_int

vault write pki_int/roles/learn \
   issuer_ref="$(vault read -field=default pki_int/config/issuers)" \
   allow_any_name=true \
   max_ttl="720h" \
   no_store=false

vault write pki_int/config/urls \
   issuing_certificates={{cluster_aia_path}}/issuer/{{issuer_id}}/der \
   crl_distribution_points={{cluster_aia_path}}/issuer/{{issuer_id}}/crl/der \
   ocsp_servers={{cluster_path}}/ocsp \
   enable_templating=true
```

## Configure ACME

```sh
vault read pki_int/config/cluster

vault secrets tune \
      -passthrough-request-headers=If-Modified-Since \
      -allowed-response-headers=Last-Modified \
      -allowed-response-headers=Location \
      -allowed-response-headers=Replay-Nonce \
      -allowed-response-headers=Link \
      pki_int

vault write pki_int/config/acme enabled=true
```

## Certbot - Nginx

```sh
sudo yum install nginx
sudo yum install certbot
sudo yum install python3-certbot-nginx

sudo systemctl enable nginx
sudo systemctl start nginx

sudo sh -c 'echo 127.0.0.1 test.learn.internal >> /etc/hosts'

sudo vim /etc/nginx/conf.d/test.learn.internal.conf
server {
        root /usr/share/nginx/html;
        index index.html index.htm;

        server_name test.learn.internal;

        location / {
                try_files $uri $uri/ =404;
        }
}

sudo systemctl reload nginx

sudo sh -c 'REQUESTS_CA_BUNDLE=/etc/vault.d/vault.ca certbot \
  --nginx \
  -d test.learn.internal \
  --server https://127.0.0.1:8200/v1/pki_int/acme/directory'

curl -k https://test.learn.internal
```

```sh
# check cert

echo | openssl s_client -showcerts -servername test.learn.internal -connect test.learn.internal:443 2>/dev/null | openssl x509 -inform pem -noout -text

curl --insecure -vvI https://test.learn.internal 2>&1 | awk 'BEGIN { cert=0 } /^\* SSL connection/ { cert=1 } /^\*/ { if (cert) print }'
```
