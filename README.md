# SOBR Calculator by MJ
### for Veeam Backup and Replication
**Developed by Martin Jorge** — [martin.jorge@veeam.com](mailto:martin.jorge@veeam.com)

A storage and resource estimator for **Veeam Backup & Replication** with **VDC Vault**, complementing the [official Veeam calculator](https://www.veeam.com/calculators/simple/vdc) with additional functionality.

---

## What does it calculate?

- **Performance Tier** and **Capacity Tier** storage separately
- **Full GFS retention** — daily, weekly, monthly, yearly
- **Immutability overhead** — with a step-by-step explanation of why it exists
- **Visual simulation** of restore points with active lock indicators per restore point
- **Bandwidth requirements** — daily incremental vs. peak (synthetic full day)
- **Proxy sizing** — vCPU, RAM, and repository disk throughput based on configured bandwidth
- **Consistency validation** — warns when immutability periods conflict with retention settings

---

## Simulation modes

### Full SOBR (Performance + Capacity Tier)
For new deployments. Calculates both tiers from scratch based on source data, change rate, compression, GFS retention, and immutability settings.

### Capacity Tier only (VDC Vault extension)
For customers with an existing on-premises SOBR who want to extend to VDC Vault. Only calculates the Capacity Tier storage and the bandwidth/proxy requirements to send data to the Vault.

- With **Copy policy**: incrementals and synthetic fulls are both copied — daily change rate and daily retention apply.
- With **Move policy** only: only synthetic fulls are moved — daily change rate and incremental retention are not applicable.

---

## How to deploy on GitHub Pages

1. Upload `index.html` and `README.md` to the root of your repository
2. Go to **Settings → Pages**
3. Under *Source*, select **Deploy from a branch**
4. Choose the `main` (or `master`) branch and `/ (root)` folder
5. Save — within 1–2 minutes your URL will be: `https://<your-username>.github.io/<repo>/`

## Project structure

```
/
└── index.html     ← entire app in a single file (no build tools required)
