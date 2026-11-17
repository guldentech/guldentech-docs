# Hostnames and domains

To put your app on the internet, you need a hostname (like `myapp.guldentech.com` or `www.mysite.com`) that points at
your Service. **We publish hostnames for you**: you don't create an Ingress, and you can't. Every hostname on the
platform is listed in one place, the `guldentech-k8s` repository, so nobody can claim an address that belongs to
someone else.

## Request a hostname

Email [guldentechjobs@gmail.com](mailto:guldentechjobs@gmail.com) with:

- the hostname you want (and `www.` too, if it's your own domain)
- your namespace
- the **Service name and port** to send the traffic to (a Service of type `ClusterIP`, see
  [Getting started](/pages/getting-started))

We add it, the change is reviewed and merged, and a minute or two later `https://<hostname>` works. To change or remove
a hostname, ask the same way.

?> Make sure the Service exists and has running pods first. Until it does, visitors get a Cloudflare or 502 error
page.

## A `guldentech.com` subdomain

Nothing else to do: `*.guldentech.com` is already ours. Tell us the name you want, for example `myapp.guldentech.com`.

## Your own domain

Your domain has to move into GuldenTech's Cloudflare account, which means switching its nameservers (not adding an A or
CNAME record). Cloudflare can only route a domain through the tunnel when its DNS is hosted there.

1. Email us your domain and a copy of all its current DNS records. Include email records (MX, SPF/TXT, DKIM) so mail
   keeps working.
2. We add the domain to Cloudflare, copy your records, and send you two nameservers, like `dale.ns.cloudflare.com` and
   `dina.ns.cloudflare.com`.
3. At your registrar (Porkbun, GoDaddy, Namecheap, ...), replace the domain's nameservers with those two.
4. The switch usually takes minutes, sometimes a few hours. Your site keeps working from its old setup until then.
5. Once the nameservers have switched, we publish the hostname (see above).

After the switch, GuldenTech manages your domain's DNS records. Ask us when you need one added or changed.

!> A domain that isn't on GuldenTech's Cloudflare can't be published. Don't ask us to publish one before step 4 is done.

## What Cloudflare handles

- **HTTPS.** Cloudflare serves the certificate. There's nothing to configure in your app, and no certificate secret to
  create or renew. Plain `http://` requests are redirected to `https://`.
- **Caching.** Cloudflare caches static files like images, CSS and JS, so after replacing a file under the same name,
  visitors may see the old one for a few hours.
- **Your app sees plain HTTP.** The request reaches your pod over the tunnel as normal HTTP, with the visitor's
  original scheme in the `X-Forwarded-Proto` header. If your framework redirects to HTTPS itself, make it trust that
  header, or it can loop.

!> Uploads are limited to 100 MB per request, a Cloudflare free-plan limit. Split large uploads into chunks.

## Staying out of search engines

If a site shouldn't show up on Google (a staging copy, for example), tell us. We add a rule that marks it `noindex`.

## Don't apply your own Ingress

Older guides had you apply an Ingress with cert-manager and Traefik annotations. That no longer works:

- You don't have permission to create Ingresses, so `kubectl apply` of one is refused.
- The cluster no longer issues certificates for sites, and Traefik no longer serves them.
- If your GitHub Actions workflow or `manifests/` folder still applies an Ingress, remove it. The deploy will fail on it,
  even though the rest of it goes through.
