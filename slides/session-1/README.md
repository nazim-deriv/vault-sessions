# Session-1:

This session covers following topics:
- What is vault
- Pre-requisites to set it up on kubernetes cluster
- Setting it up in the cluster
- Vault unsealing / sealing
- Storing secrets in vault
- Deploying application and injecting secret from vault

---
## What is vault
HashiCorp Vault is an open-source tool designed for securely storing, accessing, and managing secrets, such as passwords, API keys, and cryptographic keys.

---
## Pre-requisites to set it up on kubernetes cluster
- Kubernetes cluster
- Helm
```
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

---
## Setting it up on the cluster
- Clone the repo
```
git clone git@github.com:nazim-deriv/vault-sessions.git
cd vault-sessions/slides/session-1/
```
- Since we will be using a helm based approach, it just takes two commands to install vault
```
helm repo add hashicorp https://helm.releases.hashicorp.com
helm upgrade --install vault hashicorp/vault --values values.yaml
```
- Login to the vault's pod & initialize the vault
```
kubectl exec -it pod/vault-0 -- sh
vault operator init
```

---
- If you wish to access the vault UI, there is a yaml file called `vault-ui-traefik.yaml` which will create an ingress traefik rule to make UI accessible on port `80` over `localhost` [Note that this setup assumes you already have traefik deployed on the cluster. If its not present, then deploy it before creating the rule]
```
kubectl apply -f vault-ui-traefik.yaml
```
- Add following entry in your `/etc/hosts` file
```
127.0.0.1 test-vault.com
```
- Access the vault UI by visiting: http://test-vault.com/ on your browser

---
## Vault unsealing / sealing
- When a Vault server is started, it starts in a sealed state. In this state, Vault is configured to know where and how to access the physical storage, but doesn't know how to decrypt any of it.
- Unsealing is the process of obtaining the plaintext root key necessary to read the decryption key to decrypt the data, allowing access to the Vault.

---
Following command is used to unseal the vault. Do note that the threshold to unseal the vault is `3` keys
```
vault operator unseal <key_1>
vault operator unseal <key_2>
vault operator unseal <key_3>
```

Once a Vault node is unsealed, it remains unsealed until one of these things happens:
- It is resealed via the API.
- The server is restarted.
- Vault's storage layer encounters an unrecoverable error.

There is also an API to seal the Vault. This will throw away the root key in memory and require another unseal process to restore it.

---
## Storing secrets in vault
- Key/Value secrets engine is a generic key-value store used to store arbitrary secrets. Secrets written to Vault are encrypted and then written to backend storage.
- Let's login to the vault & create a database secret to store a username and password at the path internal/database/config. In order to create this secret, it needs key/value secret engine in vault to be enabled.
```
vault login
vault secrets enable -path=internal kv-v2
```
- Create a key/value secret using following command:
```
vault kv put internal/database/config username="postgres" password="hshsdhtwrabzfdb4wgwge"
vault kv get internal/database/config
```

---
## Deploying application and injecting secret from vault
- Vault provides a Kubernetes authentication method that enables clients to authenticate with a Kubernetes Service Account Token. Enable the kubernetes authentication method in vault and configure it to use the location of the Kubernetes API
```
vault auth enable kubernetes
vault write auth/kubernetes/config kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443"
```

---
- For a client to read the secret data defined at `internal/database/config`, requires that the read capability be granted for the path `internal/data/database/config`.
```
vault policy write internal-app - <<EOF
path "internal/data/database/config" {
   capabilities = ["read"]
}
EOF
```

---
- Create a Kubernetes authentication role named `internal-app`. The role connects the Kubernetes service account, internal-app, and namespace, default, with the Vault policy, `internal-app`. The tokens returned after authentication are valid for 24 hours.
```
vault write auth/kubernetes/role/internal-app \
      bound_service_account_names=internal-app \
      bound_service_account_namespaces=default \
      policies=internal-app \
      ttl=24h
```

---
The role created above defined a cluster's service account `internal-app` which needs to be created
```
kubectl create sa internal-app
```
Let's deploy an application and inject the above created KV secret into the container of the pod
```
kubectl apply -f deployment-orgchart.yaml
```
The Vault-Agent injector looks for deployments that define specific annotations. None of these annotations exist in the current deployment. This means that no secrets are present on the orgchart container in the orgchart pod.
```
kubectl exec -it $(kubectl get pod -l app=orgchart -o jsonpath="{.items[0].metadata.name}") --container orgchart -- ls /vault/secrets
```

---
The Vault Agent Injector only modifies a deployment if it contains a specific set of annotations.
```
kubectl patch deploy orgchart --patch "$(cat patch-injector.yaml)"
kubectl exec -it $(kubectl get pod -l app=orgchart -o jsonpath="{.items[0].metadata.name}") --container orgchart -- cat /vault/secrets/database-config.txt
```
The structure of the injected secrets may need to be structured in a way for an application to use. Before writing the secrets to the file system a template can structure the data.
```
kubectl patch deployment orgchart --patch "$(cat patch-inject-secrets-as-template.yaml)"
kubectl exec -it $(kubectl get pod -l app=orgchart -o jsonpath="{.items[0].metadata.name}") --container orgchart -- cat /vault/secrets/database-config.txt
```

---
## Useful commands
- Initialize the vault
```
vault operator init
```
- Unseal the vault
```
vault operator unseal <key>
```
- Login to the vault
```
vault login
```
- Enable the database secrets engine
```
vault secrets enable database
```

---
- Create or update the key in the mount with the value
```
vault kv put -mount=<mount_name> <key> <value>
```
- Read the key value
```
vault kv get -mount=<mount_name> <key>
```
- Delete the secret
```
vault kv delete -mount=<mount_name> <key>
```
- Recover the deleted data if the deletion was unintentional and the `destroyed` was `false`
```
vault kv undelete -mount=<mount_name> <key>
```

---
## References
Following are few of the referenced links:

https://developer.hashicorp.com/vault/docs/concepts/seal
https://developer.hashicorp.com/vault/tutorials/getting-started/getting-started-first-secret
https://developer.hashicorp.com/vault/tutorials/getting-started/getting-started-dynamic-secrets
https://developer.hashicorp.com/vault/tutorials/kubernetes/kubernetes-sidecar
https://developer.hashicorp.com/vault/docs/secrets/kv/kv-v2
https://developer.hashicorp.com/vault/docs/secrets/databases/postgresql
https://developer.hashicorp.com/vault/api-docs/secret

