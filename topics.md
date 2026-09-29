# Topics backlog

Real problems from my project work. The daily routine writes up the first unchecked item, anonymized, then checks it off. I add new items as they come up.

- [ ] **ArcGIS silently drops attributes with the wrong field names.** A CRM-to-AGOL backfill of ~34,000 records "succeeded" with most attributes blank, because the code used CRM-style keys that didn't match the layer's real field names. The fix: validate payload keys against the layer schema before calling edit_features.
- [ ] **Never cache OBJECTIDs.** Republishing a layer can reassign OBJECTIDs. Store a stable external key (the CRM record ID) and look up the OBJECTID at write time.
- [ ] **Geocoding fallbacks that don't lie.** Handle military APO/FPO addresses (AA/AE/AP), fall back to ZIP centroids, and write null geometry instead of 0,0 when there's no address, so nothing lands on "Null Island."
- [ ] **Reuse coordinates in a migration with an address hash.** Hash normalized addresses to match records against an old layer and reuse existing coordinates. About 90% of records needed no re-geocoding, which saved credits and time.
- [ ] **The stale Lambda test event that re-ran a two-week-old delta.** A saved console test event pointed at an old staging file. One manual run re-added ~3,300 out-of-scope records and overwrote ~4,800 good ones with stale data. Recovery scripts, plus the rule: only trigger the step that builds a fresh delta.
- [ ] **Scope filters belong in every query path.** The backfill filtered to active donors, but the nightly incremental query didn't, so staff and volunteers leaked in. Put scope rules in one shared function.
- [ ] **The arcgis Python package is too big for a Lambda zip.** Why I moved the ArcGIS write step to a container image in ECR, and how the two Lambdas chain asynchronously through an S3 delta file.
- [ ] **Cron in AWS: EventBridge Scheduler, not Rules.** In the current console, Rules only handles event-pattern triggers. Also covers time zones and daylight saving time for a 2 a.m. Pacific nightly job.
- [ ] **Salesforce to ArcGIS with the OAuth Client Credentials flow.** A read-only integration user, a permission set limited to the fields the sync needs, and why I chose Parameter Store over Secrets Manager for credentials.
- [ ] **A Survey123 calculated field that silently dropped selections.** A concat() field for a multi-select repair checklist lost some checked items. The fix: rebuild the list from the individual boolean fields downstream.
- [ ] **Make.com formatting tricks for field-data emails.** Stripping leading commas from multi-select values, ifempty() fallbacks for fields that aren't always populated, and turning coded values into plain English.
- [ ] **Experience Builder's 10-Arcade-expressions-per-page cap.** With 16 image widgets on one page, I switched to plain attribute binding plus "View for empty selection" so each image reverts to a default when nothing is selected.
- [ ] **Letting members edit their own profiles without database access.** A Survey123 "profile updater" loads the member's record on login, and an hourly AGOL notebook syncs changes (including photos) into the main layer.
- [ ] **Protecting donor privacy for outside analysts.** A weekly job moves donor points to ZIP centroids in a view layer, so an external team can build visualizations without seeing addresses.
- [ ] **Screening nonprofit leads from IRS bulk data.** A funnel that starts from the IRS Business Master File and Form 990 data: revenue band, sector (NTEE) filters, and multi-chapter structure before any manual research.
- [ ] **A Google Sheet as the editable source for a Lambda pipeline.** The client edits the sheet (a Status column controls what runs), the Lambda reads it through a read-only service account, and a static list in code is the fallback if the sheet is unreachable.
- [ ] **Lowercase globalid/objectid fields breaking joins.** Hosted layers created outside the usual path had lowercase system field names. How that broke scripts and dashboard data expressions that join distribution and follow-up records by barcode.
- [ ] **Fixing ArcGIS Hub sitemap XML.** Unescaped ampersands, messy slugs, and duplicate pages that kept search engines from indexing a community mapping site.
- [ ] **Open data portals aren't all the same.** California's health data portal runs CKAN, not Socrata, and some official APIs require auth. Plan integration approaches during discovery.
