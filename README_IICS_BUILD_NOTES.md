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

### A REST action's Body field: literal JSON braces must be doubled (`{{`/`}}`), single braces are for field substitution

This took several rounds to pin down correctly, including one confirmed-wrong intermediate conclusion (corrected below) — worth reading in full if this breaks again.

1. The original hand-authored body — literal JSON starting with a bare single `{` — silently lost its outer braces when Process Designer's "Expression Editor for Body" (Type: `xQuery`) displayed it (not a hard error, just corrupted content).
2. Doubling *only the outer* braces (`{{`/`}}`) while leaving nested object braces single produced hard parse errors (`XPST0003: Unexpected token "{"...`, then `XPST0003: expected "}", found ":"`) — because it was inconsistent (some structural braces doubled, most not), not because doubling itself is wrong.
3. **This led to an intermediate wrong conclusion**: that literal braces must stay **single**, with dynamic values as `"{ $ParamName }"`. That version passed the Editor's validation with no red error — but that only proves it's syntactically parseable, not that it produces correct output.
4. **What actually happens at runtime, confirmed by inspecting the real resolved `<rest:payload>` IICS sent** (visible via the connector's own request trace): a template written with **single** structural braces gets sent with those braces **collapsed/stripped**, corrupting the JSON. A template written with **double** structural braces (`{{`/`}}` at every JSON object/array-of-object boundary) gets sent with each pair correctly **collapsed down to one literal brace**, producing valid JSON. This was proven conclusively when a fully-doubled body was sent, the resolved trace showed clean single-braced valid JSON, and CPD responded with a real business-logic 422 (not a syntax error) — meaning the request was well-formed JSON that reached the API.

So the actual working pattern: **every JSON structural brace must be written doubled** (`{{`/`}}`) — object literals, array-of-object literals, all of them, at every nesting level — while a **single**-brace enclosed expression, `{ $ParamName }`, is left alone for field substitution (referencing an action's own input parameter as a genuine XQuery variable, confirmed via Process Designer's "Fields → Input Parameters" panel, which inserts that exact syntax when you click a field — plain `{ParamName}` without `$` is never evaluated, it stays literal text). This is different from the `{ParamName}` (no `$`, single brace, no doubling) convention used in URLs and headers, which is plain string substitution, not XQuery.

Applied (double-braced) to every JSON-bodied action in `sc-api-cpd`: `CreatePerson`, `CreateOrUpdateContract` (both directly confirmed against the real API), plus `UpdatePerson`, `AddPhoneNumber`, `UpdatePhoneNumber` (same convention applied proactively, not yet individually retested since the correction — check these the same way if they error).

**Note on syncing with live edits**: the user tested fixes directly in Process Designer's UI in parallel with this repo (adding `nogen`/`testWith` attributes via the "Test" panel, and a scratch `test` action pasting in a raw curl example — that action's `url` is literally the curl command text, not a real URL, so it will never work if invoked; left in place untouched rather than deleted, since it doesn't interfere with anything). A live-edited export was used as the base for one round of fixes specifically to preserve those `testWith` values, which also caught two regressions worth remembering: `CreatePerson`'s body reverting to inconsistent brace-doubling, and `SearchPersonByExtId`'s `extSystemId` being changed to `1` instead of the design-doc-confirmed Dayforce value `11`.

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

**Resolved (first time)**: `p_ProcessOneEmployee` imported successfully once its 9 references were corrected (real post-import GUID: `2jvNEMZkr0SeKWtoaSMfYh`, vs. the authored `VYIrQ7qaaXeojtBdb9dXHt`). `p_dayforce_cpd_integration`'s `<subflow>` call to it was updated to that real GUID, and all 11 processes imported and resolved cleanly.

**Recurred after the project was emptied and rebuilt from scratch** (see "Naming history" above): since the whole `CPD-Dayforce Integration` project was deleted and every object reimported fresh under its `p_`-prefixed name, IICS assigned a **new** set of GUIDs to all 10 sub-processes — the old post-import GUIDs in the table above are now stale. The exact same 2 processes (`p_ProcessOneEmployee`, `p_dayforce_cpd_integration`) failed again with the same "Internal application provider error" for the same reason. Fixed the same way, using a fresh export of the 10 successfully-imported processes to get the new real GUIDs:

| Process | Previous real GUID (now stale) | Current real GUID |
|---|---|---|
| p_GetChangedEmployees | `82BXsUMh6wDfTfmUadXuw1` | `5GazV17lbwwifrMRgR2yGB` |
| p_GetEmployeeDetail | `aDhurm6YPSRfpX6jx48p6H` | `265GBjYvi46gdOJB8fJJM3` |
| p_SearchPerson | `7n7iMGoFQB6dEzRA4TyXyr` | `iUSMDrm4KKQfyFo32SwBSr` |
| p_CreatePersonAndContract | `7fxMqJIQj2eigrAD3AjsqS` | `8Z9az8QKChed7X499q1dm7` |
| p_UpdatePersonAndContract | `cPu3ROm1I27g3XxioV28Yq` | `3v42BhrdaWnjtVpmJkaTK7` |
| p_WriteVinciId | `a7jAK3mkjWNb0FDymjNpiw` | `0mPVVn6Wjuai6YUf2dDOE4` |
| p_AddPhoneNumber | `9WIET5Qv4lVl2j23ISZyoe` | `l9dSmhTuTIffT30huW5QsB` |
| p_UpdatePhoneNumber | `0ncoBf8fJhphLIRIv0RXOS` | `06iBdoGBgpwd5YRixGqFEb` |
| p_DeletePhoneNumber | `jlM9u0QPKK9ltcTL4PIAGA` | `4z444EUFompiGRMGDU2iwt` |
| p_ErrorAlert | `1qQ4Y1EizO2lEjgLImYXaP` | `66EdFvuO3RqfR9bDxNDBI6` |

All 8 sibling `<subflowGUID>` references in `p_ProcessOneEmployee` have been updated to these current GUIDs, and `p_dayforce_cpd_integration`'s call to `p_GetChangedEmployees` has been updated too.

**Resolved (second time)**: `p_ProcessOneEmployee` imported successfully with the corrected references; its new real GUID is `aw0FVD6pWhobtQh5uL0Ou8` (vs. the now-doubly-stale `2jvNEMZkr0SeKWtoaSMfYh` from the first resolution, and the originally-authored `VYIrQ7qaaXeojtBdb9dXHt`). `p_dayforce_cpd_integration`'s `<subflow>` call to it has been updated to that GUID. All 12 processes should now import and resolve cleanly.

**Takeaway for any future full project rebuild**: every time this project is deleted and reimported from scratch, expect this exact two-round dance again — the 10 leaf processes import fine and get fresh GUIDs, then `p_ProcessOneEmployee` needs those GUIDs substituted in before it will import, then `p_dayforce_cpd_integration` needs `p_ProcessOneEmployee`'s own fresh GUID substituted in before *it* will import. A partial re-import of just the processes (not the connectors/connections) does not trigger this, since the Connections already exist and keep their GUIDs.

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
| 1 | ~~CPD hostname is a placeholder (`{cpd-host}`)~~ — **resolved**: confirmed the real test host is `api-tst.vinci-construction.net` (same one the original "Get Persons"/"Get purchase orders" dummy actions already used), and while fixing this found the 7 newly-built actions were also missing the `/hr` path prefix entirely (`{cpd-host}/v1/persons` instead of `api-tst.vinci-construction.net/hr/v1/persons`) — an actual bug. Fixed all 7 URLs (`SearchPersonByExtId`, `CreatePerson`, `UpdatePerson`, `CreateOrUpdateContract`, `AddPhoneNumber`, `UpdatePhoneNumber`, `DeletePhoneNumber`) to `https://api-tst.vinci-construction.net/hr/v1/persons...`. Only directly confirmed for `SearchPersonByExtId` — test the other 6 and report back if any needs a different path. | `sc-api-cpd` — every CPD action |
| 1b | ~~`SearchPersonByExtId` query parameter name was wrong (`extId`)~~ — **resolved**: after the `/hr` path fix, calls still failed with a sequence of validation errors (`"Both external identifiers and extSystem must be provided or none of them"`, then `"At least one discriminant filter must be filled"` after a wrong guess at `extSystem`). Root cause confirmed from the CPD API's own documented curl example for `Get Persons`: the query parameter is **`externalIdentifiers`** (comma-separated list of ext-id values), not `extId` — `extSystemId` (with "Id") was correct all along. Fixed to `?externalIdentifiers={EmployeeNumber}&extSystemId=11`. | `sc-api-cpd` `SearchPersonByExtId` action |
| 2 | Dayforce EmployeeProperties XRefCode for the VINCI ID field is unconfirmed | `sc-api-dayforce` `WriteVinciIdToDayforce` action, `p_WriteVinciId` |
| 3 | Dayforce org (LedgerCode) → CPD `ouId` cross-reference table doesn't exist yet | `p_CreatePersonAndContract` / `p_UpdatePersonAndContract` (`tmp_resolved_COR_ouId`) |
| 4 | Dayforce contact/phone type → CPD `phoneTypeCode` mapping is unconfirmed, and no sample of the expanded `Contacts.Items` shape was available | `p_ProcessOneEmployee` `dp-comparephones` step |
| 5 | CPD authentication mechanism is unconfirmed (no CPD equivalent of Dayforce's "Get token" action exists yet) | `sc-api-cpd` — `in_CpdToken` is a process input (secure parameter) until this is resolved |
| 6 | ~~Create-path `contractId` source is unconfirmed~~ — **partially resolved, and the design doc's assumption was wrong, not just unconfirmed**: a real 400 (`"The Contracts field is required"`) proved CPD's Create Person endpoint requires the initial contract embedded directly in the same request, not created via a separate follow-up call. `CreatePerson`'s input/body now include the contract fields, and `p_CreatePersonAndContract` no longer calls `CreateOrUpdateContract` on the create path (that action is now update-only). Still open: CPD's Create Person *response* schema (only the request schema is confirmed) — `out_NewContractId` still guesses the contract id comes back at `RawResponse//contractId`; confirm against a live successful response. Also fixed in passing: `salutationCode`/`contractTypeCode` must be integers (were strings), `directoryDisplayMode` should be `directoryAvailability` on write, and several body-field syntax bugs (see the "Body field is XQuery, not text" note below). | `sc-api-cpd` `CreatePerson`/`CreateOrUpdateContract` actions, `p_CreatePersonAndContract` |
| 6b | Several CPD Create Person/Contract schema fields have no confirmed Dayforce mapping. Since sent as literal doc-example values (not real data) to test the full schema shape, four have already failed real CPD validation and been removed: `hrInformation.professionalPhoneNumbers` (`"unknown PhoneTypeCode value : 22"`), `hrInformation.lineManager`/`operationalManager`/`administrativeManager` (`"not a valid luhn key vinci id"` — `01030406` etc. aren't real VINCI IDs), top-level `auditTrail.creationUser.userId` (`"Unknown VINCI ID. Value : '01234567'"`), and `contracts[].organizationalUnitAssignment.service.serviceId` (`"...COR items (ServiceIds: [string]...) are not found in COR database"` — `"string"` isn't a real service id, and CPD validates it against a live org-reference lookup). Still present as literal placeholders, not yet failed but not real data either: `externalStaffCompanyName`, `hrInformation.professionCode`/`staffCategoryCode`/`employeePayrollSystemCode`, most of `extraInformation` (`managerErpId` in particular looks like the same fake-ID pattern already seen twice and is a likely next failure). `personVinciId` and `titleCode` remain deliberately excluded/placeholder respectively. Replace each with a real Dayforce-sourced mapping (or remove it) as CPD's validation flags it. | `sc-api-cpd` `CreatePerson`/`CreateOrUpdateContract` actions |
| 6c | Test-data-only issue, not a mapping/schema bug: `BirthDate` and `SeniorityDate`'s `testWith` values on `CreatePerson` were both `1997-05-06` (identical), and CPD enforces `groupEntryDate` (mapped from `SeniorityDate`) must be at least 12 years after `birthDate` — a real 422 (`"GroupEntryDate should be superior to BirthDate + 12 years"`) confirmed this. Fixed `SeniorityDate`'s `testWith` to `2020-06-01`. Doesn't affect real Dayforce data, since real `SeniorityDate` values won't coincide with `BirthDate`. | `sc-api-cpd` `CreatePerson` action (test values only) |
| 6d | **Genuine new production gap, not just test data**: removing `hrInformation.lineManager` entirely (after its earlier fake VINCI ID failed a Luhn check, item 6b) turned out to be wrong — real 422s confirm CPD **requires** all three manager references on this contract shape (`"'Associated Line Manager Person Id' must not be empty."`, then the same for `operationalManager`; `administrativeManager` added proactively expecting the same, not yet individually confirmed but following the identical pattern). There is no Dayforce-to-manager-VinciId mapping in this build at all — build guide 4.8/4.9 never surfaced one. For now, all three (`lineManager`, `operationalManager`, `administrativeManager`) use the same real, confirmed-valid VINCI ID (`04470043`, the "Yann Chaitas" test person pulled from a real `Get Persons` response earlier in this project) purely so testing can proceed — **this is not a real manager mapping and must not go to production as-is.** Before go-live, this needs either genuine Dayforce fields that identify an employee's line/operational/administrative managers (translated to each manager's CPD VINCI ID, likely via a `p_SearchPerson`-style lookup) or confirmation from the client that some of these can share a value or be defaulted for certain employee categories. | `sc-api-cpd` `CreatePerson` action, eventually `p_CreatePersonAndContract` |
| 6e | `digitalIdentity.validFrom` was reusing `{ $SeniorityDate }` (a past hire date) — CPD rejected it: `"ValidFrom should be at least today's date for VinciConstruction systems with pending IdSync integration when being added"`. This field means something different from a hire date (when the digital/login identity itself becomes active, apparently constrained to today-or-later for orgs with a pending identity sync), so reusing `SeniorityDate` was a mapping mistake, not just a bad test value. Changed to a bare `{ fn:current-date() }` first, which produced a *different* real error: `"validFrom must be a System.DateOnly value."` (plus a cascading `"The personDto field is required"`, very likely just a symptom of that same nested binding failure) — `fn:current-date()` returns `xs:date` with an implicit timezone suffix (e.g. `2026-08-05+00:00`), which .NET's strict `DateOnly` parser rejects. **Confirmed fix**: `{ fn:format-date(fn:current-date(), '[Y0001]-[M01]-[D01]') }`, explicitly formatting just the date with no timezone, matching the `fn:format-dateTime`/picture-string pattern already used in `p_GetChangedEmployees`. This also confirms full XQuery function calls (not just bare `$ParamName` variables) do work inside a Body's `{ }` enclosed expression — the earlier open question in this item. | `sc-api-cpd` `CreatePerson` action |
| 6f | Once `CreatePerson` worked standalone (tested directly against the connector) and the process was tried end-to-end, Process Designer's validator rejected `p_CreatePersonAndContract`/`p_UpdatePersonAndContract`'s Service steps: `"The source field input.in_ActiveWorkContract/startDate for the input parameter HireDate is undefined. Cannot find the fields input and input.in_ActiveWorkContract/startDate."` (same for `endDate`→`EmploymentStatus_EffectiveTo` and `operationalJobTitle`→`Job`). Root cause: `in_ActiveWorkContract` is declared as untyped `xml` with no sub-schema, so the validator can't resolve an inline XPath (`input.in_ActiveWorkContract/startDate`) used directly as a Service step's `serviceInput` source — unlike a `<service>` call, this exact same XPath pattern is fine when used inside an `<assignment>`'s XQuery `<expression>`. Fixed in both processes by extracting `startDate`/`endDate`/`operationalJobTitle` into their own declared temp fields (`tmp_HireDate`/`tmp_EmploymentStatus_EffectiveTo`/`tmp_Job`) via XQuery formula assignments first, then referencing those temp fields in `serviceInput` — the same pattern `tmp_resolved_COR_ouId` already used. | `p_CreatePersonAndContract`, `p_UpdatePersonAndContract` |
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
