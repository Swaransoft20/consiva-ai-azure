# Consiva.ai on Azure Kubernetes Service

Deployment guide for **Consiva.ai — Self-Hosted Consent Management**, available on Microsoft
Marketplace as an Azure Container offer (Kubernetes application).

Consiva.ai is a DPDP and GDPR consent management platform that deploys into your own Azure
Kubernetes Service cluster. It is not a SaaS tenant: the application runs on your nodes, writes
to a SQL Server instance you control, and the consent records it produces never leave your
infrastructure. Licence verification is performed entirely offline — the application works in a
cluster with no outbound internet access at all.

## Deployment guide

**[USER_GUIDE.md](USER_GUIDE.md)** — prerequisites, deploying from the Azure portal, verifying
the deployment, exposing the application, activating Enterprise with a licence key, operating
notes and troubleshooting.

## Licensing

The deployment runs on the **Free tier** by default. No licence key is required and no contact
with Consiva is needed. A signed Enterprise licence key, obtained directly from Consiva, unlocks
the Enterprise tier until the key expires — after which the deployment returns to the Free tier
automatically and keeps running.

To discuss Enterprise terms, use **Request private offer** on the Marketplace listing.

## Support

support@consiva.ai

---

Consiva.ai is a product of Swaran Soft Support Solutions Private Limited.
This repository contains documentation only. © Swaran Soft Support Solutions Private Limited.
