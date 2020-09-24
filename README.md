# KUBE- KEYCLOAK

## Intro

Steps to configure your kubernetes cluster using openID authentication and keycloak as your Identity Manager

## KEYCLOAK - Steps 


Create a REALM using the json template file [k8s-realm-example.json](./k8s-realm-example.json)

![](https://i.imgur.com/RHhAnxC.png)

Regenerate the secret for clientID of OidKube client

![](https://i.imgur.com/rv1S385.png)

Setup a Ldap User Federation

![](https://i.imgur.com/stlRm82.png)

![](https://i.imgur.com/u33Pkw4.png)

![](https://i.imgur.com/70bnJ2e.png)

To test this setup, try to "Synchronize all users" and check it out oin Manage -> Users

![](https://i.imgur.com/0btzsoC.png)

Create a LDAP Mappers

![](https://i.imgur.com/bH89qe3.png)

![](https://i.imgur.com/TTU1uuO.png)

![](https://i.imgur.com/cD6IaiU.png)

Verifying the groups

![](https://i.imgur.com/MiQr390.png)

Turn off the full group path

![](https://i.imgur.com/r9MVRLH.png)


Verify in the oidckube client the valid URis

![](https://i.imgur.com/65pagay.png)


## Kubernetes - Steps


Modify your kube-apiserver config (Ex. `/usr/lib/systemd/system/kube-apiserver.service`). Add these lines:

```
   --oidc-issuer-url=https://sso-lab.docs-planet.site/auth/realms/k8s \
   --oidc-client-id=oidckube \
   --oidc-username-claim=email \
   --oidc-groups-prefix=oidc: \
   --oidc-groups-claim=groups \
```

Restart 

```
systemctl daemon-reload
systemctl restart kube-apiserver.service
```

## Verify your token

```
KEYCLOAK_HOST=sso-lab.docs-planet.site
KEYCLOAK_REALM=k8s
KEYCLOAK_CLIENT_ID=oidckube
KEYCLOAK_CLIENT_SECRET=d0af03b5-92be-43a8-9ece-ef051d7e9832
KEYCLOAK_USER=juan
KEYCLOAK_PASS=SuperSecret2012
```

```shell
TOKEN=$(curl -s -X POST -H "Content-Type: application/x-www-form-urlencoded" \
https://$KEYCLOAK_HOST/auth/realms/$KEYCLOAK_REALM/protocol/openid-connect/token \
-d "client_id=$KEYCLOAK_CLIENT_ID\
&client_secret=$KEYCLOAK_CLIENT_SECRET\
&username=$KEYCLOAK_USER\
&password=$KEYCLOAK_PASS\
&scope=openid\
&grant_type=password\
&response_type=id_token"\
| jq -r '.id_token')
```

Check it in https://jwt.io/

```shell
FULL_TOKEN=$(curl -s -X POST -H "Content-Type: application/x-www-form-urlencoded" \
https://$KEYCLOAK_HOST/auth/realms/$KEYCLOAK_REALM/protocol/openid-connect/token \
-d "client_id=$KEYCLOAK_CLIENT_ID\
&client_secret=$KEYCLOAK_CLIENT_SECRET\
&username=$KEYCLOAK_USER\
&password=$KEYCLOAK_PASS\
&scope=openid\
&grant_type=password\
&response_type=id_token")
```

```shell
REFRESH_TOKEN=$(echo $FULL_TOKEN | jq -r '.refresh_token')
ID_TOKEN=$(echo $FULL_TOKEN | jq -r '.id_token')
```


## Testing

Verify your logs in k8s-audit.log file

```
tail -f /var/log/kubernetes/k8s-audit.log  | grep oidc
```

Setting my user credential

```
kubectl config set-cluster lab --server=https://apik8s-tst.docs-planet.site --insecure-skip-tls-verify
kubectl config set-context k8s-lab --cluster=lab --user=user-lab
kubectl config use-context k8s-lab
```

```
kubectl config set-credentials user-lab \
   --auth-provider=oidc \
   --auth-provider-arg=idp-issuer-url=https://$KEYCLOAK_HOST/auth/realms/$KEYCLOAK_REALM \
   --auth-provider-arg=client-id=$KEYCLOAK_CLIENT_ID\
   --auth-provider-arg=client-secret=$KEYCLOAK_CLIENT_SECRET \
   --auth-provider-arg=refresh-token=$REFRESH_TOKEN \
   --auth-provider-arg=id-token=$ID_TOKEN
```

Delete tokens and test kubelogin

## Setup kubelogin

```
wget https://github.com/int128/kubelogin/releases/download/v1.15.1/kubelogin_linux_amd64.zip
unzip kubelogin_linux_amd64.zip
chmod 755 kubelogin && mv kubelogin ~/bin

kubelogin version
```

Give permissions to group "oidc:kubernetes-admins"

Execute this command in master node
```
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: oidc-kubernetes-admins
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: oidc:kubernetes-admins
EOF
```

Using another way to setup user in kubectl

```
users:
- name: oidc
  user:
    exec:
      apiVersion: client.authentication.k8s.io/v1beta1
      command: kubectl
      args:
      - oidc-login
      - get-token
      - --oidc-issuer-url=https://sso-lab.docs-planet.site/auth/realms/k8s
      - --oidc-client-id=oidckube
      - --oidc-client-secret=bccd809d-3966-4e01-b587-8ae485dd4914
```

Classic way

Configure the kubeconfig like:
```
users:
- name: user-lab
  user:
    auth-provider:
      config:
        client-id: oidckube
        client-secret: bccd809d-3966-4e01-b587-8ae485dd4914
        idp-issuer-url: https://sso-lab.docs-planet.site/auth/realms/k8s
      name: oidc
```

```
kubelogin
```

### Setup OTP

https://keycloak.enciso.website/auth/realms/k8s/account/

