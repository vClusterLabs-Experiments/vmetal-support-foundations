# Glossary

The single source for approved vCluster terminology used across this curriculum. Every milestone, every brief, every chat reply, and every commit message uses the **Approved** column. Linked from `CLAUDE.md` and from milestones that introduce these terms (M5, M6).

If a learner, customer, or contributor uses a term in the **Never use** column, restate it using the approved term before proceeding. Do not edit direct quotes or external sources.

## Term mapping

| Approved | Never use | Why the distinction matters |
|---|---|---|
| Tenant Cluster | Sharing Cluster, Virtual Cluster (when meaning the cluster) | A Tenant Cluster is the cluster a tenant consumes. The **Virtual Control Plane** is the component that runs the tenant's API server, not the cluster itself. |
| Tenant Isolation | multi-tenancy, multitenancy, Multi-Tenancy | "Isolation" names the operational property (one tenant cannot see, schedule onto, or interfere with another's nodes, GPUs, or data plane). "Multi-tenancy" names a vague architectural style. |
| AI Cloud | Neocloud, neocloud | The operator/customer environment vmetal targets: bare-metal GPU fleets serving multiple tenants. |
| Control Plane Cluster | Host Cluster | The cluster that hosts vmetal's controllers and the virtual control planes for tenants. CAPI's "management cluster" maps onto this, but with the added vmetal role. |
| Virtual Control Plane | (do not substitute the cluster terms above) | Component name only. Refers to the per-tenant API server stack, not the cluster. |
| vCluster | vcluster, V-Cluster, VCluster | Product name, capitalization fixed. Lowercase v, uppercase C. |
| Sharing GPUs | (rewrite the sentence) | Do not use. If you mean GPU partitioning, MIG, time-slicing, or fractional allocation, name the specific mechanism. |

## Usage rules

1. **In curriculum content:** use approved terms in prose and headings. The first time a term appears in a milestone, you may add a one-line gloss for the learner; do not restate the full table (link here instead).
2. **In chat replies and commits:** use approved terms. If a user writes a legacy term, gently restate using the approved term ("Using our current terminology, *Tenant Cluster*, ...") before continuing.
3. **In code, configs, and CRD identifiers:** use whatever the upstream/product code requires. Do not "correct" upstream CAPI's `Cluster` kind or vmetal's CRD names to match this glossary.
4. **In external quotes:** preserve the original wording. Add a `[sic]` or footnote if disambiguation is needed.

## Updating this file

This glossary is part of the curriculum's brief content (CC BY-NC 4.0). Add a new row when a new term enters the curriculum. Move a term to **Never use** only after it has been replaced everywhere; grep before promoting:

```bash
git grep -i "<deprecated term>" -- ':!GLOSSARY.md'
```
