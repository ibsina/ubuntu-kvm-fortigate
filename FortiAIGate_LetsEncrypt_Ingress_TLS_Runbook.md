# FortiAIGate: Let's Encrypt Certificate for External HTTPS (Port 30443)

## Purpose

Configure FortiAIGate on Kubernetes to present a publicly trusted Let's Encrypt certificate to external LLM clients **without changing any TLS certificates used internally**. This runbook also explains how to preserve the configuration across Helm upgrades and reinstalls.

## Environment

| Setting | Value |
|---|---|
| Namespace | `fortiaigate` |
| Hostname | `faig.dell.cbcgslb.fortiseahk.com` |
| HTTPS endpoint | `https://faig.dell.cbcgslb.fortiseahk.com:30443` |
| Ingress class | `nginx-fortiaigate` |
| Ingress resource (observed) | `fortiaigate-ingress` |
| External TLS Secret | `faig-public-tls` |
| Internal TLS Secret | `fortiaigate-tls-secret` |
| Let's Encrypt challenge | Manual DNS-01 |
| Connectivity | Internet egress; no inbound internet access |

> **Validated state (October 29, 2026):** The Kubernetes Ingress references `faig-public-tls` for `faig.dell.cbcgslb.fortiseahk.com`, and an OpenSSL test to port `30443` returned `Verify return code: 0 (ok)` with TLS 1.3. The certificate in `faig-public-tls` showed issuer Let's Encrypt YE1, with validity October 13, 2026 to January 11, 2027. Check the live endpoint again after changes.

## Architecture and certificate separation

```text
LLM Client -- HTTPS :30443 (Let's Encrypt) --> NGINX Ingress
                                                    |
                                                    | HTTPS (existing internal TLS)
                                                    v
                                        FortiAIGate API / LLM Gateway
                                                    |
                                               Other services

NGINX Ingress external certificate: faig-public-tls
FortiAIGate internal certificate: fortiaigate-tls-secret (do not modify)
```

The certificate is validated against the **hostname**, not port `30443`. No public inbound connection is necessary for DNS-01 issuance, but the DNS TXT record must be visible in authoritative public DNS.

## Step 1 — Check existing certificate and Ingress

```bash
kubectl get pods -n fortiaigate
kubectl get secrets -n fortiaigate
kubectl get ingress -n fortiaigate -o yaml
helm list -n fortiaigate
```

Confirm that the Ingress has a TLS stanza resembling:

```yaml
spec:
  ingressClassName: nginx-fortiaigate
  tls:
    - hosts:
        - faig.dell.cbcgslb.fortiseahk.com
      secretName: faig-public-tls
```

Inspect the certificate stored in the public Secret:

```bash
kubectl get secret faig-public-tls -n fortiaigate \
  -o jsonpath='{.data.tls\.crt}' | base64 -d | \
  openssl x509 -noout -subject -issuer -dates -ext subjectAltName
```

Do **not** replace or edit `fortiaigate-tls-secret`: the original application Helm values use internal TLS for application components, including PostgreSQL and Redis references to that Secret.

## Step 2 — Issue the certificate (only if a usable certificate is not available)

On an Ubuntu machine with outbound HTTPS access:

```bash
sudo apt update
sudo apt install -y certbot dnsutils
sudo certbot certonly \
  --manual \
  --preferred-challenges dns \
  --cert-name faig-public \
  -d faig.dell.cbcgslb.fortiseahk.com \
  --agree-tos \
  -m YOUR_EMAIL@example.com \
  --no-eff-email
```

Certbot prints a **unique** challenge value. Ask the public DNS administrator to create:

| Property | Value |
|---|---|
| Type | `TXT` |
| Name | `_acme-challenge.faig.dell.cbcgslb.fortiseahk.com` |
| Content | **The exact value printed by Certbot** |
| TTL | `300` seconds (if supported) |

Verify propagation **before** pressing Enter in Certbot:

```bash
dig TXT _acme-challenge.faig.dell.cbcgslb.fortiseahk.com @1.1.1.1
```

Upon issuance, the files normally appear at:

```text
/etc/letsencrypt/live/faig-public/fullchain.pem
/etc/letsencrypt/live/faig-public/privkey.pem
```

> Let's Encrypt does not need to reach the Kubernetes application over the internet. DNS-01 validates ownership of the domain through the public TXT record.

## Step 3 — Create or restore the external TLS Secret

If `faig-public-tls` does not exist in the `fortiaigate` namespace, create it:

```bash
sudo kubectl create secret tls faig-public-tls \
  --namespace fortiaigate \
  --cert=/etc/letsencrypt/live/faig-public/fullchain.pem \
  --key=/etc/letsencrypt/live/faig-public/privkey.pem
```

If the Secret exists but its certificate should be updated, use an apply operation:

```bash
sudo kubectl create secret tls faig-public-tls \
  --namespace fortiaigate \
  --cert=/etc/letsencrypt/live/faig-public/fullchain.pem \
  --key=/etc/letsencrypt/live/faig-public/privkey.pem \
  --dry-run=client -o yaml | sudo kubectl apply -f -
```

**Kubeconfig note:** If `sudo kubectl` cannot access your cluster, use `kubectl` under your normal user account and make the certificate/key files securely accessible to that account. Never store the private key in Git.

Verify:

```bash
kubectl get secret faig-public-tls -n fortiaigate
```

## Step 4 — Point the live Ingress at the external Secret

Determine the actual name:

```bash
kubectl get ingress -n fortiaigate
```

The observed Ingress name was `fortiaigate-ingress`. Save its definition first:

```bash
kubectl get ingress fortiaigate-ingress -n fortiaigate -o yaml > ingress-backup.yaml
```

If the existing Ingress does **not** reference `faig-public-tls`, and its TLS list contains **only this hostname**, patch it:

```bash
kubectl patch ingress fortiaigate-ingress \
  -n fortiaigate \
  --type=merge \
  -p '{
    "spec": {
      "tls": [{
        "hosts": ["faig.dell.cbcgslb.fortiseahk.com"],
        "secretName": "faig-public-tls"
      }]
    }
  }'
```

> **Important:** This patch replaces `spec.tls`. If multiple TLS entries or domains exist, edit the appropriate entry instead of using this patch. The live patch is an immediate fix; it does **not** guarantee that a future Helm reconciliation will preserve the value.

Check the result:

```bash
kubectl get ingress fortiaigate-ingress -n fortiaigate \
  -o jsonpath='{.spec.tls}'
```

## Step 5 — Make the Ingress Secret reference persistent in Helm

First, find what the **currently installed Helm release** contains:

```bash
helm list -n fortiaigate
helm get manifest fortiaigate -n fortiaigate > deployed-manifest.yaml
helm get values fortiaigate -n fortiaigate -a > helm-values-backup.yaml
```

Check whether the Helm release already renders `faig-public-tls`:

```bash
grep -A 65 'kind: Ingress' deployed-manifest.yaml
```

If the *Helm manifest* already refers to `faig-public-tls`, keep a copy of the actual chart, values, and any template changes used for that release. Ensure the public Secret is restored **before** future reinstallations.

If the manifest uses a different Secret, modify **only the Ingress Helm template**, not the shared application TLS settings. For example, add a new chart-specific value:

```yaml
ingress:
  className: nginx-fortiaigate
  host: faig.dell.cbcgslb.fortiseahk.com
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
  publicTls:
    enabled: true
    secretName: faig-public-tls
```

**Note:** `ingress.publicTls` is a *proposed chart customization*, **not** a verified option in the stock FortiAIGate chart. Its value does nothing unless the Ingress template uses it.

In the chart's existing Ingress template, adapt the **existing** `spec.tls` section (do not duplicate the `spec` or `tls` fields):

```yaml
{{- if .Values.ingress.publicTls.enabled }}
tls:
  - hosts:
      - {{ .Values.ingress.host | quote }}
    secretName: {{ .Values.ingress.publicTls.secretName | quote }}
{{- end }}
```

Place this snippet at the correct indentation beneath `spec:` while keeping all existing `rules`, routes, and backend services. Preserve the original internal TLS values:

```yaml
tls:
  enabled: true
  mountPath: /etc/tls
  cert: files/certificate/dflt.crt
  key: files/certificate/dflt.key
```

From the **matching chart directory**, inspect the rendered output before upgrade:

```bash
helm template fortiaigate . -n fortiaigate -f values.yaml > rendered.yaml
grep -A 65 'kind: Ingress' rendered.yaml
grep -n 'fortiaigate-tls-secret' rendered.yaml
```

Review the entire rendered diff to make sure there are no unintended changes to services, storage, or TLS for internal components. Then, and only then, upgrade:

```bash
helm upgrade fortiaigate . -n fortiaigate -f values.yaml
```

Avoid using an unrelated or older chart bundle for this step.

## Step 6 — Verify the actual endpoint (port 30443)

```bash
openssl s_client \
  -connect faig.dell.cbcgslb.fortiseahk.com:30443 \
  -servername faig.dell.cbcgslb.fortiseahk.com \
  -verify_return_error </dev/null
```

Expected verification status:

```text
Verify return code: 0 (ok)
```

Also inspect the certificate being **served**, not just the Kubernetes Secret:

```bash
echo | openssl s_client \
  -connect faig.dell.cbcgslb.fortiseahk.com:30443 \
  -servername faig.dell.cbcgslb.fortiseahk.com \
  2>/dev/null | openssl x509 -noout -subject -issuer -dates -ext subjectAltName
```

Test an HTTPS request **without** `-k`:

```bash
curl -v https://faig.dell.cbcgslb.fortiseahk.com:30443/
```

An HTTP redirect or authentication response may be normal; the TLS handshake must succeed with hostname and certificate verification. TLS certificates are independent of port numbers, so port `30443` does not require a special certificate.

## Step 7 — Renewal when DNS has no API

Manual DNS-01 requires operator involvement for each renewal. Before expiration, request a renewed certificate with the same certificate name:

```bash
sudo certbot certonly \
  --manual \
  --preferred-challenges dns \
  --cert-name faig-public \
  -d faig.dell.cbcgslb.fortiseahk.com \
  --force-renewal
```

Publish the **new** TXT challenge value, verify public DNS propagation, and complete Certbot. Then update `faig-public-tls` using the **apply** command in Step 3 and retest Step 6.

Because DNS changes are manual, do **not** assume unattended renewal works. Add an expiration reminder or monitoring alert, and consider delegating `_acme-challenge` to an automated DNS validation service if unattended renewal is needed later.

## Reinstallation checklist

1. Preserve the external `fullchain.pem` and `privkey.pem` securely outside the FortiAIGate Helm release.
2. If the namespace was removed, recreate `fortiaigate` and restore the `faig-public-tls` Secret before deploying the chart.
3. Install using the maintained Helm chart and values that reference `faig-public-tls` under `spec.tls` of the Ingress.
4. Do not replace the application-wide TLS certificate, key, or `fortiaigate-tls-secret`.
5. Verify the live Ingress and Helm release manifest agree.
6. Test HTTPS on `30443` from an LLM client with normal certificate validation enabled.

## Troubleshooting

| Symptom | Check |
|---|---|
| Certificate works before reinstall but changes afterward | Compare `kubectl get ingress -o yaml` with `helm get manifest`; ensure Helm template references `faig-public-tls`. |
| Browser/client sees a different certificate | Inspect SNI, `spec.tls.secretName`, hostname resolution, and the certificate presented by `:30443`. |
| `faig-public-tls` is missing | Restore the Secret in namespace `fortiaigate` from valid certificate and key files. |
| Let's Encrypt challenge fails | Check the authoritative public TXT record and make sure the **current** challenge value has propagated. |
| TLS verification fails despite a valid Secret | Check hostname/SAN, certificate chain, expiry, SNI and whether the actual Ingress serves the Secret. |
| Backend returns a 502/503 | Diagnose NGINX-to-FortiAIGate backend connectivity separately. The internal backend still uses HTTPS. |

## Safety notes

- Keep external ingress TLS and internal service TLS separate.
- Do not commit private keys, TLS Secrets, or credentials to a Git repository.
- A certificate Secret can survive a Helm uninstall, but it does **not** survive deletion of its namespace unless backed up and restored.
- Changing `ingress.publicTls` in values alone will not work unless the Helm chart supports it.
- Ingress configuration must retain `nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"` to preserve the existing encrypted upstream connection.
