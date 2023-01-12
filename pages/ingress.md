# Ingress to your application

If you would like to expose your application to outside the cluser, you will need to create an ingress and service resource and tie it to a domain name.

There are two parts that need to happen.

1. You will need to generate a certificate
2. You will need to create a ingress resource pointing to your app service.

## Update your DNS to point to guldentech infra node

Example domain provider needed configuration for non guldentech.com domains.

![dns](../_media/dns.png)

## Cert

!> Be sure to update your DNS to point to guldentech infra IP addresses otherwise your cert will not be generated.

Apply the following resource to your namespace. This in return will create a secret with your TLS keys.

Below is an example:
```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: {your_site}.com
  namespace: {FILL_ME_IN}
spec:
  secretName: {your_site}.com
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  commonName: {your_site}.com
  dnsNames:
  - {your_site}.com
```

## Ingress

Apply the following resource to your namespace. This in return will expose your service to the outside world. It will force SSL for any connection.

Below is an example:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    ingress.kubernetes.io/ssl-redirect: "true" # This forces SSL
  name: {your_site}-ingress
  namespace: {FILL_ME_IN}
spec:
  rules:
  - host: {your_site}.com
    http:
      paths:
      - backend:
          service:
            name: {FILL_ME_IN}
            port:
              number: {FILL_ME_IN}
        path: /
        pathType: ImplementationSpecific
  tls:
  - hosts:
    - {your_site}.com
    secretName: {your_site}.com
```