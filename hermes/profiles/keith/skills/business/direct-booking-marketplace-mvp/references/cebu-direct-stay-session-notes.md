# Cebu Direct-Stay Session Notes

## User context

Keith is an infrastructure engineer/DBA exploring side-income web projects. He hosts five Airbnb properties in Cebu, Philippines:

- all on Cebu island
- four studio condo units
- one 1-bedroom condo
- generally close to Cebu City core / accessible locations
- one unit is about 5 minutes from Mactan-Cebu International Airport

Goal: increase occupancy and income outside Airbnb using AI-assisted web building, VPS/Docker hosting, direct inquiries, and eventually a local accommodation directory.

## Market/strategy observations from session

- Cebu short-stay supply appears competitive and growing.
- Direct booking should not rely only on a pretty homepage; build SEO landing pages around specific intent.
- Good traffic angles:
  - `mactan airport accommodation`
  - `condo near mactan airport`
  - `cebu condo daily weekly monthly`
  - `cebu staycation condo`
  - `cebu monthly stay`
  - `lapu-lapu transient`
  - `cebu city condo daily rental`
- Facebook groups can provide early lead discovery, but public scraping is limited. Best workflow is manual group participation + fast professional replies.

## Suggested initial pages

```text
/
/properties/
/properties/<listing-slug>/
/mactan-airport-accommodation/
/cebu-city-staycation-condo/
/cebu-condo-for-rent-daily-weekly-monthly/
/cebu-monthly-stay/
/list-your-property/
/contact/
```

## Direct-booking funnel

Recommended first version:

1. Guest lands from Facebook/Google/social.
2. Listing or SEO page shows clear property value and location.
3. Guest clicks floating `Ask Host` / `Check Availability` bubble.
4. Chat-style inquiry form collects:
   - check-in/check-out
   - guest count
   - name
   - email/phone/Messenger/WhatsApp
   - message
5. Inquiry is saved and sent to host + admin.
6. Host confirms availability manually.
7. Payment handled manually at first, likely GCash/bank transfer/payment proof upload.

## Embedded chat decision

For a multi-host Cebu directory, do **not** require every host to set up Facebook Messenger Customer Chat Plugin.

Reasons:

- each host would need a Facebook Page
- Meta setup/domain configuration is brittle
- personal profiles are messy for embedded messaging
- too hard for random hosts
- multi-host routing is awkward

Use embedded inquiry form first. It looks like chat but preserves lead data and avoids full marketplace messaging complexity.

## Potential monetization

Do not charge hosts before proving lead value.

Phases:

1. Free early listings to build inventory.
2. Featured placements after traffic starts.
3. Listing setup/copywriting/photo/SEO service.
4. Host mini-sites or direct-booking pages.
5. Optional paid qualified leads only when tracking and trust are mature.

Example early pricing ideas discussed:

- Featured listing: ₱299–₱999/month
- Homepage/area feature: ₱1,000–₱1,500/month
- One-time listing setup or host service packages later

## Trust and legal notes

- Marketplace/direct-contact directory needs manual host approval, verification cues, report buttons, and disclaimers.
- Guests transact directly with hosts, but reputation risk still lands on the directory.
- Do not imply that Airbnb allowing hosting means there are no tax/legal obligations. Encourage local verification when scaling.

## Style note for future sessions

Keith likes direct, no-BS guidance and wants pushback when assumptions are weak. Avoid hype around “marketplace” without calling out chicken-and-egg, trust, traffic, and monetization risks.
