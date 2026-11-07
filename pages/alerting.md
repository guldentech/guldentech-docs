# Alerting

## Platform status

[status.guldentech.com](https://status.guldentech.com/) shows the health of the GuldenTech platform: the Cloudflare
Tunnels, ingress (Traefik), TLS certificates, Kubernetes, storage, backups, Rancher, Harbor and the GitHub Actions
runners. It's checked every minute, runs on Cloudflare (so it stays up even if the cluster is down), and is where
incidents and planned maintenance are posted.

Check it first if your site, a deploy or an image pull suddenly stops working.

## Monitoring your own website

The status page checks the platform, not individual websites. To get alerted when your own site goes down, use a free
external monitor such as [UptimeRobot](https://uptimerobot.com), pointed at your site's public URL (and a health
endpoint if your app has one).

?> Questions or something looks wrong? Email [guldentechjobs@gmail.com](mailto:guldentechjobs@gmail.com).
