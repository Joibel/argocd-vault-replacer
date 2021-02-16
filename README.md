# vault-replacer

Mainly intended as a plugin for [Argo CD](https://argoproj.github.io/argo-cd/)

Will scan the current directory recursively for any .yaml (or .yml if you're so inclined) files and attempt to replaces strings of the form \<vault:/store/data/path!key\> with those obtained from a vault kv2 store. It is intended that this is run from within your argocd-server pod as a plugin.

## Authentication

It only has two methods of authenticating with vault:
* Using kubernetes authentication method https://github.com/hashicorp/vault/blob/master/website/content/docs/auth/kubernetes.mdx
* Using a token, which is only intended for debugging

Both methods expect the environment variable VAULT_ADDR to be set.

It will attempt to use kubernetes authentication through an appropriate service account first, and complain if that doesn't work. It will then use VAULT_TOKEN which should be a valid token. This tool has no way of renewing a token or obtaining one other than through a kubernetes service account.

To use the kubernetes service account your pod should be running with the appropriate service account, and will try to obtain the JWT token from /var/run/secrets/kubernetes.io/serviceaccount/token which is the default location.

It will use the environment variable VAULT_ROLE as the name of the role for that token, defaulting to "argocd".
It will use the environment variable VAULT_AUTH_PATH to determine the authorisation path for kubernetes authentication. This defaults in this tool and in vault to "kubernetes" so will probably not need configuring.

## Valid vault paths

Currently the only valid 'URL style' to a path is

\<vault:/store/data/path!key|modifier|modifier\>

You must put ../data/.. into the path. If your path or key contains !, <, > or | you must URL escape it. If your path or key has one or more leading or trailing spaces or tabs you must URL escape them you weirdo.

## Modifiers

You can modify the resulting output with the following modifiers:

* base64: Will base64 encode the secret. Use for data: sections in kubernetes secrets.


# Installing as an Argo CD Plugin
You can use [our Kustomization example](https://github.com/Joibel/vault-replacer/tree/main/examples/kustomize/argocd) to install Argo CD and to bootstrap the installation of the plugin at the same time. However the steps below will detail what is required should you wish to do things more manually. The Vault authentication setup cannot be done with Kustomize and must be done manually.

## Vault Kubernetes Authentication
You will need to set up the Vault Kubernetes authentication method for your cluster.

You will need to create a service account. In this example, our service account will be called 'argocd'. Our example creates the serviceAccount in the argocd namespace:

```YAML
apiVersion: v1
kind: ServiceAccount
metadata:
  name: argocd
  namespace: argocd
```

You will need to tell Vault about this Service Account and what policy/policies it maps to:

```
vault write auth/kubernetes/role/argocd \
        bound_service_account_names=argocd \
        bound_service_account_namespaces=argocd \
        policies=argocd \
        ttl=1h
```
This is better documented by Hashicorp themselves, do please refer to [their documentation](https://www.vaultproject.io/docs/auth/kubernetes).

Lastly, you will need to modify the argocd-repo-server deployment to use your new serviceAccount, and to allow the serviceAccountToken to automount when the pod starts up. You must patch the deployment with:
```YAML
apiVersion: apps/v1
kind: Deployment
metadata:
  name: patch-serviceAccount
spec:
  template:
    spec:
      serviceAccount: argocd
      automountServiceAccountToken: true
```
## Plugin Installation
In order to install the plugin into Argo CD, you can either build your own Argo CD image with the plugin already inside, or make use of an Init Container to pull the binary. Argo CD's documentation provides further information how to do this: https://argoproj.github.io/argo-cd/operator-manual/custom_tools/

An init container manifest will look something like this:
```YAML
containers:
- name: argocd-repo-server
  volumeMounts:
  - name: custom-tools
    mountPath: /usr/local/bin/vault-replacer
    subPath: vault-replacer
  envFrom:
    - secretRef:
        name: argocd-vault-replacer-credentials
volumes:
- name: custom-tools
  emptyDir: {}
initContainers:
- name: vault-replacer-download
  image: alpine
  command: [sh, -c]
  args:
    - wget https://github.com/Joibel/vault-replacer/releases/download/0.2/vault-replacer-0.2-linux-amd64.tar.gz

      tar -xvzf vault-replacer-0.2-linux-amd64.tar.gz
    
      chmod +x vault-replacer
      
      mv vault-replacer /custom-tools/
  volumeMounts:
    - mountPath: /custom-tools
      name: custom-tools
```

## Plugin Configuration
After installing the plugin into the /custom-tools/ directory, you need to register it inside the Argo CD config. Declaratively, you can add this to your argocd-cm configmap file:

```YAML
configManagementPlugins: |-
  - name: vault-replacer
    generate:
      command: ["vault-replacer"]
```

This is documented further in Argo CD's documentation: https://argoproj.github.io/argo-cd/user-guide/config-management-plugins/

## Testing

Create a test yaml file that will be used to pull a secret from Vault. The below will look in vault for /path/to/your/secret and will return the key 'secretkey', it will then base64 encode that value. As we are using a Vault KV2 store, we must include ../data/.. in our path:

```YAML
apiVersion: v1
kind: Secret
metadata:
  name: vault-replacer-secret
data:
  sample-secret: <vault:path/data/to/your/secret!secretkey|base64>
type: Opaque

```
In this example, we pushed the above to `https://github.com/replace-me/vault-replacer-test/vault-replacer-secret.yaml`

We then deploy this as an argocd application, making sure we tell the application to use the vault-replacer plugin:

```YAML
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: vault-replacer-test
spec:
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: vault-replacer-test
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
  source:
    repoURL: 'https://github.com/replace-me'
    path: vault-replacer-test
    plugin:
      name: vault-replacer
    targetRevision: HEAD
```