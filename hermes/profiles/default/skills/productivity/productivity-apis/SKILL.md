---
name: productivity-apis
description: "Automate productivity SaaS and location APIs: Airtable, Google Workspace, Notion, and maps/geocoding/routing workflows."
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [productivity, airtable, google-workspace, notion, maps, api, automation]
    related_skills: []
---

# Productivity APIs

## Overview

Use this umbrella for API/CLI automation of productivity and information services: Airtable records/databases, Google Workspace mail/calendar/drive/docs/sheets, Notion pages/databases/blocks, and maps/geocoding/routing/timezone lookups. The common pattern is credential discovery, schema/readback first, scoped mutation, and verification.

## When to Use

- Query or update Airtable bases, tables, fields, and records.
- Work with Gmail, Calendar, Drive, Docs, or Sheets via Google Workspace tooling.
- Search/create/update Notion pages, databases, markdown, or blocks.
- Geocode addresses, reverse geocode coordinates, find POIs, routes, or timezones.

## Universal Workflow

1. Identify the service and exact object: base/table, mailbox/calendar/file, page/database, or location/route.
2. Check credentials and token scope without exposing secrets.
3. Read schema/current state before mutation.
4. Perform the smallest scoped request.
5. Verify by reading the changed object or returning a stable ID/URL/result.

## Airtable Notes

- List bases/tables/schema before constructing record payloads.
- Match request body shapes to Airtable field types.
- Use filters/upserts carefully and verify record IDs.

## Google Workspace Notes

- First-time OAuth setup is multi-step; reuse existing config when present.
- Ask for the concrete Workspace surface when ambiguous: Gmail, Calendar, Drive, Docs, or Sheets.
- For email/calendar side effects, preserve exact user wording/times and verify send/create IDs.

## Notion Notes

- Ensure the integration has access to the target page/database.
- Prefer the CLI path when installed; use raw API only when needed.
- Respect block type constraints and paginate search/database queries.

## Maps Notes

- Use geocoding/reverse-geocoding for place resolution, OSRM-style routing for travel distances, and timezone lookup for scheduling context.
- Report coordinates, provider assumptions, and route mode.

## Pitfalls

- Mutating before reading schema/current state.
- Confusing display names with stable IDs.
- Losing pagination on large result sets.
- Exposing OAuth tokens/API keys in logs.

## Verification Checklist

- [ ] Credentials/scopes checked.
- [ ] Target object/schema read before mutation.
- [ ] Pagination handled where relevant.
- [ ] Mutations verified by readback.
- [ ] Stable IDs/URLs/coordinates included in final output.
