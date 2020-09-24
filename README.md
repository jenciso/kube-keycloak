# KUBE- KEYCLOAK

## Intro

Steps to configure your kubernetes cluster using openID authentication and keycloak as your Identity Manager

## Steps


Create a REALM using the json template file [k8s-realm-example.json](./k8s-realm-example.json)

![](https://i.imgur.com/RHhAnxC.png)

Regenerate the secret for clientID of OidKube client

![](https://i.imgur.com/rv1S385.png)

Setup a Ldap User Federation

![](https://i.imgur.com/stlRm82.png)

![](https://i.imgur.com/u33Pkw4.png)

![](https://i.imgur.com/70bnJ2e.png)

To test this setup, try to "Synchronize all users"






