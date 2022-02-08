# 2. Deploying Consul

Deploying Consul should now be relatively trivial with Vault running. We'll make a new role to generate the certs, then start it up

```
$ cd 2.Consul-Installation
```

### Generating the key/cert pairs

We're now going to make a second role for local.labs.andyrepton.com, with a much shorter ttl as these certs will be regularly renewed.

```
$ vault write pki_int/roles/local-labs-andyrepton-com allowed_domains="local.labs.andyrepton.com" allow_subdomains=true max_ttl=48h
Success! Data written to: pki_int/roles/local-labs-andyrepton-com
```

And now generate 3 key/cert pairs for our Consul nodes:

```
$ for X in 1 2 3; do vault write -format=json pki_int/issue/local-labs-andyrepton-com common_name="consul${X}.local.labs.andyrepton.com" alt_names="server.local.labs.andyrepton.com" ttl="720h" | jq -r '.data.certificate, "\u0000", .data.issuing_ca, "\u0000", .data.private_key, "\u0000"' | { IFS= read -r -d '' certificate && printf '%s\n' "$certificate" > consul${X}.local.labs.andyrepton.com.crt; FS= read -r -d '' issuing_ca && printf '%s\n' "$issuing_ca" >> consul${X}.local.labs.andyrepton.com.crt; IFS= read -r -d '' private_key && printf '%s\n' "$private_key" > consul${X}.local.labs.andyrepton.com.pem; } ; done
```

Note: we need to add the alt_names here as Consul uses this to determine if a node is allowed to join the cluster as a leader.

### Starting up Consul

Now we have our certs, we can start up our consul cluster:

```
$ cd consul-docker-compose
$ docker-compose up --detach
```

