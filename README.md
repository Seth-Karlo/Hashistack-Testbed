# Hashistack Testbed

Learning how to roll a best practise Hashistack (Terraform, Consul, Vault, Nomad)

Please note: this is _not_ an experiment to build the infrastructure alone, but to configure Consul, Nomad and Vault using Terraform as well. If you're only interested in deploying a working cluster, there are some great modules available at github.com/hashicorp/

** If you're experiencing any issues, please check out the TROUBLESHOOTING.md file**

Disclaimer: this is an experiment for myself to build a new system from scratch to keep my skills up to date, I make no promises as to how secure this is! Please review before copying code.

There are various folders here that gradually add more complicated Hashistack functionality. You can start at any folder; the changes are additive.

# Contents

1. Installing the tools
2. Deploying Vault, Consul and Nomad leaders
3. Deploying a Nomad agent
4. Deploying an app using raw_exec
5. Configuring Consul and Nomad to use Vault for tokens
6. Deploying an app with templating to use Vault
7. Enabling containerd and CNI on my Nomad Agent
8. Enabling Consul ACLs
9. Deploying an app with a service stanza
10. Enabling Nomad ACLs


# Installation and some ground rules

I've found that over the years, despite being heavily involved in containers and cloud native systems, I've yet to really use docker-compose in anything but anger, and I wanted to get some experience with it. I've also gotten a little bit older, and the idea of spinning up half a dozen servers on AWS just to test some stuff raised the old scottish blood in me about wasting money, so I've elected to use docker-compose for this little project.

Also, as installation of tools should be relatively simple using containers, I've decided to use those as much as I can over installing tools using a package manager

One final rule I've set myself is to avoid bash scripts. I'd like this to be clean, understandable and if I need to run through a list of commands to make something work, then I'll document how that's done. The reader is welcome to automate these steps away with the tool of their choosing if they wish!

### Fixing my Hosts file:

Throughout this tutorial I'll be using entries at local.labs.andyrepton.com. As we'll be using SSL and such during this we'll need to match the hostnames and these will be written into various files. You're more than welcome to use `local.labs.andyrepton.com` yourself, I promise it won't resolve to anything on the actual public internet.

```
127.0.0.1 kubernetes.docker.internal vault.local.labs.andyrepton.com consul.local.labs.andyrepton.com nomad.local.labs.andyrepton.com
```

## 1. Deploying Vault

See 1.Vault-Installation/README.md

