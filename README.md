# Bootstrapping HRZN K8s

### Components
- Helm
- cert-manager
- Keycloak
- Concourse

### Environment
- [Install Helm](https://helm.sh/docs/using_helm/#installing-helm)
```
$ helm repo add jetstack https://charts.jetstack.io
"jetstack" has been added to your repositories
$ helm repo update
Hang tight while we grab the latest from your chart repositories...
...Skip local chart repository
...Successfully got an update from the "incubator" chart repository
...Successfully got an update from the "stable" chart repository
Update Complete. ⎈ Happy Helming!⎈
```

### Execution
- [Install Tiller](https://helm.sh/docs/using_helm/#installing-tiller)
```
$ kubectl create -f rbac-config.yaml
serviceaccount "tiller" created
clusterrolebinding "tiller" created
$ helm init --service-account tiller --history-max 200
```

- [Install cert-manager](https://docs.cert-manager.io/en/latest/getting-started/install.html)
```
$ kubectl apply -f https://raw.githubusercontent.com/jetstack/cert-manager/release-0.7/deploy/manifests/00-crds.yaml
customresourcedefinition.apiextensions.k8s.io/certificates.certmanager.k8s.io created
customresourcedefinition.apiextensions.k8s.io/challenges.certmanager.k8s.io created
customresourcedefinition.apiextensions.k8s.io/clusterissuers.certmanager.k8s.io created
customresourcedefinition.apiextensions.k8s.io/issuers.certmanager.k8s.io created
customresourcedefinition.apiextensions.k8s.io/orders.certmanager.k8s.io created
$ kubectl create namespace cert-manager
namespace/cert-manager created
$ kubectl label namespace cert-manager certmanager.k8s.io/disable-validation=true
namespace/cert-manager labeled
$ helm install \
  --name cert-manager \
  --namespace cert-manager \
  --version v0.7.0 \
  jetstack/cert-manager
NAME: cert-manager
NAMESPACE: cert-manager
STATUS: DEPLOYED
```

- Verify cert-manager installation
```
$ kubectl get pods --namespace cert-manager
NAME                        READY   STATUS    RESTARTS
cert-manager                1/1     Running   0
cert-manager-cainjector     1/1     Running   0
cert-manager-webhook        1/1     Running   0
```

- Create Cloudflare secret
```
$ kubectl create -f cloudflare.yaml
secret/cloudflare-api-key-secret created
```

- Set up ACME ClusterIssuer
```
$ kubectl create -f hrzn-acme.yaml
clusterissuer.certmanager.k8s.io/hrzn-acme-prod created
```

- Verify the account has registered successfully
```
$ kubectl describe clusterissuer hrzn-acme
...
Status:
  Acme:
    Uri:  https://acme-v02.api.letsencrypt.org/acme/acct/54086923
  Conditions:
    Message:               The ACME account was registered with the ACME server
    Reason:                ACMEAccountRegistered
    Status:                True
    Type:                  Ready
Events:                    <none>
```

- Issue the certificate
```
$ kubectl create -f hrzn-studio.yaml
certificate.certmanager.k8s.io/hrzn-studio created
```

- Verify the certificate has issued successfully
```
$ kubectl describe certificate hrzn-studio
...
Status:
  Conditions:
    Message:               Certificate is up to date and has not expired
    Reason:                Ready
    Status:                True
    Type:                  Ready
Events:
  Type    Reason         From          Message
  ----    ------         ----          -------
  Normal  OrderCreated   cert-manager  Created Order resource "hrzn-studio"
  Normal  OrderComplete  cert-manager  Order "hrzn-studio" completed successfully
  Normal  CertIssued     cert-manager  Certificate issued successfully
```

- [Install ingress-nginx](https://kubernetes.github.io/ingress-nginx/deploy/#using-helm)
```
$ helm install stable/nginx-ingress --name edge --set rbac.create=true
NAME: edge
NAMESPACE: default
STATUS: DEPLOYED
```

- [Install Keycloak](https://github.com/helm/charts/tree/master/stable/keycloak)
```
$ helm install --name keycloak stable/keycloak -f keycloak.yaml
NAME: keycloak
NAMESPACE: default
STATUS: DEPLOYED
```

- Retrieve initial Keycloak user password
```
$ kubectl get secret --namespace default keycloak-http -o jsonpath="{.data.password}" | base64 --decode; echo
secret
```

- Configure Keycloak
```
log in, switch theme, configure smtp
sanitize html display name
enable forgot password and verify email

add concourse realm, switch theme, configure smtp
set (html) display name
enable forgot password, remember me, and verify email

create concourse client, set display name
root url: https://concourse.hrzn.studio
access type: confidential
dag: disabled
valid redirect uri: /sky/issuer/callback
base url: /
admin url:
web origins: *

add user role to concourse client
add username builtin mapper
create groups mapper (groups membership, not full path, add to id/userinfo)

create initial user in concourse realm
create main group for concourse admins
map group to user role from concourse client
add initial user as member of group
```

- Create Concourse secrets
```
$ mkdir -p concourse-secrets && cd concourse-secrets
$ docker run --rm -v$(pwd):$(pwd) -w $(pwd) -it ubuntu:latest bash
root@abcdef012345678:/path# apt-get update && apt-get install -y openssh-client
...
root@abcdef012345678:/path# ssh-keygen -t rsa -f host-key  -N ''
mv host-key.pub host-key-pub
ssh-keygen -t rsa -f worker-key  -N ''
mv worker-key.pub worker-key-pub
ssh-keygen -t rsa -f session-signing-key  -N ''
rm session-signing-key.pub

...
root@abcdef012345678:/path# exit
exit
$ ls
host-key  host-key-pub  session-signing-key  worker-key  worker-key-pub
```

- Retrieve client secret from Keycloak concourse client credentials screen
```
$ echo -n 12345678-abcd-1234-abcd-1234567890ab > oidc-client-secret
$ echo -n concourse > oidc-client-id
$ ls
host-key  host-key-pub  oidc-client-id  oidc-client-secret  session-signing-key  worker-key  worker-key-pub
```

- Send secrets to Kubernetes
```
$ kubectl create secret generic concourse-concourse --from-file=.
secret/concourse-concourse created
$ cd ..
```

- [Install Concourse](https://github.com/helm/charts/tree/master/stable/concourse)
```
$ helm install --name concourse stable/concourse -f concourse.yaml
NAME: concourse
NAMESPACE: default
STATUS: DEPLOYED
```

- Test Concourse logins
