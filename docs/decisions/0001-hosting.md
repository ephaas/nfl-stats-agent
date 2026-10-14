# 0001. Hosting

Copied from `PLAN.md` Phase 0 before any measuring.

## Pre-registered criteria

- **A passes** if all three hold:
  - no out-of-memory restarts, with peak memory under 450 MB
  - warm p95 of 5 seconds or less, with zero 10-second timeouts
  - cold start of 90 seconds or less
  If cold start fails because of image size, try a trimmed image built by us.
- **B passes** if the subset fits in 0.9 GB or less, warm p95 is 2 seconds or less, and cold start is 5 seconds or less. Known downside: production would run a Postgres SQL tool instead of nfl-mcp, so production would no longer be the evaluated system.
- **C (Azure for Students) is a write-up only.** No sign-up and no deploy; you decide after seeing the numbers.
Cost of nfl-mcp's template (0.5 vCPU, 1 GiB, 24/7), at the [official prices](https://azure.microsoft.com/en-us/pricing/details/container-apps/) with the [idle-billing rules](https://learn.microsoft.com/en-us/azure/container-apps/billing):
  - **Expected (mostly idle):** about $10.20 a month, or about 9.8 months of the $100 credit ($10–12 a month realistically, 8–10 months).
  - **Worst case (billed active 24/7):** about $34 a month, or about 2.9 months.
  - **Other costs:** logs have 5 GB a month free, so they cost nothing here.
  - **Student status ends around June 2027**, so the demo likely stops then whatever credit is left.
  - **Never upgrade to pay-as-you-go; any card prompt means stop.**
  - **Open risk:** a $0.10/hour "Environment Management Hour" meter (about $73 a month). Check in the no-sign-in pricing calculator that it doesn't apply.
