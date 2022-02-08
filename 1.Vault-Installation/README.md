# 1. Deploying Vault

I elected to deploy Vault first, because then I can use it to generate the certificates for the other systems that should use TLS in an easy way. Of course, for the initial Vault this means generating a self signed CA that I'll then import.


### Generating the self signed key/cert pair for Vault

To start off, we'll use a self signed key/cert pair to communicate with the Vault, but that'll soon change.

```
$ cd certs
$ openssl req -x509 -newkey rsa:2048 -keyout vault.local.labs.andyrepton.com.pem -out vault.local.labs.andyrepton.com.crt -sha256 -days 30 -nodes -subj "/CN=vault.local.labs.andyrepton.com"
```

### Starting up Vault

```
$ cd ../1.Vault-Installation/vault-docker-compose
$ docker-compose up --detach
$ cd ../../
```

### Initialising the Vault

```
$ export VAULT_ADDR=https://vault.local.labs.andyrepton.com:8200

$ vault operator init -tls-skip-verify
Unseal Key 1: CpjJuoRNMm5r+AdBI6zoX/35NtqpJ2Rc9R1k2z+2eGAY
Unseal Key 2: ecpRoDA0viLbYO/SsidDgi4On0Fyv3huA0rxC8HbssrH
Unseal Key 3: etPSCdwRvlRRIn4iOYmj9kv2h2jz8fce762lToVjMYhy
Unseal Key 4: HB9xQ8Go8Rj07xoe5jfFtGU9PMgnfywDYrQE5IUc6ntD
Unseal Key 5: txRQ16QMTKpgKc8gpyfjKu2+Qx9hyyj7ocsKZiz5okCN

Initial Root Token: s.sFEN11yfl4sXCMOYOxwF8KQV

Vault initialized with 5 key shares and a key threshold of 3. Please securely
distribute the key shares printed above. When the Vault is re-sealed,
restarted, or stopped, you must supply at least 3 of these keys to unseal it
before it can start servicing requests.

Vault does not store the generated master key. Without at least 3 key to
reconstruct the master key, Vault will remain permanently sealed!

It is possible to generate new unseal keys, provided you have a quorum of
existing unseal keys shares. See "vault operator rekey" for more information.

$ export VAULT_TOKEN=s.sFEN11yfl4sXCMOYOxwF8KQV
```

### Unseal the Vault:

```
$ vault operator unseal -tls-skip-verify
Unseal Key (will be hidden):
Key                Value
---                -----
Seal Type          shamir
Initialized        true
Sealed             true
Total Shares       5
Threshold          3
Unseal Progress    1/3
Unseal Nonce       521a4d8d-dc8c-62f4-134e-18a4eabbdfeb
Version            1.9.2
HA Enabled         false
$ vault operator unseal -tls-skip-verify
Unseal Key (will be hidden):
Key                Value
---                -----
Seal Type          shamir
Initialized        true
Sealed             true
Total Shares       5
Threshold          3
Unseal Progress    2/3
Unseal Nonce       521a4d8d-dc8c-62f4-134e-18a4eabbdfeb
Version            1.9.2
HA Enabled         false
$ vault operator unseal -tls-skip-verify
Unseal Key (will be hidden):
Key             Value
---             -----
Seal Type       shamir
Initialized     true
Sealed          false
Total Shares    5
Threshold       3
Version         1.9.2
Cluster Name    vault-cluster-52d30705
Cluster ID      fb776a01-7cf9-091b-52f1-55b9f93b25e5
HA Enabled      false
```

Side note: as we'll be working with untrusted certs on Vault for a bit now, we can set the following for now:

```
$ export VAULT_SKIP_VERIFY=true
```

### Generate a root CA

Vault is able to act as our CA for the rest of this work, which is much better than using self signed certs, so let's do that next.

```
s cd ./certs
$ vault secrets enable pki
Success! Enabled the pki secrets engine at: pki/

# Tune the PKI mount
$ vault secrets tune -max-lease-ttl=87600h pki
Success! Tuned the secrets engine at: pki/

# Generate our Root CA
$ vault write -field=certificate pki/root/generate/internal \
     common_name="local.labs.andyrepton.com" \
     ttl=87600h > root_CA_cert.crt

# Configure our CA and CRL URLs
$ vault write pki/config/urls issuing_certificates="https://vault.local.labs.andyrepton.com/v1/pki/ca" crl_distribution_points="https://vault.local.labs.andyrepton.com/v1/pki/crl"
Success! Data written to: pki/config/urls
```

### Generate the intermediate CA

Ok ok, an intermediate for something this simple is probably overkill, but it's good practise to get into the habit of this in case your intermediate gets compromised.

```
$ vault secrets enable -path=pki_int pki
Success! Enabled the pki secrets engine at: pki_int/
```

Let's set the TTL a bit lower than our root:

```
$ vault secrets tune -max-lease-ttl=43800h pki_int
Success! Tuned the secrets engine at: pki_int/
```

Now, rather than generate our CA internally, we'll grab a csr and sign it with our root CA:

```
$ vault write -format=json pki_int/intermediate/generate/internal common_name="local.labs.andyrepton.com Intermediate Authority" | jq -r '.data.csr' > pki_intermediate.csr
$ vault write -format=json pki/root/sign-intermediate csr=@pki_intermediate.csr format=pem_bundle ttl="43800h"  | jq -r '.data.certificate' > intermediate.cert.pem
$ vault write pki_int/intermediate/set-signed certificate=@intermediate.cert.pem
Success! Data written to: pki_int/intermediate/set-signed
```

> **_Tip:_** This is how you would import an intermediate from your HSM if you have one

Finally for now, set the CA and CRL URLs:

```
$ vault write pki_int/config/urls issuing_certificates="https://vault.local.labs.andyrepton.com:8200/v1/pki_int/ca" crl_distribution_points="http://vault.local.labs.andyrepton.com:8200/v1/pki_int/crl"
Success! Data written to: pki_int/config/urls
```

### Configure our laptops to trust this new CA

If you are on Linux, the process here will be slightly different. Please open a PR if you're interested in writing the instructions!

For OS X:

```
$ sudo security add-trusted-cert -d -r trustRoot -k /System/Library/Keychains/SystemRootCertificates.keychain root_CA_cert.crt
```

### Create a role and new cert for Vault

Now we want to create a new PKI role and a key/cert pair for our Vault, so we can disable the skip of TLS verify. We'll start with the role, with the certs valid for a very large 720h, as Vault cannot regenerate these itself (or can it...? A workaround might be made later in this).

```
$ vault write pki_int/roles/vault-local-labs-andyrepton-com allowed_domains="vault.local.labs.andyrepton.com" allow_bare_domains=true allow_subdomains=false max_ttl=720h
```

And now that that's done, let's create a new key/cert pair!

```
$ vault write -format=json pki_int/issue/vault-local-labs-andyrepton-com common_name="vault.local.labs.andyrepton.com" tll="720h" | jq -r '.data.certificate, "\u0000", .data.issuing_ca, "\u0000", .data.private_key, "\u0000"' | { IFS= read -r -d '' certificate && printf '%s\n' "$certificate" > vault.local.labs.andyrepton.com.crt; IFS= read -r -d '' issuing_ca && printf '%s\n' "$issuing_ca" >> vault.local.labs.andyrepton.com.crt; IFS= read -r -d '' private_key && printf '%s\n' "$private_key" > vault.local.labs.andyrepton.com.pem; }
```

This will create a new key and cert pair for our vault. Now that's done, let's shut down the vault and use our new key/cert pairs!

```
$ cd ../1.Vault-Installation/vault-docker-compose/
$ docker-compose down
$ docker-compose up --detach
```

And test if we can now communicate with our Vault:

```
$ curl https://vault.local.labs.andyrepton.com:8200
<a href="/ui/">Temporary Redirect</a>.
$ unset VAULT_SKIP_VERIFY
$ vault operator unseal #Go do the unseal now
```

