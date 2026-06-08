# VDC Vault Storage Calculator

A storage estimator for **Veeam Backup & Replication** with VDC Vault. Complements the [official Veeam calculator](https://www.veeam.com/calculators/simple/vdc) with additional functionality.

## What does it calculate?

- **Performance Tier** and **Capacity Tier** separately
- **Full GFS retention** — daily, weekly, monthly, yearly
- **Immutability overhead** — with a detailed explanation of why it exists
- **Visual simulation** of restore points day by day, with active lock indicator

## How to deploy on GitHub Pages

1. Upload `index.html` to the root of your repository
2. In the repository: **Settings → Pages**
3. Under *Source*, select **Deploy from a branch**
4. Choose the `main` (or `master`) branch and the `/ (root)` folder
5. Save — within 1–2 minutes your URL will be `https://<your-username>.github.io/<repo>/`

## Project structure

```
/
└── index.html     ← entire app in a single file
└── README.md
```

No Node.js, npm, or build tools required. Pure HTML/CSS/JS.

## Supported parameters

| Parameter | Description |
|---|---|
| Source data (TB) | Total size of the data to be backed up |
| Daily change rate (%) | Percentage of data that changes each day |
| Compression (%) | Expected compression ratio |
| Block generation period | How often (in days) Veeam generates a new synthetic full |
| GFS retention | Daily / Weekly / Monthly / Yearly restore points |
| Immutability | Object Lock period per tier |
| Copy / Move policy | Whether the Capacity Tier is active |

## Notes

- Results are **estimates**. Actual storage may vary depending on deduplication, data type, and repository configuration.
- Immutability overhead formula: `compressed_full × (immutability_period ÷ block_gen_period)`
- Unofficial tool, not affiliated with Veeam Software.
