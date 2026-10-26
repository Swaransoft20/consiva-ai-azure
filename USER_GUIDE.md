# Consiva.ai on Azure Kubernetes Service — deployment guide

Consiva.ai is a DPDP cookie and consent management platform. This guide covers deploying it from
Microsoft Marketplace into an Azure Kubernetes Service (AKS) cluster you already own, supplying
the configuration it needs, exposing it to your users, and activating an Enterprise licence.

The application runs entirely inside your cluster. It makes no call to Consiva at runtime — not to
check your licence, not for telemetry, not at all.

---

## 1. Before you start

You need:

| | |
|---|---|
| **An AKS cluster** | Running, and in the resource group you will deploy from. The deployment installs into it as a cluster extension; it does not create a cluster. |
| **A SQL Server database** | Azure SQL Managed Instance, SQL Server on an Azure VM, or any SQL Server instance your cluster can reach. **Consiva.ai ships no database** — your consent records are your own regulated personal data and stay in a database you control. |
| **An SMTP relay** | Host, port, username, password and a from-address you are permitted to send as. |
| **The URL you will serve the application at** | For example `https://consent.example.com`. |
| **A licence key** | Only if you have bought Enterprise. Leave it blank for the Free tier. |

The application creates its own database schema the first time it starts. Point it at an empty
database; it does the rest.

**Resources used.** One Pod with two containers, requesting 750m CPU and 1.5Gi memory in total.
No CPU or memory limits are set, so the Pod is free to burst during startup, when it creates the
schema and seeds reference data.

---

## 2. Deploy from the Azure portal

From the offer in Microsoft Marketplace, choose **Create**.

### Basics

| Field | What to enter |
|---|---|
| **Subscription / Resource group** | The resource group containing the AKS cluster you want to install into. |
| **AKS cluster** | Pick your cluster from the list. Only clusters in the selected resource group appear. |
| **Extension resource name** | A name for this installation, 6–30 lowercase letters and digits — for example `consivaprod`. Every Kubernetes object created is prefixed with it. |
| **Kubernetes namespace** | The namespace to install into, created if it does not exist. Default `consiva`. One instance per namespace, so a second installation on the same cluster simply needs a different namespace. |

### Application

**Application URL** — the origin your users will reach Consiva.ai at, scheme and host only, no
path and no trailing slash, for example `https://consent.example.com`.

This one value drives three things: the links inside verification and password-reset emails, the
CORS allowlist that decides which sites the dashboard answers, and the consent SDK snippet you
embed in your websites. Getting it wrong does not stop the application starting — it produces
emails with dead links and a banner that will not load. Set it to the address you will actually
publish, before you publish it.

### Database

Paste a full SQL Server connection string, for example:

```
Server=tcp:consiva-sql.database.windows.net,1433;Database=consiva;User ID=consiva;Password=<password>;Encrypt=True;TrustServerCertificate=False;IPAddressPreference=IPv4First
```

**Include `;IPAddressPreference=IPv4First`.** Without it the SQL client can fail to resolve the
server from inside a Linux container and the application never reaches your database.

The account in the string needs permission to create tables in that database, because the schema
is created on first start.

### Security

**Session signing key** — at least 32 characters of random data. It signs the login sessions this
installation issues to your own users.

Generate a fresh one, and never reuse a key between installations: anyone holding it can mint a
valid session for your tenant. Any of these will do:

```bash
openssl rand -base64 48
```
```powershell
[Convert]::ToBase64String((1..48 | ForEach-Object { Get-Random -Max 256 }))
```

This is not the licence key and not the licence verification key. The verification key is built
into the application and cannot be changed by anyone deploying it — that is deliberate.

### Email (SMTP)

Host, port (usually 587), username, password, from-address and from-name.

**All of these are required and the application will not start without them.** That is on purpose:
verification codes go to members of the public exercising statutory data rights, who have no way
to recover from a silently dropped message. A broken mail path should be loud, not quiet.

### Licence (optional)

Leave blank for the **Free tier** — 1 domain, 1,000 cookie consents, 50 form consents, 90-day
retention. A blank key is entirely valid and is the normal case.

If you have bought an **Enterprise** licence, Consiva emails you a signed key. Paste it here. See
section 5.

Then **Review + create**.

---

## 3. After deployment

Check the installation:

```bash
az aks get-credentials --resource-group <rg> --name <cluster>
kubectl get pods -n consiva
kubectl get svc  -n consiva
```

You should see one Pod with 2/2 containers ready, and two Services on port 80 — one for the
application UI, one for the API.

**Health endpoints, and which one to watch.** The backend exposes two, and they mean different
things:

| Endpoint | Meaning |
|---|---|
| `/health` | **Liveness.** Returns 200 as soon as the process is up — including when it is running *degraded* because the database is not reachable yet, in which case the response body says `"status":"degraded"`. This is what the Pod probes use, so that a database arriving late does not crash-loop a perfectly good deployment. |
| `/health/ready` | **Readiness in the honest sense.** Returns 503 until the schema and the seed data are actually in place, 200 once they are. |

**Point your own monitoring and alerting at `/health/ready`.** `/health` alone cannot tell a
working deployment from one that will never converge.

If the Pod is running but `/health/ready` stays 503, the database is the first thing to check:

```bash
kubectl logs -n consiva deploy/<extension-resource-name> -c backend --tail=50
```

The backend retries in the background and converges on its own as soon as the database becomes
reachable — no restart, no redeploy.

---

## 4. Exposing the application

**No Ingress is created by default.** An Ingress provisions an external load balancer, which
regularly takes longer to become healthy than the deployment is given, so the install would be
judged failed while the load balancer was still coming up. Exposing the application is therefore a
deliberate step you take afterwards.

Two Services are created and ready to be routed to:

| Service | Port | Serves |
|---|---|---|
| `<name>-frontend` | 80 → 3000 | the dashboard |
| `<name>-backend` | 80 → 5000 | `/api/*`, `/sdk.js`, `/sdk.min.js` |

Both must be served from the **same origin** — the one you entered as the Application URL. The
dashboard calls the API as a relative path, and the consent SDK is loaded from that origin by your
websites.

### Option A — your own ingress controller (recommended)

Route by path to the two Services, most specific first:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: consiva
  namespace: consiva
  annotations:
    # your controller's TLS / cert-manager annotations here
spec:
  ingressClassName: <your-ingress-class>
  rules:
    - host: consent.example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend: { service: { name: <name>-backend,  port: { number: 80 } } }
          - path: /sdk.js
            pathType: Exact
            backend: { service: { name: <name>-backend,  port: { number: 80 } } }
          - path: /sdk.min.js
            pathType: Exact
            backend: { service: { name: <name>-backend,  port: { number: 80 } } }
          - path: /
            pathType: Prefix
            backend: { service: { name: <name>-frontend, port: { number: 80 } } }
```

The `/sdk.js` and `/sdk.min.js` rules are not optional. The consent SDK is a static file served by
the *backend*; without those two rules it falls through to the frontend, which has no such file,
and every website embedding your banner gets a 404.

Terminate TLS at the ingress and point the DNS name in your Application URL at it.

### Option B — the chart's built-in Ingress

The chart contains the same routing, switched off. Turn it on with an extension update:

```bash
az k8s-extension update \
  --name <extension-resource-name> \
  --cluster-name <cluster> --resource-group <rg> --cluster-type managedClusters \
  --configuration-settings ingress.enabled=true ingress.className=<your-ingress-class>
```

It has no `host` and no TLS configuration, so it is a starting point, not a finished edge. Option A
gives you somewhere to put certificates and controller-specific annotations.

### Quick check without exposing anything

```bash
kubectl port-forward -n consiva svc/<name>-frontend 8080:80
```

Then open `http://localhost:8080`. The first visit redirects to `/login`, which is correct.

---

## 5. Activating Enterprise with a licence key

### How licensing works

- **No key → Free tier.** The application runs, fully, with Free tier limits.
- **Valid signed key → Enterprise**, until the moment the key expires.
- At expiry the deployment **drops back to Free. It does not stop running and nothing is
  deleted.** Your data stays exactly where it is.
- The key is a signed token verified **inside your own cluster**, offline, against a public key
  built into the application. Consiva is never contacted, the key is never transmitted anywhere,
  and the application works in a cluster with no outbound internet access at all.
- Expiry is re-checked continuously, not just at startup, so a long-running deployment changes
  tier on its own at the right moment without a restart.

To buy Enterprise, contact Consiva. You are sent a key file; its contents are one long line of
text.

### Adding or replacing a key on a running deployment — no data loss

Supplying a licence key later, or replacing an expiring one, is a configuration change. **It never
touches your database.** The Pod is replaced automatically — the chart hashes its configuration
into the Pod template, so changing the key rolls the deployment with no manual step from you. It
takes a few seconds, and every consent record, domain and setting is untouched.

The same is true of any other setting you change this way, such as the Application URL or your
SMTP credentials: the change reaches the running containers without a restart command.

> ### ⚠ Read this before you run the update
>
> **An extension update replaces the whole settings map. It does not merge into it.**
>
> If you supply only the licence key, every other protected setting — your database connection
> string, your session signing key and your SMTP password — is **wiped**, and the new Pod cannot
> start. You will see `The ConnectionString property has not been initialized` and
> `DEPLOYMENT_MODE=self-hosted requires SMTP configuration. Missing: Smtp:Password.`
>
> Your running service is not interrupted: Kubernetes keeps the previous Pod because the new one
> never becomes ready. But **Enterprise never activates and the rollout stays stuck** until you
> correct it — after which it completes by itself.
>
> **Always pass every setting in the same call**, in both `--config-settings` and
> `--config-protected-settings`.

**Azure CLI only.** Protected settings cannot be changed in the Azure portal — the portal states
this itself on the extension's configuration page.

First read back your current non-secret settings, so you can resupply them exactly:

```bash
az k8s-extension show \
  --name <extension-resource-name> \
  --cluster-name <cluster> --resource-group <rg> --cluster-type managedClusters \
  --query configurationSettings -o json
```

Then update, passing everything:

```bash
az k8s-extension update \
  --name <extension-resource-name> \
  --cluster-name <cluster> --resource-group <rg> --cluster-type managedClusters \
  --config-settings \
      appDomainUrl='<your application URL>' \
      name='<extension-resource-name>' \
      namespace='<namespace>' \
      smtp.host='<host>' smtp.port='587' smtp.username='<username>' \
      smtp.fromEmail='<from address>' smtp.fromName='<from name>' \
  --config-protected-settings \
      db.connectionString='<your connection string>' \
      jwt.secret='<your existing signing key>' \
      smtp.password='<your SMTP password>' \
      enterpriseLicenseKey='<paste the key here>'
```

`jwt.secret` must be the **same value you deployed with**. Changing it invalidates every existing
session.

**Keep your deployment parameters in source control or a secret store.** Protected settings
cannot be read back out of Azure — `az k8s-extension show` returns the non-secret ones only. If
you have not recorded them, you cannot resupply them, and you cannot change any setting without
them.

Keep secrets in `--config-protected-settings`, never in `--config-settings`: protected settings
are not readable back out of the resource.

> `--configuration-settings` and `--configuration-protected-settings` still work but are
> deprecated and will be removed. Use the shorter forms above.

Confirm the tier in the dashboard under **Settings → Billing**, which reads the same licence state
the backend enforces. The backend also logs it on every start, which is the quickest check:

```bash
kubectl logs -n <namespace> deploy/<extension-resource-name> -c backend | grep -i licen
```

A licensed deployment logs `Enterprise license key verified (issued to: …, expires: …)` followed
by `Self-hosted tier at startup: Enterprise (licensed)`. Without a key it logs `No Enterprise
license key configured — this deployment runs on the Free tier.`

To clear a key and return to Free, set it to an empty string.

---

## 6. Operating notes

**Backups are yours.** The application holds no state outside the database you supplied — back
that up the way you would any other production SQL Server database, and your installation's full
state is captured. Keep a copy of the session signing key with it: restoring a database against a
different signing key invalidates every existing login session (users simply sign in again).

**Upgrades.** A new version is an extension update; the application migrates its own schema on
start. Take a database backup first, as with any schema change.

**Scaling.** The deployment runs a single replica. Background jobs use SQL Server–backed
scheduling, so additional replicas do not duplicate scheduled work, but multi-replica operation
has not been exercised by Consiva — treat it as untested rather than unsupported, and talk to us
before relying on it.

**Uninstalling.** Deleting the extension removes the Kubernetes objects. **Your database is not
touched** — delete it separately if you intend to destroy the data, and be certain of your
retention obligations before you do.

---

## 7. Troubleshooting

| Symptom | Cause and fix |
|---|---|
| Pod `0/2`, backend restarting | Almost always SMTP: the backend deliberately fails fast when host, username, password or from-address is missing. Check `kubectl logs ... -c backend`. |
| `/health` 200 but `/health/ready` 503 | The database is not reachable, or the account cannot create the schema. The backend retries and converges on its own once that is fixed. Check the connection string, firewall rules, and that `;IPAddressPreference=IPv4First` is present. |
| Emails arrive with broken links | The Application URL does not match the address users actually reach. Update it — **resupplying every setting**, see section 5. |
| Consent banner 404s on `/sdk.js` | The `/sdk.js` and `/sdk.min.js` routes are missing from your Ingress — see section 4. |
| Dashboard loads, API calls fail CORS | Same cause: the Application URL does not match the browser's origin. |
| **I applied my licence key and nothing changed; a Pod is crash-looping** | You supplied only the key on the update, so your other protected settings were wiped. Resupply all of them — see the warning in section 5. |
| **The new Pod will not start, but the application still works** | Kubernetes is holding the previous Pod because the new one never became ready. Nothing is down. Correct the configuration and the rollout completes by itself. |
| Still on Free after supplying a key | Confirm the key went into **protected** settings and was pasted whole, with no line breaks. Check `Settings → Billing`. If the key has expired, the deployment is correctly back on Free — ask Consiva for a renewal. |

For support, contact Consiva with your extension resource name, namespace, AKS cluster region and
the last 50 lines of backend logs. **Never send us your licence key, session signing key or
connection string.**
