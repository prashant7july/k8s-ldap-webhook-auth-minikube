# k8s-ldap-webhook-auth-minikube
A Minikube demo showing LDAP-backed webhook token authentication for the Kubernetes API server using OpenLDAP and a Node.js webhook service

> **📝 Note**: If you encounter authentication issues, please refer to [TROUBLESHOOTING.md](./TROUBLESHOOTING.md) for detailed debugging steps and solutions.

## Implementing a custom Kubernetes authentication method

### Create new user entry for LDAP
```sh
cd ldap-bitnami

docker compose up -d

docker ps

ldapadd -x -H ldap://localhost:1389 \
  -D "cn=admin,dc=example,dc=org" \
  -w adminpassword \
  -f alice.ldif

ldapsearch -x -H ldap://localhost:1389 \
  -D "cn=admin,dc=example,dc=org" \
  -w adminpassword \
  -b "dc=example,dc=org" "(cn=alice)"

ldapsearch -LLL -H ldap://localhost:1389 \
           -x \
           -D "cn=admin,dc=example,dc=org" -w adminpassword \
           -b "dc=example,dc=org" \
           '(cn=alice)'

ldapsearch -LLL -H ldap://localhost:1389 \
           -x \
           -D "cn=admin,dc=example,dc=org" -w adminpassword \
           -b "dc=example,dc=org" \
           '(&(objectClass=inetOrgPerson)(cn=alice)(userPassword=alicepassword))'
```

### Run Node Project
```sh
cd auth-services

docker compose up -d
```

Check `http://localhost:4040` to get new tunneling HTTPS url and use in Kubectl config file in server section in yaml

### Kubectl config file using token authn `webhook-config.yaml`

```sh
minikube start
minikube ssh
sudo -i
cat <<EOF > /var/lib/minikube/certs/webhook.yaml
apiVersion: v1
kind: Config
clusters:
  - name: webhook-authn
    cluster:
      insecure-skip-tls-verify: true
      server: http://host.docker.internal:7443/auth
users:
  - name: webhook-user
contexts:
  - name: webhook-context
    context:
      cluster: webhook-authn
      user: webhook-user
current-context: webhook-context
EOF
exit
exit
```

**Note**: For local development, use `http://host.docker.internal:7443/auth` as the server URL. For production, replace with your ngrok URL from `http://localhost:4040`.

#### Check from inside the Minikube VM
```sh
minikube ssh
sudo cat /var/lib/minikube/certs/webhook.yaml

OR

sudo less /var/lib/minikube/certs/webhook.yaml
```

### [Enable webhook token authn on api-server](https://evalle.github.io/blog/20190521-configure-kube-apiserver-in-minikube.html)

#### Stop Minikube (required to change API server flags)

```bash
minikube stop
```

#### 🚀 4. Start Minikube with webhook config flag

```bash
minikube start \
  --extra-config=apiserver.authentication-token-webhook-config-file=/var/lib/minikube/certs/webhook.yaml \
  --extra-config=apiserver.authentication-token-webhook-version=v1
```

**Important**: The `authentication-token-webhook-version=v1` flag is required to ensure compatibility with the TokenReview API version.

---

### To verify the API server is using the file

```bash
minikube ssh
sudo -i
vi /etc/kubernetes/manifests/kube-apiserver.yaml
cat /etc/kubernetes/manifests/kube-apiserver.yaml
```

Expected output:

```yaml
--authentication-token-webhook-config-file=/var/lib/minikube/certs/webhook.yaml
```

### Testing the LDAP authentication method

1. Use kubectl cli k8s client and point to the right configfile using --kubeconfig flag,
2. Make a request using bearer token,
3. API-Server receives a request and triggers a Webhook Token,

`kubectl --kubeconfig admin.conf config set-credentials alice --token alice:alicepassword`

`kubectl --kubeconfig admin.conf config view` or `kubectl config view`

apiVersion: v1
clusters: null
contexts: null
current-context: ""
kind: Config
preferences: {}
users:
- name: alice
  user:
    token: REDACTED

`kubectl config view --minify --flatten > admin.conf`

kubectl config view --kubeconfig admin.conf | grep server

kubectl --kubeconfig admin.conf config set-credentials alice --token=alice:alicepassword

kubectl config view --kubeconfig admin.conf | grep token

kubectl --kubeconfig admin.conf config set-context alice-context --cluster=minikube --user=alice

kubectl --kubeconfig admin.conf config use-context alice-context

kubectl config view --kubeconfig admin.conf --raw

kubectl --kubeconfig admin.conf config get-contexts

kubectl --kubeconfig admin.conf config use-context minikube

kubectl --kubeconfig admin.conf get pods --all-namespaces
