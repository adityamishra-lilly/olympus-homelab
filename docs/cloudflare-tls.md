# Cloudflare and TLS

How traffic gets from a browser to a pod on this cluster, which certificates are involved,
and why they're set up the way they are. Written down because it's the kind of thing I'll
have forgotten the next time a cert expires.

## The two TLS legs

With a DNS record proxied through Cloudflare (orange cloud), a request crosses two separate
TLS connections:

```
browser ──TLS #1──▶ Cloudflare edge ──TLS #2──▶ home router :443 ──▶ Traefik ──▶ Service ──▶ Pod
          edge cert                    origin cert
```

- **Leg 1, browser to Cloudflare.** Cloudflare's own edge certificate (Universal SSL).
  Automatic, publicly trusted, nothing to manage.
- **Leg 2, Cloudflare to origin.** This is the cert stored in the cluster. Traefik terminates
  TLS with it, and traffic inside the cluster is plain HTTP to the pod.

## SSL/TLS encryption modes

Set under SSL/TLS → Overview. It controls how Cloudflare treats leg 2.

| Mode              | Leg 2                                | Verdict                                       |
| ----------------- | ------------------------------------ | --------------------------------------------- |
| Off / Flexible    | Plain HTTP                           | Never. The padlock in the browser is a lie.   |
| Full              | TLS, any cert accepted (self-signed) | Encrypted but unverified, so it can be MITM'd |
| **Full (strict)** | TLS, cert must be valid for the host | **Use this.** Origin CA or a public CA cert   |

Also turn on **Always Use HTTPS** so `http://` is redirected at the edge.

## Options for the origin certificate

### 1. Cloudflare Origin CA certificate (current)

Issued from SSL/TLS → Origin Server. Signed by Cloudflare's private CA, so **only Cloudflare
trusts it**. A browser hitting the origin directly gets a certificate warning, which is fine
because nothing should hit it directly.

- Validity is selectable up to 15 years. Long validity means no renewal work, but a leaked key
  stays useful for that long. I pick **1–2 years** and note the expiry in `current-issues.md`.
  Certs can be revoked from the same dashboard page.
- Key type: **ECDSA**. It's smaller and faster than RSA, and Cloudflare is the only client,
  so compatibility isn't a concern.
- Hostnames: **`am-labs.dev` and `*.am-labs.dev`**. One cert covers the apex and every
  first-level subdomain (`ld`, `grafana`, …). A wildcard does not cover two levels
  (`a.b.am-labs.dev`), and it does not cover the apex on its own, which is why both are listed.
- Only works while the record is proxied. Turn the orange cloud off and every browser rejects it.

### 2. cert-manager with Let's Encrypt (DNS-01 via the Cloudflare API)

The GitOps-native answer, and where this is heading. cert-manager runs in the cluster,
proves domain ownership by writing a TXT record through a scoped Cloudflare API token, and
renews 90-day certs on its own. Git holds only a `ClusterIssuer`, a `Certificate` per host and
the API token (SOPS-encrypted). No PEM files.

- Publicly trusted, so it still works if a record is ever un-proxied.
- DNS-01 rather than HTTP-01: it works for wildcards and doesn't need port 80 reachable.
- API token scope: `Zone:DNS:Edit` on `am-labs.dev` only.
- Cloudflare also publishes `origin-ca-issuer`, a cert-manager issuer for Origin CA certs.
  That automates option 1, but I'd rather have certs that aren't tied to the proxy.

### 3. Cloudflare Tunnel (cloudflared)

An outbound-only connection from the cluster to Cloudflare. No port forward, the home IP is
never published, and no origin cert is needed because the tunnel itself is encrypted and
authenticated. It's how linkding was first exposed (`apps/staging/linkding/cloudflare.yaml`,
currently disabled). Cost: another moving part to run, and all traffic depends on it.

## Locking down the origin

An Origin CA cert doesn't stop anyone reaching the origin. If someone finds the home IP,
they can talk to Traefik directly and skip Cloudflare's WAF, rate limiting and DDoS
protection. Two layers close that:

1. **Authenticated Origin Pulls (mTLS).** SSL/TLS → Origin Server → Authenticated Origin Pulls.
   Cloudflare then presents a client certificate on leg 2. Traefik is configured with a
   `TLSOption` (`clientAuth.clientAuthType: RequireAndVerifyClientCert`, CA =
   Cloudflare's origin-pull CA) and rejects any connection without it.
2. **Firewall.** Only allow 443 in from Cloudflare's published ranges
   (https://www.cloudflare.com/ips/). The list changes occasionally, so check it now and then.

With a Tunnel, neither is needed because there's no open port.

## Sharing one cert across namespaces

Secrets are namespaced and Ingresses can only reference secrets in their own namespace. Two
ways to reuse a wildcard cert:

- **Copy the SOPS secret into each app's overlay** (`apps/staging/<app>/<app>-tls-secret.yaml`).
  More files, but it's plain Kubernetes. **This is what I do**, to keep Ingresses
  controller-agnostic.
- **Traefik default certificate** via a `TLSStore` named `default`. Set it once and Ingresses
  don't need a `tls.secretName`. Less duplication, but Traefik-specific.

## Creating and committing an origin cert

```sh
# 1. Dashboard: SSL/TLS → Origin Server → Create Certificate
#    ECDSA, am-labs.dev + *.am-labs.dev, 1–2 years. Save cert.pem and key.pem OUTSIDE the repo.

# 2. Build the Secret manifest
kubectl create secret tls site-tls-secret -n site \
  --cert=cert.pem --key=key.pem --dry-run=client -o yaml \
  > apps/staging/site/site-tls-secret.yaml

# 3. Encrypt in place (rule in apps/staging/.sops.yaml encrypts data/stringData only)
sops --encrypt --in-place apps/staging/site/site-tls-secret.yaml

# 4. Delete the plaintext PEMs
```

Check what a committed secret contains (needs the age private key locally):

```sh
sops -d apps/staging/site/site-tls-secret.yaml | yq '.data."tls.crt"' | base64 -d \
  | openssl x509 -noout -subject -issuer -dates -ext subjectAltName
```

## DNS for the apex site

- `A am-labs.dev → <home public IP>`, proxied.
- `CNAME www → am-labs.dev`, proxied.
- Redirect Rule: `www.am-labs.dev/*` → `https://am-labs.dev/${1}`, 301.

Apex CNAMEs are fine on Cloudflare (CNAME flattening) if the target ever becomes a hostname.

## Caching

nginx in the site image already sends the right headers. Cloudflare respects them.

- `/_astro/*` → `public, max-age=31536000, immutable` (fingerprinted, safe to cache forever).
- HTML → `public, max-age=0, must-revalidate` (new deploys show up immediately).

`cf-cache-status` in the response headers shows whether the edge served it.

## Checking it works

```sh
# Origin, bypassing Cloudflare (-k because Origin CA isn't publicly trusted)
curl -k --resolve am-labs.dev:443:<node-ip> -I https://am-labs.dev/

# Through Cloudflare
curl -I https://am-labs.dev/          # expect 200, server: cloudflare, cf-ray
curl -I https://www.am-labs.dev/      # expect 301 to the apex

# With Authenticated Origin Pulls on, the direct curl above should now fail the handshake
```

## Decisions and roadmap

| Decision             | Choice now                      | Later                                     |
| -------------------- | ------------------------------- | ----------------------------------------- |
| SSL mode             | Full (strict)                   | —                                         |
| Origin cert          | Wildcard Origin CA, ECDSA, 1–2y | cert-manager + Let's Encrypt DNS-01       |
| Cert distribution    | SOPS secret per namespace       | cert-manager `Certificate` per app        |
| Origin exposure      | Port forward 443                | AOP + Cloudflare-only firewall, or Tunnel |
| Cert expiry tracking | Note in `current-issues.md`     | cert-manager renews; alert on failure     |
