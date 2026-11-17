# Hostnames and domains

To put your app on the internet, you need a hostname (like `myapp.guldentech.com` or `www.mysite.com`) that points at
your Service. **We publish hostnames for you**: you don't create an Ingress, and you can't. Every hostname on the
platform is listed in one place, the `guldentech-k8s` repository, so nobody can claim an address that belongs to
someone else.

## Publish a hostname

To put a hostname on your app, add it to your namespace's `config.yaml` in a pull request
(see [Making a change](/pages/namespaces?id=making-a-change-pull-request) for how to fork and open one):

```yaml
allowFrom: [cloudflare-tunnel-ingress]   # lets web traffic reach your pods
ingresses:
  myapp:
    spec:
      ingressClassName: cloudflare-tunnel
      rules:
      - host: myapp.guldentech.com
        http:
          paths:
          - { path: /, pathType: Prefix, backend: { service: { name: myapp, port: { number: 80 } } } }
```

Use your Service's name and port (a Service of type `ClusterIP`, see [Getting started](/pages/getting-started)). One
Ingress can list several hosts. When it's merged, the hostname starts working a minute or two later. To change or
remove a hostname, change or delete its block the same way.

Rules the pull request has to follow (a change that breaks them is rejected):
- a hostname can only belong to one namespace, and no wildcard hosts (`*.example.com`);
- every rule needs a `host`;
- no `tls:` section and no certificate settings: Cloudflare handles HTTPS.

?> Make sure the Service exists and has running pods first. Until it does, visitors get a Cloudflare or 502 error
page.

## A `guldentech.com` subdomain

Nothing else to set up: `*.guldentech.com` is already ours. Pick the name in your pull request, for example `myapp.guldentech.com`. Names that belong to the platform (`harbor`, `console`, `status`, ...) are taken.

## Your own domain

Your domain has to move into GuldenTech's Cloudflare account, which means switching its nameservers (not adding an A or
CNAME record). Cloudflare can only route a domain through the tunnel when its DNS is hosted there.

1. Email us ([guldentechjobs@gmail.com](mailto:guldentechjobs@gmail.com)) your domain and a copy of all its current DNS records. Include email
   records (MX, SPF/TXT, DKIM) so mail keeps working.
2. We add the domain to Cloudflare, copy your records, and send you two nameservers, like `dale.ns.cloudflare.com` and
   `dina.ns.cloudflare.com`.
3. At your registrar (Porkbun, GoDaddy, Namecheap, ...), replace the domain's nameservers with those two.
4. The switch usually takes minutes, sometimes a few hours. Your site keeps working from its old setup until then.
5. Once the nameservers have switched, publish the hostname with a pull request (see above).

After the switch, GuldenTech manages your domain's DNS records. Ask us when you need one added or changed.

!> A domain that isn't on GuldenTech's Cloudflare can't be published, and a pull request that lists one can't be merged.
Finish step 4 first.

## What Cloudflare handles

HTTPS (there's no certificate to create or renew, and `http://` is redirected), and caching of static files, so a
replaced file under the same name can look old for a few hours. Your app receives plain HTTP with the visitor's scheme
in the `X-Forwarded-Proto` header: if your framework redirects to HTTPS itself, make it trust that header or it can loop.

!> Uploads are limited to 100 MB per request (a Cloudflare free-plan limit). Split large uploads into chunks.

You don't apply an Ingress yourself any more, and a workflow or `manifests/` folder that still does will fail on it.
To keep a staging site out of search engines, say so in your pull request and we add a `noindex` rule.
