# SOBR Calculator by MJ
### for Veeam Backup and Replication
**Developed by Martin Jorge** — [martin.jorge@veeam.com](mailto:martin.jorge@veeam.com)

A **resource** estimator for offloading an on-premises **Veeam Backup & Replication** SOBR to **Veeam Data Cloud Vault**: what bandwidth, latency headroom, proxies and repository disk the offload needs, and why each number comes out the way it does.

Capacity is reported too, but only as a rough estimate to drive those resource figures. **For a TB number you intend to quote or buy against, use the [official Veeam VDC calculator](https://www.veeam.com/calculators/simple/vdc).**

The input panel mirrors the **Capacity Tier** page of the VBR *Scale-out Backup Repository* wizard, so the settings you fill in here are the same ones you set in the console.

---

## What does it calculate?

- **VDC Vault capacity** — full GFS retention (daily, weekly, monthly, yearly)
- **Immutability overhead** — with a step-by-step explanation of why it exists
- **Both VBR immutability modes** — for the entire retention duration, or for the minimum period only
- **Upload profile** — the one-time seed of the whole compressed source, then the daily change. Object storage offload is block-based, so unchanged blocks are never re-sent and there is no day that re-uploads a full.
- **Upload requirements**, in either direction:
  - **Time** — give it the maintenance window, get the link that meets it (sized on the initial seed, since that is the binding transfer) plus the lower rate that only keeps pace with the daily change
  - **Bandwidth** — give it the link the customer actually has, get how long the seed and each daily change take, and whether they fit the window
- **Latency ceiling** — TCP throughput modelled from RTT, parallel tasks and proxies, so you can see when latency (not the link) is the bottleneck
- **Azure- and AWS-backed Vault regions**, with the provider-specific Block Generation period applied automatically
- **Proxy sizing** — vCPU, RAM and repository disk throughput
- **On-prem repository load** by backup mode — forever forward, periodic synthetic full or periodic active full, which cost the repository very differently
- **S3 API overhead** — object counts derived from the configured storage optimization block size
- **Visual simulation** of restore points with active lock indicators
- **Consistency validation** — warns when immutability periods conflict with retention settings

Scope note: this tool sizes the **Capacity Tier**. The on-premises Performance Tier is assumed to already exist — its capacity is not calculated — but the repository I/O its backup mode implies *is* reported, because that is frequently the real constraint rather than the link.

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
         + new_full × ⌈stranded_days ÷ full_cycle⌉    (periodic full modes only)
new_full  = min(compressed_full, daily_incremental × full_cycle)
```

The second term depends on the backup mode, and for one mode it does not exist at all:

- **Forever forward incremental** never starts a new chain, so there is no periodic full and nothing is added for one.
- **Periodic synthetic or active full** does start one — but a new full does not cost a full. The offload is block-based, so the new chain reuses everything the Vault already holds and only the change since the previous full is new. Same rule as the GFS points below.

A longer full cycle therefore costs *more* per full, not less: each one carries more accumulated change.

---

## Backup mode and the repository

The backup mode on the on-prem job does not change what reaches the Vault — block reuse means only changed blocks travel either way — but it changes what the **repository** has to sustain, by a factor of two between the two periodic modes:

| Mode | Repository reads | Repository writes |
|---|---|---|
| Forever forward incremental | the daily increment | the daily increment |
| Periodic **synthetic** full | a full | a full |
| Periodic **active** full | — | a full |

A synthetic full is the heaviest of the three on the repository: it assembles the new full from blocks the repository already holds, so the repository does both sides of the copy. An active full re-reads from production, so that read lands on the production storage instead.

---

## GFS points and shared blocks

GFS points do **not** each cost a full. The Vault stores unique blocks, and consecutive points share nearly everything — what a point adds is the change accumulated since the previous one, capped at a full:

```
gfs ≈ min(full, daily_change × 7)   × weeklies
    + min(full, daily_change × 30)  × monthlies
    + min(full, daily_change × 365) × yearlies
```

For 10 TB at 50% compression and 3% daily change, 4 weeklies + 12 monthlies comes to **58 TB rather than 80 TB**. The weeklies are where most of the difference is: a week of change is a fraction of a full, while a month of 3% daily change is already most of one.

They do still **pin** those blocks for their retention, which is why they appear in the capacity figure at all. What they do not do is re-upload, so they add nothing to the bandwidth estimate.

---

## On the latency figures

The built-in round-trip times are **hand-written estimates with no published source**. They are not measurements and they are not drawn from any dataset; AWS regions reuse the figure of their co-located Azure region.

This matters more than it looks: RTT is the denominator of the bandwidth-delay-product ceiling, so an estimate that is off by 2× moves every duration the calculator reports by the same factor. For anything you intend to commit to, measure the real RTT from a proxy (`ping` or `traceroute` to the Vault endpoint) and switch *Round-trip time* to **I measured it myself**. The measured value overrides the table.

One input to that ceiling is now sourced rather than assumed: a repository task slot is **not** one connection. Veeam opens up to **64 concurrent S3/BLOB operations per task slot**, and recommends staying under **6016 connections** against public cloud object storage — so the default 1 proxy × 4 tasks is 256 connections, not 4. Source: [Using a SOBR and Capacity Tier](https://www.veeam.com/blog/sobr-architecture-guide.html).

What remains an assumption is the 256 KB per-connection TCP window. The bandwidth-delay-product formula itself is standard networking; that constant is the modelled input to it, and it is marked as an assumption in the source.

---

## The PDF report

**Save as PDF** produces a standalone report, not a print of the page. Printing drops the app entirely — controls, tabs, intermediate panels — and emits only the report: a letterhead with the target region, timestamp and tool version; every setting that produced the numbers; every resource figure; any caveats that apply to the run (window not met, latency ceiling, RTT still an estimate); and the assumptions statement.

It comes from the browser's own print-to-PDF through an `@media print` stylesheet, which keeps the app a single dependency-free file. The saved file is named `SOBR-Vault-report_<region>_<date>.pdf`.

---

## How to run it

Clone the repository and open `index.html` in a browser. No build tools, no server, no dependencies to install:

```bash
git clone https://github.com/martinljor/veeamcalc
```

Google Fonts and Font Awesome are loaded from a CDN — without internet access the calculator still works, just without the custom typeface and icons.

## Project structure

```
/
└── index.html     ← entire app in a single file (no build tools required)
```

The version shown in the header comes from the `APP_VERSION` constant in `index.html`.
