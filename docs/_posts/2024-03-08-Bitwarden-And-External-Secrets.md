---
title: "Bitwarden and External Secrets"
date: 2024-03-08 00:00:00 -0000
excerpt_separator: "<!--more-->"
categories:
  - Technical Blog
tags:
  - homelab
  - kubernetes
header:
  image: /assets/images/Bitwarden External Secrets - Connecting to CLI.png
---

I am a massive fan of [Bitwarden](https://bitwarden.com/). It’s my preferred password manager.

It’s open source and can be self-hosted. However, I prefer to offload that responsibility to them by using their SaaS offering. I have the same stance when it comes to secrets management for Kubernetes. I avoid running my own secrets store (e.g. Hashicorp Vault) because the solution is so complex yet crucial that I don’t want to be responsible for it when it breaks. I’ve been in that position before, and troubleshooting was always painful and not worth the effort in the end. Nowadays, I prefer to pay for a service with a team of engineers responsible for keeping the solution stable and secure by offloading that responsibility onto cloud providers by utilizing services like [Google Secret Manager](https://cloud.google.com/security/products/secret-manager). So, when I discovered that External Secrets could integrate with Bitwarden, I was ecstatic.

If you are unaware, [**External Secrets**](https://external-secrets.io/latest/) is a Kubernetes operator that integrates secret management systems (e.g. Vault, AWS Secrets Manager, or GCP Secrets Manager). Based on `ExternalSecret` objects in the cluster, it connects to external `SecretStores` to retrieve the secrets and then populates Kubernetes Secrets with that data. The `ExternalSecret` object is perfectly safe to store in Git since it’s only a reference to the real secret values.

![Diagram of ExternalSecret to Kubernetes Secret](/assets/images/Bitwarden%20External%20Secrets%20-%20ExternalSecret.png)

I had intended to utilize GCP Secrets Manager as my `SecretStore` for my bare-metal Kubernetes cluster. It’s relatively cheap for my use case, likely costing me under $1 per month for the number of secrets and API calls my cluster would make. However, I already pay for and regularly use Bitwarden to manage my passwords, so it felt natural to use it for my Kubernetes secrets.

# Implementation

Bitwarden is not one of the [official providers](https://external-secrets.io/latest/provider/aws-secrets-manager/) for External Secrets. Instead, the integration uses the [webhook provider](https://external-secrets.io/latest/provider/webhook/) and the Bitwarden CLI (bw). Using [the bw serve command](https://bitwarden.com/help/cli/#serve), the Bitwarden CLI can spin up a local webserver that will accept any command accessible from the CLI in the form of RESTful API calls. The External Secrets webhook provider can then connect to and request values from items in the Bitwarden vault.

![Diagram of External Secrets connecting to Bitwarden CLI](/assets/images/Bitwarden%20External%20Secrets%20-%20Connecting%20to%20CLI.png)

There is a guide in the External Secrets docs on [“Bitwarden support using webhook provider](https://external-secrets.io/latest/examples/bitwarden/)”. This post will cover most of the same implementation, but I will go into my experience and more detail here.

## The Bitwarden CLI Container Image

The first challenge is that Bitwarden does **not** provide a container image with the CLI, so you must build one yourself. I maintain a GitHub repository with the Dockerfile, entry point script, and published container images here: [https://github.com/morey-tech/container-bitwarden-cli](https://github.com/morey-tech/container-bitwarden-cli)

```docker
FROM debian:sid
ENV BW_CLI_VERSION=2024.2.0
RUN apt update && \
    apt install -y wget unzip && \
    wget https://github.com/bitwarden/clients/releases/download/cli-v${BW_CLI_VERSION}/bw-linux-${BW_CLI_VERSION}.zip && \
    unzip bw-linux-${BW_CLI_VERSION}.zip && \
    chmod +x bw && \
    mv bw /usr/local/bin/bw && \
    rm -rfv *.zip
COPY entrypoint.sh /
RUN chmod +x /entrypoint.sh
CMD ["/entrypoint.sh"]
```

The Dockerfile is pretty straight-forward. It will fetch the Bitwarden CLI from the official releases on GitHub at the specified version, install it into the image, copy in the entrypoint script, and make it executable.

```bash
#!/bin/bash
set -e
bw config server ${BW_HOST}
bw login --apikey
export BW_SESSION=$(bw unlock ${BW_PASSWORD} --raw)
bw unlock --check
echo 'Running `bw server` on port 8087'
bw serve --hostname 0.0.0.0
```

The entry point script will:

- Configure the server with the host specified in the `BW_HOST` environment variable. If unset, it will default to the host for the Bitwarden SaaS offering.
- Authenticate the bw CLI using the API key specified in the environment variables. We’ll come back to this in a moment.
- Unlock the vault using the master password.
- Then, it spins up the local webserver using bw serve.

## Security

Oh yes, you read that correctly; the entry point script requires the master password for the Bitwarden account to function. In this case, it is stored in a Kubernetes Secret, which left me nervous about the solution. If a threat actor got access to the Kubernetes cluster and permissions to read secrets in the external-secrets namespace, they would gain access to the master password. Of course, this is unlikely given that the cluster is in a private network, behind the authentication and authorization of the Kubernetes api-server.

If they have already achieved that level of access, they could hit the Bitwarden webhook to access the secrets anyway. Using a `NetworkPolicy` to limit ingress traffic to only the external-secrets pods, I can mitigate that problem.

```yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: external-secret-2-bw-cli
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/instance: bitwarden-cli
      app.kubernetes.io/name: bitwarden-cli
  ingress:
    - from:
      - podSelector:
          matchLabels:
            app.kubernetes.io/instance: external-secrets
            app.kubernetes.io/name: external-secrets
```

But, still, it made me uncomfortable. Having the master password to my personal Bitwarden account in my Kubernetes cluster, where I run services that are exposed to the open internet, is **not** something I was willing to do. However, I found a rather elegant compromise that worked out better overall.

## Account Setup

I’m already subscribed to the Bitwarden Families plan, which, for $3.33 per month, includes up to 6 users, of which I was using 2. The family plan is fantastic because you get access the organization feature to share password entries between users. In each organization, you can have collections of passwords with fine-grained permission control.

I’m sure you can see where I’m going with this; I created a Bitwarden user dedicated to the Kubernetes cluster. I added it to my organization, which gave it a premium subscription at no cost. Then, I created a collection in the organization dedicated to the cluster and gave that account read-only access.

![Diagram of Bitwarden account setup](/assets/images/Bitwarden%20External%20Secrets%20-%20Account%20Setup.png)

With this setup, I can create an entry in the collection from my primary personal Bitwarden account. And external-secrets in my Kubernetes cluster can retrieve values from that entry to populate Secrets used by services in the cluster.

## Kubernetes Configuration

I’m a big fan of kustomize+helm, I think it’s the perfect combination of templating and last mile patching. But that’s a topic for another blog post. I use this combination for the deployment of external-secrets and the bitwarden-cli container I covered earlier.

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: external-secrets-system
resources:
- bitwarden-cli-deploy.yaml
- secret-stores.yaml

helmCharts:
- name: external-secrets
  includeCRDs: true
  version: 0.9.0
  repo: https://charts.external-secrets.io
  releaseName: external-secrets
  namespace: external-secrets-system
  valuesInline:
    resources:
      requests:
        cpu: 100m
        memory: 256Mi

images:
- name: ghcr.io/morey-tech/bitwarden-cli
  newTag: v0.2.0
```

This kustomization.yaml will deploy:

- The `external-secrets` Helm chart to the `external-secrets-system` namespace. The `-system` suffix is specific to my environment.
- A `Deployment`, `Service`, and `NetworkPolicy` (as covered earlier) for the `bitwarden-cli` container with the tag `v0.2.0`.
- Three ClusterSecretStore resources that use the webhook provider to configure External Secrets to use the `bitwarden-cli` deployment. One of them is used to retrieve the standard username and password property, one is used to retrieve custom field properties, and the other is used to retrieve the note property. Each of these is a separate JSON path on the webhook, which is why they are separate secret stores.

All of which can be committed into Git and deployed by Argo CD.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: external-secrets
  namespace: argocd
spec:
  destination:
    namespace: external-secrets-system
    server: 'https://kubernetes.default.svc'
  project: default
  source:
    path: environments/rubrik/system/external-secrets
    repoURL: 'https://github.com/morey-tech/homelab.git'
    targetRevision: HEAD
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - allowEmpty=true
    - CreateNamespace=true
```

What can’t be committed into the Git repository, is the Secret used by the `bitwarden-cli` `Deployment` to authenticate and unseal the Bitwarden vault. This is [one of two manual bootstrapping steps](https://github.com/morey-tech/homelab/tree/main/environments/rubrik#how-to-bootstrap-the-cluster) that I have yet to automate. But it looks like this:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: bitwarden-cli
  namepsace: external-secrets-system
type: Opaque
data:
  # The master password for the account.
  BW_PASSWORD:
  # https://bitwarden.com/help/personal-api-key/
  BW_CLIENTID:
  BW_CLIENTSECRET: 
```

# Bring It All Together

This diagram brings it all together to show the full flow from when an `ExternalSecret` is created to editing secrets with my personal Bitwarden User:

![Full flow diagram of Bitwarden and ExternalSecrets](/assets/images/Bitwarden%20External%20Secrets%20-%20Bring%20it%20all%20together.png)

## Usage

With External Secrets deployed with the Bitwarden CLI alongside it, I am ready to populate Secrets in my cluster with values from password entires in my Bitwarden vault (specifically the collection in my organization). Using the `ExternalSecret` resource, I can define what the generated Secret should look like:

```yaml
{% raw %}
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: example-secret
spec:
  target:
    name: example-secret
    deletionPolicy: Delete
  template:
    type: Opaque
    data:
      username: |-
        {{ .user }}
      password: |-
        {{ .pass }}
      testKey: |-
        {{ .testKey }}
```

The keys in the double brackets (`{{ .key }}`) are references to the data section, which is where I defined which Bitwarden entry to take values from.

```yaml
spec:
  # ...
  data:
  - secretKey: user
    sourceRef:
      storeRef:
        name: bitwarden-login
        kind: ClusterSecretStore
    remoteRef:
      key: f98dbd25-e105-4555-c026-d121910ac03f
      property: username
```

In this example, using the `bitwarden-login` `ClusterSecretStore`, the `{{ .user }}` key is populated with the username property from the Bitwarden vault entry with the ID used in the `remoteRef.key`. Unfortunately, I can not find a way to retrieve the ID of an entry from the browser extension (though not surprising). So, to get this value, I have to log into the [Bitwarden web vault](https://vault.bitwarden.com/#/login), view the desired entry, and copy it from the `itemId=` attribute in the URL.
{% endraw %}


```yaml
https://vault.bitwarden.com/#/vault?itemId=f98dbd25-e105-4555-c026-d121910ac03f
```

Thankfully, because I’m using a separate Bitwarden account for External Secrets, I can keep that master password for it in my personal Bitwarden, making logging into the web vault a breeze. And, once I have that ID, I can paste it into the Bitwarden browser extension logged into my personal account to pull up the entry used in the `ExternalSecret`.

![Screenshot of ExternalSecret in Argo CD](/assets/images/Xnapper-2024-03-06-17.56.40.png)

# Conclusion

I wouldn’t recommend this approach for any significant production workload. Frankly, I may even out grow this solution and need to transition to proper secrets manager (e.g. one from [Bitwarden](https://bitwarden.com/products/secrets-manager/) or Google) in the future. But for a single cluster in a homelab environment where you are already a Bitwarden user, it’s a super efficient secrets management solution.