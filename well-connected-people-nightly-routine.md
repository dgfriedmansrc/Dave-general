# Connected-To maintenance — nightly routine

This is the prompt for the recurring Attio maintenance task.

## What it's for

When you open any person's record, you want the **`Connected To`** field to show
everyone that person is a Connector for. You create a target as a People record and
attach someone you know as their **Connector**; this job keeps the reverse view
(`Connected To`) on that connector up to date.

## What changed (2026-07-24)

Previously the job maintained `Connected To` **only** for the people on the
"Well connected people" list. It now maintains `Connected To` for **every connector in
the intro-request pipeline** — i.e. everyone attached as a `Connector` on a pipeline
target's People record, or listed as a `referrer` on a "Friendly outreach to do" entry.

The count field ("Number of connected to") and Well-Connected-People list membership are
left exactly as they were: the count is still refreshed for people already on that list,
and **no one new is added to it**. Duplicate-awareness and add-only link maintenance are
unchanged.

---

## Routine prompt (paste this into the scheduled task)

```
Run tonight's Connected-To maintenance job (the recurring Attio task we set up). Use the Attio MCP tools — load via ToolSearch if not active: list-records-in-list, list-records, search-records, get-records-by-ids, update-record, update-list-entry-by-record-id.

GOAL: Keep the "Connected to" field current for EVERY connector in the intro-request pipeline. A connector is anyone attached to a pipeline target as a Connector — either listed in a target People record's "connector" attribute, or listed as the "referrer" on a "Friendly outreach to do" list entry. For each such connector M: (A) find all people for whom M is a CONNECTOR or a REFERRER and add any NEW ones to M's "Connected to" multi-select on their Person record (add only, never remove). Then (B), ONLY IF M is already an entry on the "Well connected people" list, set M's list-entry "Number of connected to" to the current size of M's "Connected to" field — do NOT add anyone new to that list. The job is DUPLICATE-AWARE: it must also catch links attached to duplicate copies of the same person.

KEY IDS:
- People object_id: 79ce8145-d114-4d25-85bd-99ceb30e0f8e
- "Friendly outreach to do" list slug: friendly_outreach_to_do — this IS the intro-request pipeline. Entries have a multi-select record-reference attribute "referrer" pointing to people, and each entry's parent_record is the target Person.
- Person attribute "connector": multi-select record-reference — target person P's "connector" list holds the people who are connectors TO P.
- Person attribute "connected_to": multi-select record-reference — the reverse-view field being maintained (lives on the connector).
- "Well connected people" list slug: well_connected_people_1 (parent object = people); list-entry number attribute "number_of_connected_to" ("Number of connected to") is the count field shown in the "Well-Connected Stats" view. This list stays hand-curated — the job only refreshes counts for people already on it.

STEPS:
1. BUILD THE CONNECTOR SET (the widened step). Collect the set of connectors M to maintain:
   a. Fetch ALL entries in friendly_outreach_to_do (paginate limit=50 + offset until has_more=false). For each entry, collect (i) every person record_id in its "referrer" attribute, and (ii) the entry's parent_record.record_id (the target).
   b. Batch get-records-by-ids (object "people") on all target record_ids from 1a-ii, read each target's "connector" attribute, and collect every person record_id it references.
   c. Also fetch ALL entries in well_connected_people_1 (paginate) and collect each entry's parent_record.record_id — so anyone already curated as well-connected is still maintained. Remember this subset: it is the ONLY set eligible for the count step (6).
   d. Connector set = union of 1a-i (referrers) + 1b (target connectors) + 1c (existing list members), de-duplicated. Each is a member M.
   e. For each member M, read M's name and email_addresses (get-records-by-ids, batched).
2. For each member M, build the set S of records representing the same person: search-records on "people" with query = M's full name; keep results whose full name matches M's (case-insensitive) OR that share at least one email address with M; always include M. S = M + its duplicates.
3. For EACH record R in S, run both reverse lookups:
   a. CONNECTOR: list-records object "people" filter {"attribute":"connector","op":"eq","value":{"object_id":"79ce8145-d114-4d25-85bd-99ceb30e0f8e","record_id":"<R>"}} (paginate). Collect matching record_ids.
   b. REFERRER: list-records-in-list on friendly_outreach_to_do filter {"attribute":"referrer","op":"eq","value":{"object_id":"79ce8145-d114-4d25-85bd-99ceb30e0f8e","record_id":"<R>"}} (paginate). Collect each entry's parent_record.record_id.
4. Union all collected record_ids across every R in S; exclude any record that is itself in S.
5. If the union is non-empty: update-record object "people", record_id=M, patch_multiselect_values=true, values={"connected_to":[{"target_object":"people","target_record_id":"<id>"}, ...]}. This preserves existing entries and adds only new ones. Always write to M (the record being processed), even when a match came from a duplicate.
6. COUNT STEP — ONLY for members M that are already on the well_connected_people_1 list (the subset from 1c). Skip everyone else; do not add them to the list. For each eligible M: re-read M's record (get-records-by-ids), count the number of values in its "connected_to" field, then update-list-entry-by-record-id with list="well_connected_people_1", parent_object="people", parent_record_id=M, entry_values={"number_of_connected_to": <count>}. This overwrites the number with the current total (it is a plain number, not additive).
7. Reply with a concise summary: total connectors processed, broken down by source (pipeline referrer / target connector attribute / already on the Well connected people list); per member — duplicates folded in, connector matches, referrer matches, and newly-added "Connected to" count; and for the Well-connected-people members, their refreshed "Number of connected to". Flag any members with duplicate records so they can be merged. Pure Attio data task: do NOT create PRs or touch any git repo.
```
