# Import root CA certificate into Vault PKI Engine

## This repo shows an example of how to upload a certificate for the root CA in the Vault PKI engine. First, you will create a certificate for your root CA using openssl. Then you will upload the certificate into Vault PKI engine, and create the root CA within Vault. Finally, create a role and issue the certificate.

Why do we need this? Because the API call `/pki/intermediate/set-signed` does not support uploading a pem bundle.

### Generate root CA cert and key. Then format the pem file for upload to Vault.

```shell
mkdir opensslca
openssl genrsa -out opensslca/rootCAKey.pem 2048
openssl req -x509 -sha256 -new -nodes -key opensslca/rootCAKey.pem -days 3650 -out opensslca/rootCACert.pem
openssl x509 -in opensslca/rootCACert.pem -text

awk 'NF {sub(/\r/, ""); printf "%s\\n",$0;}' opensslca/rootCAKey.pem > rootCACertpayload.pem
awk 'NF {sub(/\r/, ""); printf "%s\\n",$0;}' opensslca/rootCACert.pem >> rootCACertpayload.pem
```

### Enable the PKI engine

```shell
vault secrets enable pki
vault secrets tune -max-lease-ttl=87600h pki
vault write pki/config/urls \
     issuing_certificates="$VAULT_ADDR/v1/pki/ca" \
     crl_distribution_points="$VAULT_ADDR/v1/pki/crl"
```

### Create `payload.json`

```shell
cat <<EOF >>payload.json
{
    "pem_bundle": "$(cat rootCACertpayload.pem)"
}
EOF
```

### Upload cert to Vault

```curl
curl \
    --header "X-Vault-Token: root" \
    --request POST \
    --data "@payload.json" \
    http://127.0.0.1:8200/v1/pki/config/ca
```

### Generate root CA in Vault

```shell
key_ref=$(curl \
    --header "X-Vault-Token: root" \
    http://127.0.0.1:8200/v1/pki/issuer/default | jq -r .data.key_id)
issuer_ref=$(curl \
    --header "X-Vault-Token: root" \
    http://127.0.0.1:8200/v1/pki/issuer/default | jq -r .data.issuer_id)

vault list pki/issuers/
vault read pki/issuer/$issuer_ref
vault write pki/root/generate/existing common_name="rootca.seesquared.local" issuer_name=reissued-root key_ref=$key_ref
```

### Create role and issue cert

```shell
vault write pki/roles/myapp \
     issuer_ref="$issuer_ref" \
     allowed_domains="seesquared.local" \
     allow_subdomains=true \
     max_ttl="720h"
vault write -format=json pki/issue/myapp common_name="myapp.seesquared.local" ttl="24h" > cert.json
jq -r .data.certificate cert.json > myappcert.crt
```

### View cert

```shell
openssl x509 -in myappcert.crt -text
```
