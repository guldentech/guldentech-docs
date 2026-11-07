# Ingress to your application

If you would like to expose your application to outside the cluser, you will need to create an ingress and service resource and tie it to a domain name.

## DNS

GuldenTech is moving to [Cloudflare Tunnels](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/)
exclusively. Visitors connect to Cloudflare, and Cloudflare forwards traffic to the cluster over a private tunnel.
There is no server IP to point your domain at anymore.

!> The old Miami server (`144.202.38.252`) is being retired. If your domain still points at it, reach out to
Ryan so we can move it over before it's shut off.

### A `guldentech.com` subdomain

Nothing to do. `*.guldentech.com` already points at the cluster, so create your Ingress (below) with a host like
`myapp.guldentech.com` and it works.

### Your own domain

Work with Ryan to set your domain up. It has to move into GuldenTech's Cloudflare account, which means switching
its nameservers, not adding an A or CNAME record. Cloudflare can only route a domain through the tunnel when its
DNS is hosted there.

1. Reach out to Ryan ([guldentechjobs@gmail.com](mailto:guldentechjobs@gmail.com)) with your domain and a copy of
   all its current DNS records. Include email records (MX, SPF/TXT, DKIM) so mail keeps working.
2. Ryan adds the domain to Cloudflare, copies your records, and sends you two nameservers, like
   `dale.ns.cloudflare.com` and `dina.ns.cloudflare.com`.
3. At your registrar (Porkbun, GoDaddy, Namecheap, ...), replace the domain's nameservers with those two.
4. The switch usually takes minutes, sometimes a few hours. Your site keeps working from its old setup until then.

After the switch, GuldenTech manages your domain's DNS records. Ask Ryan when you need one added or changed.

?> What Cloudflare handles for you: HTTP is redirected to HTTPS, and visitors get a Cloudflare certificate on top
of the cluster's Let's Encrypt one. Cloudflare also caches static files like images, CSS and JS, so after
replacing a file under the same name, visitors may see the old one for a few hours.

!> Uploads are limited to 100 MB per request, a Cloudflare free-plan limit. Split large uploads into chunks.

## Ingress

Apply the following resource to your namespace. This in return will expose your service to the outside world. It will force SSL for any connection.

Below is an example:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    ingress.kubernetes.io/ssl-redirect: "true"
    traefik.ingress.kubernetes.io/router.middlewares: default-redirect-https@kubernetescrd
    cert-manager.io/cluster-issuer: letsencrypt-prod
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

Also this will create a secret containing your TLS keys. The certificate is only issued once your domain's DNS
is on GuldenTech's Cloudflare (see the DNS section above), so finish that first or issuing will fail.