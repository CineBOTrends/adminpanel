# CineBOTrends — Admin Panel (admin.cinebotrends.com)

Full dashboard (all drill-downs) + Decap CMS, served behind Cloudflare Access.

## Contents
- index.html, js/, css/, assets/, fonts/  -> the full dashboard app (reads /data)
- admin/                                   -> Decap CMS (login at /admin/)

## Data
`data/` is NOT in this repo. The collector's GitHub Action pushes the FULL data
tree here on each run. Cloudflare Pages then serves it.

## Login
admin/config.yml points at:
  repo:     CineBOTrends/dashboard   (editorial posts commit there)
  base_url: https://cbt-oauth.cinebotrends.workers.dev   (OAuth Worker)
Register the GitHub OAuth App + deploy the OAuth Worker for login to work.

## Lock down
Put Cloudflare Access in front of admin.cinebotrends.com (gate to your emails).
