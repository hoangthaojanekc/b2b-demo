# b2b-demo — marketing ops + attribution portfolio build

## Goal
Simulate a B2B marketing stack for a fictional company and build lead-to-revenue
measurement in BigQuery. Company, product lines, personas, and channels are defined
in config/company.yml so the demo can be re-skinned for any B2B target
(SaaS, deep tech, healthcare, fintech) without touching pipeline code.

## Fictional company (default — edit config/company.yml to change)
- Name: Meridian Labs (fictional; no real company names or branding anywhere)
- Two product lines, each with its own buyer personas and campaigns
- Long sales cycle, buying committees, account-based motion

## Stack
- Site: static index.html on GitHub Pages, GTM + GA4, HubSpot tracking + embedded form
- Identity stitching: GA4 client_id (_ga cookie) → hidden HubSpot field `ga_client_id`
  → joins to GA4 BQ export `user_pseudo_id`
- CRM: Salesforce Developer Edition (campaigns, leads, contacts, accounts, opps, contact roles)
- Warehouse: BigQuery (raw_hubspot, raw_salesforce, raw_ga4, raw_syndicated datasets)
- Transform: dbt, staging → intermediate → marts
- Marts: W-shaped attribution (sourced + influenced), funnel conversion, pipeline

## Conventions
- Windows + VS Code; secrets in .env via python-dotenv, never committed (.env in .gitignore)
- All company, product, persona, and channel names read from config/company.yml — never hardcoded
- No PII in GA4; synthetic data only for volume
- Never write my personal email, phone, or name-based contact info into any file
  (code, configs, package metadata, README). Use placeholders.
- Work one phase at a time; stop and confirm with me before moving to the next step
