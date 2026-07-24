# Well-Connected People — nightly maintenance routine

This is the prompt for the recurring Attio maintenance task.

## What changed (2026-07-24)

Previously the job ran **only** for members of the "Well connected people" list.
It now runs for **everyone in the intro-request pipeline** — i.e. every person listed
as a `referrer` on any entry in the "Friendly outreach to do" list — plus anyone already
on the "Well connected people" list (so no one who was covered before falls out).

Because the count in step 6 lives on a `well_connected_people_1` **list entry**, each
processed connector is now **upserted into that list** (added if missing) so their count
has somewhere to be written. Net effect: the "Well connected people" list + its
"Well-Connected Stats" view becomes a computed leaderboard of every pipeline connector
ranked by how many people they connect you to, rather than a hand-curated subset.

Everything else (duplicate-awareness, add-only link maintenance, the count overwrite) is
unchanged.

---

## Routine prompt (paste this into the scheduled task)

```
Run tonight's "Well connected people" maintenance job (the recurring Attio task we set up). Use the Attio MCP tools — load via ToolSearch if not active: list-records-in-list, list-records, search-records, get-records-by-ids, update-record, update-list-entry-by-record-id, add-record-to-list.

GOAL: For every CONNECTOR in the intro-request pipeline — defined as every person listed as a referrer on any entry of the "Friendly outreach to do" list, PLUS anyone already on the "Well connected people" list — do the following: (A) find all people for whom that connector is a CONNECTOR or a REFERRER and add any NEW ones to that connector's "Connected to" multi-select on their Person record (add only, never remove); (B) ensure that connector is an entry on the "Well connected people" list (add them if missing), then set their list-entry "Number of connected to" count to the current size of their "Connected to" field. The job is DUPLICATE-AWARE: it must also catch links attached to duplicate copies of the same person.

KEY IDS:
- People object_id: 79ce8145-d114-4d25-85bd-99ceb30e0f8e
- "Well connected people" list slug: well_connected_people_1 (parent object = people); list-entry number attribute "number_of_connected_to" ("Number of connected to") — this is the count field, shown in the "Well-Connected Stats" view
- "Friendly outreach to do" list slug: friendly_outreach_to_do — entries have a multi-select record-reference attribute "referrer" pointing to people. This list IS the intro-request pipeline, and its distinct referrers are the connectors this job processes.
- Person attribute "connector": multi-select record-reference — person P's connector list holds people who are connectors TO P
- Person attribute "connected_to": multi-select record-reference — the link field being maintained

STEPS:
1. BUILD THE MEMBER SET (this is the widened step):
   a. Fetch ALL entries in friendly_outreach_to_do (paginate limit=50 + offset until has_more=false). For each entry, read its "referrer" multi-select and collect every referenced person record_id. These are the connectors in the intro-request pipeline.
   b. Fetch ALL entries in well_connected_people_1 (paginate the same way) and collect each entry's parent_record.record_id, so anyone already curated as well-connected stays covered even if they have no live pipeline entry right now.
   c. Union (a) + (b) and de-duplicate the record_ids. Call each resulting record a member M.
   d. For each member M, read M's name and email_addresses (get-records-by-ids, batched).
2. For each member M, build the set S of records representing the same person: search-records on "people" with query = M's full name; keep results whose full name matches M's (case-insensitive) OR that share at least one email address with M; always include M. S = M + its duplicates.
3. For EACH record R in S, run both reverse lookups:
   a. CONNECTOR: list-records object "people" filter {"attribute":"connector","op":"eq","value":{"object_id":"79ce8145-d114-4d25-85bd-99ceb30e0f8e","record_id":"<R>"}} (paginate). Collect matching record_ids.
   b. REFERRER: list-records-in-list on friendly_outreach_to_do filter {"attribute":"referrer","op":"eq","value":{"object_id":"79ce8145-d114-4d25-85bd-99ceb30e0f8e","record_id":"<R>"}} (paginate). Collect each entry's parent_record.record_id.
4. Union all collected record_ids across every R in S; exclude any record that is itself in S.
5. If the union is non-empty: update-record object "people", record_id=M, patch_multiselect_values=true, values={"connected_to":[{"target_object":"people","target_record_id":"<id>"}, ...]}. This preserves existing entries and adds only new ones. Always write to M (the record being processed), even when a match came from a duplicate.
6. LIST-ENTRY + COUNT STEP (do for EVERY member M, even if nothing was added):
   a. Ensure M is an entry on well_connected_people_1. If M does not already have an entry on that list, add it with add-record-to-list (list=well_connected_people_1, parent_object="people", record_id=M). If M already has an entry, do nothing here.
   b. Re-read M's record (get-records-by-ids) and count the number of values in its "connected_to" field.
   c. Set the list-entry count: update-list-entry-by-record-id with list="well_connected_people_1", parent_object="people", parent_record_id=M, entry_values={"number_of_connected_to": <count>}. This overwrites the number with the current total (it is a plain number, not additive).
7. Reply with a concise summary: total connectors processed and how many came from the pipeline vs. already on the list; per member — duplicates folded in, connector matches, referrer matches, newly-added connected_to count, whether they were newly added to the Well connected people list, and final "Number of connected to". Flag any members with duplicate records so they can be merged. Pure Attio data task: do NOT create PRs or touch any git repo.
```
