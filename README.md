# SOBR Calculator by MJ
### for Veeam Backup and Replication
**Developed by Martin Jorge** — [martin.jorge@veeam.com](mailto:martin.jorge@veeam.com)

A sizing and upload-time estimator for offloading an on-premises **Veeam Backup & Replication** SOBR to **Veeam Data Cloud Vault**, complementing the [official Veeam calculator](https://www.veeam.com/calculators/simple/vdc) with additional functionality.

The input panel mirrors the **Capacity Tier** page of the VBR *Scale-out Backup Repository* wizard, so the settings you fill in here are the same ones you set in the console.

---

## What does it calculate?

- **VDC Vault capacity** — full GFS retention (daily, weekly, monthly, yearly)
- **Immutability overhead** — with a step-by-step explanation of why it exists
- **Both VBR immutability modes** — for the entire retention duration, or for the minimum period only
- **Upload profile** — the one-time seed of the whole compressed source, then the daily change. Object storage offload is block-based, so unchanged blocks are never re-sent and there is no day that re-uploads a full.
- **Upload requirements**, in either direction:
  - **Required bandwidth** — give it a backup window, get the Mbps you need
  - **Backup duration** — give it the available Mbps, get how long the upload takes and whether it fits the window
- **Latency ceiling** — TCP throughput modelled from RTT, parallel tasks and proxies, so you can see when latency (not the link) is the bottleneck
- **Azure- and AWS-backed Vault regions**, with the provider-specific Block Generation period applied automatically
- **Proxy sizing** — vCPU, RAM and repository disk throughput
- **S3 API overhead** — object counts derived from the configured storage optimization block size
- **Visual simulation** of restore points with active lock indicators
- **Consistency validation** — warns when immutability periods conflict with retention settings

Scope note: this tool sizes the **Capacity Tier only**. The on-premises Performance Tier is assumed to already exist and is not calculated.

---

## Immutability modes

VDC Vault always enforces immutability through Object Lock. VBR offers two ways to apply it, and the choice changes the overhead substantially:

| Mode | Effective lock | Overhead |
|---|---|---|
| For the entire duration of their retention policy *(recommended)* | `max(minimum, retention)` | Highest — nothing can be deleted early |
| For the minimum immutability period only | the configured minimum | Lower — space is reclaimed as soon as retention allows |

## Block Generation

On top of the immutability period, Veeam adds a **Block Generation** window that it fixes per provider and does **not** expose in the console. Blocks written inside one generation share a single expiration date, which saves API calls — and keeps them billable past your retention.

| Backing cloud | Generation |
|---|---|
| AWS — Amazon S3 | 30 days |
| Azure — Blob ("all other types") | 10 days |

VDC Vault is offered on Azure and AWS only, so these are the two cases the calculator models. Both values are quoted from the [Block Generation](https://helpcenter.veeam.com/docs/vbr/userguide/block_gen.html?ver=13) page, which also states plainly: *"You do not have to configure it, the Block Generation setting is applied automatically."*

The calculator derives this from the Vault region you pick, so an AWS-backed region carries 20 more days of lock than an Azure one for the same settings. The overhead is modelled as the data still locked once retention has released it:

```
stranded_days = (immutability + block_generation) − retention
overhead ≈ daily_incremental × stranded_days
         + compressed_full × ⌈stranded_days ÷ block_generation⌉
```

There is no synthetic-full cadence to configure here: because the offload is block-based, a synthetic full on the performance tier does not re-transfer blocks the Vault already holds.

---

## On the latency figures

The built-in round-trip times are **hand-written estimates with no published source**. They are not measurements and they are not drawn from any dataset; AWS regions reuse the figure of their co-located Azure region.

This matters more than it looks: RTT is the denominator of the bandwidth-delay-product ceiling, so an estimate that is off by 2× moves every duration the calculator reports by the same factor. For anything you intend to commit to, measure the real RTT from a proxy (`ping` or `traceroute` to the Vault endpoint) and switch *Round-trip time* to **I measured it myself**. The measured value overrides the table.

The same caveat applies to two constants feeding that ceiling — the assumed 256 KB TCP window and the mapping of one VBR parallel task to one TCP stream. The bandwidth-delay-product formula itself is standard networking; these inputs to it are estimates, and both are marked as assumptions in the source.

---

## How to run it

Clone the repository and open `index.html` in a browser. No build tools, no server, no dependencies to install:

```bash
git clone https://github.com/martinljor/veeamcalc
```

Google Fonts and Font Awesome are loaded from a CDN — without internet access the calculator still works, just without the custom typeface and icons.

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
```

The version shown in the header comes from the `APP_VERSION` constant in `index.html`.
