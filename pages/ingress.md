# Ingress to your application

If you would like to expose your application to outside the cluser, you will need to create an ingress and service resource and tie it to a domain name.

There are two parts that need to happen.

1. You will need to generate a certificate
2. You will need to create a ingress resource pointing to your app service.

## Update your DNS to point to guldentech infra node


## Cert

!> Be sure to update your DNS to point to guldentech infra IP addresses otherwise your cert will not be generated.

Below is an example:
```yaml```

## Ingress

Below is an example:
```yaml```

Example domain provider needed configuration for non guldentech.com domains.

![dns](../_media/dns.png)