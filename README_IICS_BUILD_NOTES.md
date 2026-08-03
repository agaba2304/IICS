# CPD-Dayforce Integration — Build Notes

This repo tracks the IICS Explore export for the **CPD-Dayforce Integration** project (`p_dayforce_cpd_integration` orchestrator + supporting connectors/sub-processes), built against `Dayforce_CPD_Functional_Design_Build_Guide.docx` v1.0.

`IICSExport.zip` at the repo root is the re-importable package (see "Regenerating the zip" below). Everything under `Explore/` is the same content extracted to individual files so changes are reviewable in git diffs/PRs — **edit the extracted files, not the zip directly.**

## Naming history

Names have changed several times as this was built out; current names are what matters, but in case old names show up in a stale export or an old conversation, here's the lineage:

| Asset | Original (dummy) name | Interim name (post schema-fix) | **Current name** |
|---|---|---|---|
| CPD Service Connector | `cnctr_cpd_get` | `SvcConn_Get_Post_Cpd` | **`sc-api-cpd`** |
| Dayforce Service Connector | `cnctr-test-dayforce` | `SvcConn_get_post_dayforce` | **`sc-api-dayforce`** |
| CPD Connection | *(didn't exist)* | `Conn-get-post-cpd` | **`ac-sc-api-cpd`** |
| Dayforce Connection | *(didn't exist)* | `Conn-get-post-dayforce` | **`ac-sc-api-dayforce`** |
| Delta-query process | *(didn't exist)* | `GetChangedEmployees_Process` | **`p_GetChangedEmployees`** |
| Orchestrator | `Prcs_Dayforce_CPD` (empty stub) | *(same name)* | **`p_dayforce_cpd_integration`** |
| Per-employee body | *(didn't exist)* | `ProcessOneEmployee_Process` | **`p_ProcessOneEmployee`** |
| Employee detail lookup | *(didn't exist)* | `GetEmployeeDetail_Process` | **`p_GetEmployeeDetail`** |
| Person search | *(didn't exist)* | `SearchPerson_Process` | **`p_SearchPerson`** |
| New-hire create path | *(didn't exist)* | `CreatePersonAndContract_Process` | **`p_CreatePersonAndContract`** |
| Existing-employee update path | *(didn't exist)* | `UpdatePersonAndContract_Process` | **`p_UpdatePersonAndContract`** |
| VINCI ID writeback | *(didn't exist)* | `WriteVinciId_Process` | **`p_WriteVinciId`** |
| Add phone | *(didn't exist)* | `AddPhoneNumber_Process` | **`p_AddPhoneNumber`** |
| Update phone | *(didn't exist)* | `UpdatePhoneNumber_Process` | **`p_UpdatePhoneNumber`** |
| Delete phone | *(didn't exist)* | `DeletePhoneNumber_Process` | **`p_DeletePhoneNumber`** |
| Fault alert formatter | *(didn't exist)* | `ErrorAlert_Process` | **`p_ErrorAlert`** |

Every process in the project now carries the `p_` prefix (dropping the old `_Process` suffix, matching the convention already used for `p_GetChangedEmployees`/`p_dayforce_cpd_integration`), and every `<service>` step in every process file continues to reference the two **Connections** (`ac-sc-api-cpd`, `ac-sc-api-dayforce`) by name — never the raw Service Connectors — as established in the "Connectors and Connections" section below.

**Note on how this naming attempt differs from the previous one:** an earlier attempt to rename the Connections/Service Connectors by reimporting XML with a changed `displayName` over the *existing* live objects did not work — it either left the live objects untouched or created duplicates (see the reverted commit history on this branch). That attempt has been undone; the four connector/connection objects were then **deleted entirely from the org**, so this export is now a **fresh, first-ever import of all 16 objects into an empty project** — not a rename of anything pre-existing. That sidesteps the duplicate/no-op problem, but carries over the *other* confirmed risk from earlier in this build: **IICS assigns a brand-new GUID to every object on its first import, discarding whatever GUID is in the authored XML** — this was already proven true for processes (see "Process cross-references" below) and has never been directly tested for `AI_SERVICE_CONNECTOR`/`AI_CONNECTION` objects, since those were always created/renamed by hand in the UI in every prior round of this build, never imported fresh from a zip.

**Resolved**: exactly this happened. The first-ever import reassigned fresh GUIDs to all four objects, so every process's `<serviceGUID>` was pointing at the old (authored) GUIDs instead of what the org actually created. Fixed by substituting the real post-import GUIDs everywhere:

| Object | Authored GUID | Real (post-import) GUID |
|---|---|---|
| `sc-api-cpd` (Service Connector) | `aDMzhpLojjoke0pnhn1H4E` | `fpG58YCPVm0l9BZGk7R7ir` |
| `sc-api-dayforce` (Service Connector) | `kvKgEPCM9y7i86SJRF8qEK` | `3BiZs14UV6Jf9uH8m7hr2n` |
| `ac-sc-api-cpd` (Connection) | `0zkkeyRXHvpjfEh4Tgq5sM` | `3ek6aEwc6QOkqvy88jyFsM` |
| `ac-sc-api-dayforce` (Connection) | `jEzhmqLAzmBhdgzZWel1I2` | `6hySfDN0kMFjW58a4QsJiN` |

Confirmed via a real export of the org's now-live connectors/connections: the connector/connection *content* (every action, input, output, binding) was already byte-for-byte identical to what's in this repo — only the GUIDs (and the connection's internal reference to its wrapped connector's GUID) needed correcting. Every `<serviceGUID>` across all 12 processes, and the two Connection files' `businessConnector guid="..."` attributes, now point at the real GUIDs above.

## How this was built

The original repo only contained the orchestrator as an empty `Start -> End` process plus two connectors captured from live test calls (under their original dummy names, see table above), including several hardcoded secrets (a Dayforce password and long-lived Bearer JWTs, and a CPD Bearer token). This pass:

- Removed every hardcoded secret and replaced it with a `{curly-brace}` bind parameter (`access_token`, `cpd_token`, `df_username`, `df_password`) — **the exposed credentials were live at the time this repo was read and should be rotated regardless of this change.**
- Extended both connectors with every REST action listed in the design doc's API reference (section 3).
- Built the orchestrator (`p_dayforce_cpd_integration`) and 11 sub-processes implementing the design doc's build guide (section 4).

## Connectors and Connections

IICS's real object model, confirmed by actually publishing these in the org: **Service Connector** (the `AI_SERVICE_CONNECTOR` file — defines actions/operations) → **Connection** (`AI_CONNECTION` — wraps a Service Connector with an agent/auth binding) → **Process Service step** (references the Service Connector's action; the Connection is what makes that action executable by a Secure Agent). A raw Service Connector cannot be selected as a Service step's target until it is (a) schema-valid and (b) has at least one Connection built on top of it.

Getting the two Service Connectors to actually **publish** took three fix rounds, all confirmed by testing directly against this org (not guessed):
1. **Stray XML comments** (`<!-- -->` as the first child of `<types1:Entry>`) — removed. Every real IICS export checked had zero comments anywhere in the object body.
2. ~~`entireResponse="true"` output fields~~ — briefly suspected and reverted, then **confirmed valid** after finding a real published connector using it successfully. Left out of the connectors initially (output fields were left empty/minimal on the working connectors as republished) — **this turned out to be an actual bug, not just an omission**: several processes' XQuery expressions reference `$output.RawResponse` (`p_SearchPerson`, `p_CreatePersonAndContract`, `p_GetChangedEmployees`, `p_GetEmployeeDetail`) to reach nested/undeclared fields, but the connector actions they call (`SearchPersonByExtId`, `CreatePerson`, `GetChangedEmployees`, `GetEmployeeDetail`) had no `RawResponse` output field at all, so Process Designer correctly flagged those steps as invalid ("the variable output.RawResponse cannot be resolved"). Fixed by adding an `entireResponse="true" type="xml"` field named `RawResponse` to each of those four actions' `<output>` — every other action's output is untouched since nothing else references `$output.RawResponse`.
3. **`<input><field .../></input>` instead of `<input><parameter .../></input>`** — this was the actual bug. `<field>` is only correct inside `<output>`; `<input>` uses `<parameter name=".." type=".." required=".."/>`. Confirmed both by a real published reference connector and by the user successfully publishing both connectors after this fix.

Once published, the user created the Connections (`ac-sc-api-cpd`, `ac-sc-api-dayforce`, both `AI_CONNECTION` objects, both published, both bound to agent `hubdevinfoadm02`) and renamed/republished the Service Connectors as `sc-api-cpd` and `sc-api-dayforce`.

**Important, confirmed directly in Process Designer**: a `<service>` step's "Service Type" picker resolves by **Connection name**, not Service Connector name — confirmed by the user checking the dropdown after the connector-name version still showed "Service Type: Select" / empty Input Fields despite both connectors being published. All `<service>` steps across every process file reference the **Connection**:

| | Service Connector (defines actions) | Connection (what `<service>` steps reference) | Connection GUID (`<serviceGUID>`) |
|---|---|---|---|
| CPD | `sc-api-cpd` (GUID `fpG58YCPVm0l9BZGk7R7ir`) | `ac-sc-api-cpd` | `3ek6aEwc6QOkqvy88jyFsM` |
| Dayforce | `sc-api-dayforce` (GUID `3BiZs14UV6Jf9uH8m7hr2n`) | `ac-sc-api-dayforce` | `6hySfDN0kMFjW58a4QsJiN` |

(These are the GUIDs IICS assigned on the fresh import into the emptied project — see "Naming history" above for how they were confirmed and substituted in.)

So every `<serviceName>` is `ac-sc-api-cpd:ActionName` or `ac-sc-api-dayforce:ActionName` (not the `SvcConn_*` connector name), and every matching `<serviceGUID>` is the **Connection's** own GUID, not the connector's. The action names themselves (`SearchPersonByExtId`, `CreatePerson`, etc.) are unchanged — they're still defined on the Service Connector, just addressed indirectly through the Connection.

## Process cross-references: IICS reassigns GUIDs on import

After the Connection fix above, 9 of the 11 processes imported cleanly. The remaining two (`p_ProcessOneEmployee`, `p_dayforce_cpd_integration`) failed with a generic "Internal application provider error" — and those two are exactly the only files that call *other processes in this same package* via `<subflow>` (everything else only calls the two Connections, which already existed).

Confirmed by re-exporting the 9 successfully-imported processes from the org and diffing: **IICS assigns each newly-imported process a brand new `GUID`/`types1:GUID`, ignoring whatever GUID was declared in the imported XML.** e.g. `p_UpdatePhoneNumber` was authored with GUID `2MQBdKskt65bUvaVMnxm9g`; after import IICS assigned it `0ncoBf8fJhphLIRIv0RXOS`. Every `<subflowGUID>` reference pointing at the *authored* GUID of a sibling process therefore breaks the moment that sibling is actually imported, since its real GUID is different.

The diff also confirmed this is **the only thing that needed fixing** — content, steps, links, and logic were preserved byte-for-byte (modulo formatting/whitespace and bookkeeping fields like `ParentFlowIds`/timestamps), which is good independent confirmation that the schema itself (service calls, assignments, containers, event handling) was already correct.

Fixed by remapping every `<subflowGUID>` in `p_ProcessOneEmployee` and `p_dayforce_cpd_integration` to the real post-import GUIDs of the 9 already-imported processes:

| Process | Authored GUID | Real (post-import) GUID |
|---|---|---|
| p_GetChangedEmployees | `3oBo3euvAbRVfbijBuFPeO` | `82BXsUMh6wDfTfmUadXuw1` |
| p_GetEmployeeDetail | `KPffEx7EOwfjp7pn7kzZ2M` | `aDhurm6YPSRfpX6jx48p6H` |
| p_SearchPerson | `DB1JRZkPV3HvqKV1vfjqgb` | `7n7iMGoFQB6dEzRA4TyXyr` |
| p_CreatePersonAndContract | `ugYTHbESlUs3xY4PcfoFaW` | `7fxMqJIQj2eigrAD3AjsqS` |
| p_UpdatePersonAndContract | `TUaU8PTn1SEcDVyUTRKb6U` | `cPu3ROm1I27g3XxioV28Yq` |
| p_WriteVinciId | `zBI0ZwpxPB1sKMv6wHgpd1` | `a7jAK3mkjWNb0FDymjNpiw` |
| p_AddPhoneNumber | `AwxVtpAsbduNEHRbWUeFYb` | `9WIET5Qv4lVl2j23ISZyoe` |
| p_UpdatePhoneNumber | `2MQBdKskt65bUvaVMnxm9g` | `0ncoBf8fJhphLIRIv0RXOS` |
| p_DeletePhoneNumber | `MrEEevrr6RSALfwqq4ZHmt` | `jlM9u0QPKK9ltcTL4PIAGA` |
| p_ErrorAlert | `xpZXrmJaYOBu7TJpkrH1WV` | `1qQ4Y1EizO2lEjgLImYXaP` |

**Resolved**: `p_ProcessOneEmployee` imported successfully once its 9 references were corrected (real post-import GUID: `2jvNEMZkr0SeKWtoaSMfYh`, vs. the authored `VYIrQ7qaaXeojtBdb9dXHt`). `p_dayforce_cpd_integration`'s `<subflow>` call to it has been updated to that real GUID. All 11 processes should now import and resolve cleanly.

## Revision history on the process XML schema

**First pass** (superseded): hand-authored using BPEL/ActiveVOS terminology (`<invoke>`, `<invokeProcess>`, `<decision>`, `<forEach>`, `<scope>`/`<faultHandlers>`) inferred from general knowledge, because the only real example available at the time was the trivial `Start -> End` stub. **This failed to import correctly** — Process Designer showed every step as a blank, un-typed placeholder stuck in "Validation in progress" (screenshotted by the user after import).

**Second pass** (current): before rewriting, real populated IICS Process Designer 11 exports were located and fetched from several public repositories (search via GitHub code search + `raw.githubusercontent.com`, since `docs.informatica.com` blocks automated fetches). Cross-referencing multiple independent real `.PROCESS.xml` files (all generated by the same `Informatica Process Designer 11` this repo's stub already declared) confirmed the actual schema:

| Concept | Real element (confirmed) | Notes |
|---|---|---|
| Process inputs | `<input><parameter name=".." type=".." required=".." description=".."><options>...</options></parameter></input>` | Not `<parameterSet><parameter direction="in">` (that was the first-pass guess) |
| Process outputs | `<output><field name=".." type=".."><options>...</options></field></output>` | |
| Process working variables | `<tempFields><field .../></tempFields>` | Referenced as `temp.xxx` / `$temp.xxx` |
| Assignment/mapper step | `<assignment id=".."><operation source="formula\|field\|constant" to="target"><expression language="XQuery">...</expression></operation>...<link .../></assignment>` | The whole expression language is XQuery, not a generic "expression" DSL |
| Call a connector action | `<service id=".."><serviceName>ConnectorName:ActionName</serviceName><serviceGUID>ConnectorWrapperGUID</serviceGUID><serviceInput><parameter name="actionInputField" source="field" updatable="true">SOURCE</parameter></serviceInput><link .../></service>` | **No `serviceOutput` element** — an action's declared `<output><field>` entries become available flat as `output.<fieldName>` (or `output.<fieldName>[1]/path` for object/xml fields) to every step for the rest of that process, once the service call runs |
| Call a sub-process | `<subflow id=".."><subflowGUID>TargetProcessGUID</subflowGUID><subflowPath>TargetProcessName</subflowPath><runForEach>true\|false</runForEach><input><parameter name=".." source="field" updatable="true">SOURCE</parameter></input><outputDef/><link .../></subflow>` | Has its own outgoing `<link>` when used standalone; drops it when used inside an `<eventContainer>` (see below) |
| Loop over a list | `runForEach="true"` on a `<subflow>` call | **There is no separate "repeat/forEach" element.** Looping only exists as "call this one sub-process once per item in a list field." This is why a new sub-process, `p_ProcessOneEmployee`, was introduced to hold the whole per-employee body — the Orchestrator loops by calling it with `runForEach="true"`, since there was no way to loop an inline multi-step sequence directly |
| Decision / branch | `<container id=".." type="exclusive"><flow id="branchA">...</flow><flow id="branchB">...</flow><link targetId="branchA" type="containerLink"><condition source="formula"><function name="true"><arg name="field">{$temp.xxx}</arg></function></condition></link><link targetId="branchB" type="containerLink"><condition source="formula"><function name="false">...</function></condition></link><link targetId="nextNode"/></container>` | Branch `<flow>` blocks have no `<start>`/`<end>` of their own — their last activity links back to the container's own `id` with `type="containerLink"` to signal "branch done" |
| Fault handling (try/catch) | `<eventContainer id=".."><flow id="normal">...</flow><flow id="fault">...</flow><link targetId="normalOrFault" type="containerLink"/> (routing) <link targetId="nextNode"/> (exit) <events><catch faultField="faultInfo" interrupting="true"><link targetId="faultFlowId" type="containerLink"/></catch></events></eventContainer>` | Fault details become available as `output.faultInfo[1]/code`, `.../reason`, `.../detail` inside the catch's fault flow |

**Confidence levels**, being specific about what's now verified vs. still inferred:
- **High confidence** (directly confirmed across 3+ independent real exports): `<input>`/`<output>`/`<tempFields>`, `<assignment>`/`<operation>`/XQuery `<expression>`, `<service>`/`serviceName`/`serviceGUID`/`serviceInput`, `<subflow>` (standalone, `runForEach="false"`), `<container type="exclusive">` decision branching, the `<eventContainer>`/`<events><catch>` fault pattern (for a single watched activity).
- **Lower confidence, flagged inline in the affected files**:
  - `runForEach="true"` — no real example with it set to `true` was found in any public export searched; only the field's existence and default-`false` value are confirmed. `p_ProcessOneEmployee`'s three phone-loop `<subflow>` calls and the Orchestrator's `sf-processemployees` call use it, with each per-item field bound directly to the list field (e.g. `temp.tmp_PhonesToAdd/phoneTypeCode`) — check Process Designer's own loop-configuration UI after import and correct the binding if it differs.
  - Exact response shapes: no sample was available for the Dayforce delta/bulk employee list, the expanded `WorkContracts`/`Contacts` sub-trees, or the CPD phone-number list — all XQuery paths touching those are best-effort against the one flat single-employee sample and one no-phones person-search sample actually provided.

**`<eventContainer>` wrapping a multi-step flow: confirmed wrong, removed.** The one thing flagged above as "confirmed only for a single watched activity, not tested for a whole multi-step sequence" turned out to be exactly that: `p_ProcessOneEmployee` imported with its `<eventContainer>` step rendering as a blank, un-typed "name:" placeholder — the same symptom the very first schema pass produced everywhere, before the whole invoke/decision/forEach rewrite. Rather than guess a third fault-handling shape, the `<eventContainer>`/`<events><catch>` wrapper was removed entirely; `p_ProcessOneEmployee` is now a plain sequential flow (same pattern as every other file that already imports cleanly), and it imported successfully once that was done. **Fault handling (build guide 4.13) is an open item again** — add it back through Process Designer's own UI (most versions expose a per-step or wrap-in-scope "Add Exception/Fault Handler" action) rather than hand-authored XML. `p_ErrorAlert` (which formats the alert text) is unaffected and still importable/callable on its own; it's just not wired into anything's error path right now.

Before treating any of these processes as build-ready:
1. Open each `.PROCESS.xml` in IICS Process Designer (or re-import the zip into an Explore dev project) and confirm it now validates without the blank/un-typed step problem from the first pass.
2. Pay special attention to the `runForEach="true"` subflow calls and the `p_ProcessOneEmployee` fault container — these are the two areas flagged above as lower-confidence.
3. Re-save from Process Designer once validated so the file picks up your IICS version's canonical XML (visual layout coordinates, etc.), then re-export.

## Open items (carried over from the design doc, not introduced by this build)

| # | Item | Where it shows up |
|---|------|-------------------|
| 1 | CPD hostname is a placeholder (`{cpd-host}`) | `sc-api-cpd` — every new CPD action |
| 2 | Dayforce EmployeeProperties XRefCode for the VINCI ID field is unconfirmed | `sc-api-dayforce` `WriteVinciIdToDayforce` action, `p_WriteVinciId` |
| 3 | Dayforce org (LedgerCode) → CPD `ouId` cross-reference table doesn't exist yet | `p_CreatePersonAndContract` / `p_UpdatePersonAndContract` (`tmp_resolved_COR_ouId`) |
| 4 | Dayforce contact/phone type → CPD `phoneTypeCode` mapping is unconfirmed, and no sample of the expanded `Contacts.Items` shape was available | `p_ProcessOneEmployee` `dp-comparephones` step |
| 5 | CPD authentication mechanism is unconfirmed (no CPD equivalent of Dayforce's "Get token" action exists yet) | `sc-api-cpd` — `in_CpdToken` is a process input (secure parameter) until this is resolved |
| 6 | Create-path `contractId` source is unconfirmed — the design doc's Create/Update Contract call is a single PATCH to an existing `{contractId}`, which presupposes CPD already issued one on Create Person | `p_CreatePersonAndContract` |
| 7 | Search Person match criteria — this build uses **Ext Id / VINCI ID** per project decision (not name+DOB). Known limitation carried from the design doc: this can't by itself distinguish a genuine new hire from a prior-run assignment that hasn't propagated yet | `p_SearchPerson`, `sc-api-cpd` `SearchPersonByExtId` action |
| 8 | Error-alert recipients and format are unconfirmed, **and no Email/notification connector exists yet in this project** to reference | `p_ErrorAlert` only formats the alert text (`out_Subject`/`out_Body`); it does not send anything yet — add a `<service>` step once an Email connector exists |
| 9 | `LastRunTimestamp` persistence across scheduled runs (the Orchestrator takes it as an input/emits `out_CurrentRunTimestamp` as an output, but the actual storage — a cache table, a file, a custom parameter — isn't wired up) | `p_dayforce_cpd_integration`, `p_GetChangedEmployees` |
| 10 | The Schedule object (build guide 4.16) is an IICS Administrator config, not part of this Explore export — must be created separately once the process is deployed | N/A |
| 11 | Fault handling (build guide 4.13) is not wired up — the `<eventContainer>` wrap around `p_ProcessOneEmployee`'s body failed to import (unrecognized element), so it was removed. `p_ErrorAlert` still exists and formats an alert, but nothing calls it yet | `p_ProcessOneEmployee` — add via Process Designer's own Exception/Fault Handler UI once the process is otherwise validated |

## Process inventory

| Process | Role |
|---|---|
| `p_dayforce_cpd_integration` | Orchestrator: get changed employees, get a run-scoped Dayforce token, loop `p_ProcessOneEmployee` over each XRefCode |
| `p_GetChangedEmployees` | Dayforce delta query (build guide 4.4) |
| `p_ProcessOneEmployee` | Per-employee body: detail + search + create-or-update decision + phone reconciliation. Looped by the Orchestrator via `runForEach`. Fault handling not yet wired (see Open Item 11) |
| `p_GetEmployeeDetail` | Dayforce employee detail, expanded (build guide 4.5) |
| `p_SearchPerson` | CPD search by Ext Id/VINCI ID (build guide 4.6) |
| `p_CreatePersonAndContract` | New-hire path (build guide 4.8) |
| `p_UpdatePersonAndContract` | Existing-employee path (build guide 4.9) |
| `p_WriteVinciId` | VINCI ID writeback, create path only (build guide 4.10) |
| `p_AddPhoneNumber` / `p_UpdatePhoneNumber` / `p_DeletePhoneNumber` | Phone reconciliation actions (build guide 4.12), looped by `p_ProcessOneEmployee` |
| `p_ErrorAlert` | Formats a fault notification (build guide 4.13-4.14) — sending is not yet wired, see Open Item 8 |

## Regenerating the zip

After editing files under `Explore/`, rebuild `IICSExport.zip` and refresh `exportPackage.chksum` (plain uppercase-hex SHA-256 per file, same format Informatica's export produces):

```bash
cd /home/user/IICS
rm -f IICSExport.zip
zip -r IICSExport.zip Explore exportMetadata.v2.json ContentsofExportPackage_IICSExport.csv exportPackage.chksum
```

Recompute `exportPackage.chksum` first if any tracked file changed (plain SHA-256 hex, uppercased, of each file's bytes, keyed by its repo-relative path with spaces escaped as `\ `).
