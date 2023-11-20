# Vault - ACME - Cert-Manager

Links:

- [Enable ACME with PKI secrets engine | Vault | HashiCorp Developer](https://developer.hashicorp.com/vault/tutorials/secrets-management/pki-acme-caddy)
- [Cert Manager - Helm](https://cert-manager.io/docs/installation/helm/)
- [Cert Manager - ACME](https://cert-manager.io/docs/configuration/acme/)
- [How To Set Up an Nginx Ingress on DigitalOcean Kubernetes Using Helm](https://www.digitalocean.com/community/tutorials/how-to-set-up-an-nginx-ingress-on-digitalocean-kubernetes-using-helm#step-4-securing-the-ingress-using-cert-manager)
- [Cert-manager "remote error: tls: unrecognized name" errors](https://stackoverflow.com/questions/71181517/cert-manager-remote-error-tls-unrecognized-name-errors)

## Enable PKI

```sh
export VAULT_LB=https://<vault-addr>:8200

vault secrets enable pki
vault secrets tune -max-lease-ttl=87600h pki

vault write -field=certificate pki/root/generate/internal \
   common_name="eu-north-1.elb.amazonaws.com" \
   issuer_name="root-2023" \
   ttl=87600h > root_2023_ca.crt

vault write pki/config/cluster \
   path=${VAULT_LB}/v1/pki \
   aia_path=${VAULT_LB}/v1/pki

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
   common_name="eu-north-1.elb.amazonaws.com Intermediate Authority" \
   issuer_name="vault-intermediate" \
   | jq -r '.data.csr' > pki_intermediate.csr

vault write -format=json pki/root/sign-intermediate \
   issuer_ref="root-2023" \
   csr=@pki_intermediate.csr \
   format=pem_bundle ttl="43800h" \
   | jq -r '.data.certificate' > intermediate.cert.pem

vault write pki_int/intermediate/set-signed certificate=@intermediate.cert.pem
vault write pki_int/config/cluster \
   path=${VAULT_LB}/v1/pki_int \
   aia_path=${VAULT_LB}/v1/pki_int

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

vault read pki_int/config/cluster
```

## Configure ACME

```sh
vault secrets tune \
      -passthrough-request-headers=If-Modified-Since \
      -allowed-response-headers=Last-Modified \
      -allowed-response-headers=Location \
      -allowed-response-headers=Replay-Nonce \
      -allowed-response-headers=Link \
      pki_int

vault write pki_int/config/acme enabled=true

# Export Vault CA
export VAULT_CA=$(sudo -u vault cat /etc/vault.d/vault.ca | base64 -w 0)
```

## Install Cert Manager

```sh
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.2/cert-manager.crds.yaml

kubectl create namespace cert-manager

helm repo add jetstack https://charts.jetstack.io

helm repo update
helm search repo jetstack/cert-manager

helm install cert-manager \
    --namespace cert-manager \
    --version v1.13.2 \
   jetstack/cert-manager
```

## Configure an issuer

```yaml
cat > /tmp/cluster-issuer.yaml <<EOF
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: vault-acme
spec:
  acme:
    server: ${VAULT_LB}/v1/pki_int/acme/directory
    caBundle: ${VAULT_CA}
    privateKeySecretRef:
      # Secret resource that will be used to store the account's private key.
      name: example-issuer-account-key
    # Add a single challenge solver, HTTP01 using nginx
    solvers:
    - http01:
        ingress:
          ingressClassName: nginx
EOF

kubectl apply -f /tmp/cluster-issuer.yaml -n cert-manager

kubectl get clusterissuer.cert-manager.io -n cert-manager
kubectl describe clusterissuer.cert-manager.io/vault-acme -n cert-manager
```

## Install Nginx Ingress

```sh
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm search repo ingress-nginx/ingress-nginx

kubectl create ns ingress

helm install ingress-nginx \
  --namespace ingress \
  --timeout 600s \
  --set controller.publishService.enabled=true \
  --version 4.8.3 \
  ingress-nginx/ingress-nginx
```

## Generate the certificate

```yaml
cat > /tmp/hello-deploy.yaml <<EOF
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-world
  labels:
    app: hello-world
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hello-world
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      containers:
      - name: echo-server
        image: hashicorp/http-echo
        args:
        - -listen=:8080
        - -text="Hello from Kubernetes!"
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 250m
            memory: 256Mi
---
apiVersion: v1
kind: Service
metadata:
  name: hello-world
spec:
  selector:
    app: hello-world
  ports:
    - port: 80
      targetPort: 8080
EOF

kubectl create -f /tmp/hello-deploy.yaml -n ingress
```

```sh
export NGINX_LB=$(kubectl get service/ingress-nginx-controller -n ingress -o json | jq -r ' .status.loadBalancer.ingress[0].hostname')

cat > /tmp/hello-ingress.yaml <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /\$1
    cert-manager.io/cluster-issuer: vault-acme
  name: hello-world-ingress
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - ${NGINX_LB}
    secretName: hello-kubernetes-tls
  rules:
    - host: ${NGINX_LB}
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: hello-world
                port:
                  number: 80
EOF

kubectl create -f /tmp/hello-ingress.yaml -n ingress

kubectl get ingress -n ingress
kubectl describe certificate hello-kubernetes-tls -n ingress
```

## Verify the certificate

```sh
# Test connection
curl -k "https://${NGINX_LB}"

# Check certificate
echo | openssl s_client -showcerts -servername ${NGINX_LB} -connect ${NGINX_LB}:443 2>/dev/null | openssl x509 -inform pem -noout -text

curl --insecure -vvI "https://${NGINX_LB}" 2>&1 | awk 'BEGIN { cert=0 } /^\* SSL connection/ { cert=1 } /^\*/ { if (cert) print }'
```
