# CPD-Dayforce Integration — Build Notes

This repo tracks the IICS Explore export for the **CPD-Dayforce Integration** project (`Prcs_Dayforce_CPD` orchestrator + supporting connectors/sub-processes), built against `Dayforce_CPD_Functional_Design_Build_Guide.docx` v1.0.

`IICSExport.zip` at the repo root is the re-importable package (see "Regenerating the zip" below). Everything under `Explore/` is the same content extracted to individual files so changes are reviewable in git diffs/PRs — **edit the extracted files, not the zip directly.**

## How this was built

The original repo only contained `Prcs_Dayforce_CPD` as an empty `Start -> End` process plus two connectors (originally named `cnctr_cpd_get`, `cnctr-test-dayforce` — since renamed/republished in IICS as `SvcConn_Get_Post_Cpd` and `SvcConn_get_post_dayforce`, see "Connectors and Connections" below) captured from live test calls, including several hardcoded secrets (a Dayforce password and long-lived Bearer JWTs, and a CPD Bearer token). This pass:

- Removed every hardcoded secret and replaced it with a `{curly-brace}` bind parameter (`access_token`, `cpd_token`, `df_username`, `df_password`) — **the exposed credentials were live at the time this repo was read and should be rotated regardless of this change.**
- Extended both connectors with every REST action listed in the design doc's API reference (section 3).
- Built the orchestrator (`Prcs_Dayforce_CPD`) and 11 sub-processes implementing the design doc's build guide (section 4).

## Connectors and Connections

IICS's real object model, confirmed by actually publishing these in the org: **Service Connector** (the `AI_SERVICE_CONNECTOR` file — defines actions/operations) → **Connection** (`AI_CONNECTION` — wraps a Service Connector with an agent/auth binding) → **Process Service step** (references the Service Connector's action; the Connection is what makes that action executable by a Secure Agent). A raw Service Connector cannot be selected as a Service step's target until it is (a) schema-valid and (b) has at least one Connection built on top of it.

Getting the two Service Connectors to actually **publish** took three fix rounds, all confirmed by testing directly against this org (not guessed):
1. **Stray XML comments** (`<!-- -->` as the first child of `<types1:Entry>`) — removed. Every real IICS export checked had zero comments anywhere in the object body.
2. ~~`entireResponse="true"` output fields~~ — briefly suspected and reverted, then **confirmed valid** after finding a real published connector using it successfully. Not reintroduced in this pass (output fields were left empty/minimal on the working connectors as republished), but it's safe to add back if response-body access is needed later.
3. **`<input><field .../></input>` instead of `<input><parameter .../></input>`** — this was the actual bug. `<field>` is only correct inside `<output>`; `<input>` uses `<parameter name=".." type=".." required=".."/>`. Confirmed both by a real published reference connector and by the user successfully publishing both connectors after this fix.

Once published, the user created the Connections (`Conn-get-post-cpd`, `Conn-get-post-dayforce`, both `AI_CONNECTION` objects, both published, both bound to agent `hubdevinfoadm02`) and renamed/republished the Service Connectors as `SvcConn_Get_Post_Cpd` and `SvcConn_get_post_dayforce`.

**Important, confirmed directly in Process Designer**: a `<service>` step's "Service Type" picker resolves by **Connection name**, not Service Connector name — confirmed by the user checking the dropdown after the connector-name version still showed "Service Type: Select" / empty Input Fields despite both connectors being published. All `<service>` steps across every process file reference the **Connection**:

| | Service Connector (defines actions) | Connection (what `<service>` steps reference) | Connection GUID (`<serviceGUID>`) |
|---|---|---|---|
| CPD | `SvcConn_Get_Post_Cpd` (GUID `aDMzhpLojjoke0pnhn1H4E`, unchanged from the original `cnctr_cpd_get` — renamed in place) | `Conn-get-post-cpd` | `0zkkeyRXHvpjfEh4Tgq5sM` |
| Dayforce | `SvcConn_get_post_dayforce` (GUID `kvKgEPCM9y7i86SJRF8qEK`, new — recreated rather than renamed from `cnctr-test-dayforce`) | `Conn-get-post-dayforce` | `jEzhmqLAzmBhdgzZWel1I2` |

So every `<serviceName>` is `Conn-get-post-cpd:ActionName` or `Conn-get-post-dayforce:ActionName` (not the `SvcConn_*` connector name), and every matching `<serviceGUID>` is the **Connection's** own GUID, not the connector's. The action names themselves (`SearchPersonByExtId`, `CreatePerson`, etc.) are unchanged — they're still defined on the Service Connector, just addressed indirectly through the Connection.

## Process cross-references: IICS reassigns GUIDs on import

After the Connection fix above, 9 of the 11 processes imported cleanly. The remaining two (`ProcessOneEmployee_Process`, `Prcs_Dayforce_CPD`) failed with a generic "Internal application provider error" — and those two are exactly the only files that call *other processes in this same package* via `<subflow>` (everything else only calls the two Connections, which already existed).

Confirmed by re-exporting the 9 successfully-imported processes from the org and diffing: **IICS assigns each newly-imported process a brand new `GUID`/`types1:GUID`, ignoring whatever GUID was declared in the imported XML.** e.g. `UpdatePhoneNumber_Process` was authored with GUID `2MQBdKskt65bUvaVMnxm9g`; after import IICS assigned it `0ncoBf8fJhphLIRIv0RXOS`. Every `<subflowGUID>` reference pointing at the *authored* GUID of a sibling process therefore breaks the moment that sibling is actually imported, since its real GUID is different.

The diff also confirmed this is **the only thing that needed fixing** — content, steps, links, and logic were preserved byte-for-byte (modulo formatting/whitespace and bookkeeping fields like `ParentFlowIds`/timestamps), which is good independent confirmation that the schema itself (service calls, assignments, containers, event handling) was already correct.

Fixed by remapping every `<subflowGUID>` in `ProcessOneEmployee_Process` and `Prcs_Dayforce_CPD` to the real post-import GUIDs of the 9 already-imported processes:

| Process | Authored GUID | Real (post-import) GUID |
|---|---|---|
| GetChangedEmployees_Process | `3oBo3euvAbRVfbijBuFPeO` | `82BXsUMh6wDfTfmUadXuw1` |
| GetEmployeeDetail_Process | `KPffEx7EOwfjp7pn7kzZ2M` | `aDhurm6YPSRfpX6jx48p6H` |
| SearchPerson_Process | `DB1JRZkPV3HvqKV1vfjqgb` | `7n7iMGoFQB6dEzRA4TyXyr` |
| CreatePersonAndContract_Process | `ugYTHbESlUs3xY4PcfoFaW` | `7fxMqJIQj2eigrAD3AjsqS` |
| UpdatePersonAndContract_Process | `TUaU8PTn1SEcDVyUTRKb6U` | `cPu3ROm1I27g3XxioV28Yq` |
| WriteVinciId_Process | `zBI0ZwpxPB1sKMv6wHgpd1` | `a7jAK3mkjWNb0FDymjNpiw` |
| AddPhoneNumber_Process | `AwxVtpAsbduNEHRbWUeFYb` | `9WIET5Qv4lVl2j23ISZyoe` |
| UpdatePhoneNumber_Process | `2MQBdKskt65bUvaVMnxm9g` | `0ncoBf8fJhphLIRIv0RXOS` |
| DeletePhoneNumber_Process | `MrEEevrr6RSALfwqq4ZHmt` | `jlM9u0QPKK9ltcTL4PIAGA` |
| ErrorAlert_Process | `xpZXrmJaYOBu7TJpkrH1WV` | `1qQ4Y1EizO2lEjgLImYXaP` |

**Resolved**: `ProcessOneEmployee_Process` imported successfully once its 9 references were corrected (real post-import GUID: `2jvNEMZkr0SeKWtoaSMfYh`, vs. the authored `VYIrQ7qaaXeojtBdb9dXHt`). `Prcs_Dayforce_CPD`'s `<subflow>` call to it has been updated to that real GUID. All 11 processes should now import and resolve cleanly.

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
| Loop over a list | `runForEach="true"` on a `<subflow>` call | **There is no separate "repeat/forEach" element.** Looping only exists as "call this one sub-process once per item in a list field." This is why a new sub-process, `ProcessOneEmployee_Process`, was introduced to hold the whole per-employee body — the Orchestrator loops by calling it with `runForEach="true"`, since there was no way to loop an inline multi-step sequence directly |
| Decision / branch | `<container id=".." type="exclusive"><flow id="branchA">...</flow><flow id="branchB">...</flow><link targetId="branchA" type="containerLink"><condition source="formula"><function name="true"><arg name="field">{$temp.xxx}</arg></function></condition></link><link targetId="branchB" type="containerLink"><condition source="formula"><function name="false">...</function></condition></link><link targetId="nextNode"/></container>` | Branch `<flow>` blocks have no `<start>`/`<end>` of their own — their last activity links back to the container's own `id` with `type="containerLink"` to signal "branch done" |
| Fault handling (try/catch) | `<eventContainer id=".."><flow id="normal">...</flow><flow id="fault">...</flow><link targetId="normalOrFault" type="containerLink"/> (routing) <link targetId="nextNode"/> (exit) <events><catch faultField="faultInfo" interrupting="true"><link targetId="faultFlowId" type="containerLink"/></catch></events></eventContainer>` | Fault details become available as `output.faultInfo[1]/code`, `.../reason`, `.../detail` inside the catch's fault flow |

**Confidence levels**, being specific about what's now verified vs. still inferred:
- **High confidence** (directly confirmed across 3+ independent real exports): `<input>`/`<output>`/`<tempFields>`, `<assignment>`/`<operation>`/XQuery `<expression>`, `<service>`/`serviceName`/`serviceGUID`/`serviceInput`, `<subflow>` (standalone, `runForEach="false"`), `<container type="exclusive">` decision branching, the `<eventContainer>`/`<events><catch>` fault pattern (for a single watched activity).
- **Lower confidence, flagged inline in the affected files**:
  - `runForEach="true"` — no real example with it set to `true` was found in any public export searched; only the field's existence and default-`false` value are confirmed. `ProcessOneEmployee_Process`'s three phone-loop `<subflow>` calls and the Orchestrator's `sf-processemployees` call use it, with each per-item field bound directly to the list field (e.g. `temp.tmp_PhonesToAdd/phoneTypeCode`) — check Process Designer's own loop-configuration UI after import and correct the binding if it differs.
  - The `<eventContainer>` fault pattern was confirmed for a container watching **one** activity (a single `<subflow>` or `<service>`); `ProcessOneEmployee_Process` wraps a **whole multi-step sequence** in its `normalFlow` branch instead. The container/catch/containerLink mechanics should be the same, but this specific shape (multi-step watched flow) wasn't seen in an example — re-verify in Process Designer.
  - Exact response shapes: no sample was available for the Dayforce delta/bulk employee list, the expanded `WorkContracts`/`Contacts` sub-trees, or the CPD phone-number list — all XQuery paths touching those are best-effort against the one flat single-employee sample and one no-phones person-search sample actually provided.

Before treating any of these processes as build-ready:
1. Open each `.PROCESS.xml` in IICS Process Designer (or re-import the zip into an Explore dev project) and confirm it now validates without the blank/un-typed step problem from the first pass.
2. Pay special attention to the `runForEach="true"` subflow calls and the `ProcessOneEmployee_Process` fault container — these are the two areas flagged above as lower-confidence.
3. Re-save from Process Designer once validated so the file picks up your IICS version's canonical XML (visual layout coordinates, etc.), then re-export.

## Open items (carried over from the design doc, not introduced by this build)

| # | Item | Where it shows up |
|---|------|-------------------|
| 1 | CPD hostname is a placeholder (`{cpd-host}`) | `SvcConn_Get_Post_Cpd` — every new CPD action |
| 2 | Dayforce EmployeeProperties XRefCode for the VINCI ID field is unconfirmed | `SvcConn_get_post_dayforce` `WriteVinciIdToDayforce` action, `WriteVinciId_Process` |
| 3 | Dayforce org (LedgerCode) → CPD `ouId` cross-reference table doesn't exist yet | `CreatePersonAndContract_Process` / `UpdatePersonAndContract_Process` (`tmp_resolved_COR_ouId`) |
| 4 | Dayforce contact/phone type → CPD `phoneTypeCode` mapping is unconfirmed, and no sample of the expanded `Contacts.Items` shape was available | `ProcessOneEmployee_Process` `dp-comparephones` step |
| 5 | CPD authentication mechanism is unconfirmed (no CPD equivalent of Dayforce's "Get token" action exists yet) | `SvcConn_Get_Post_Cpd` — `in_CpdToken` is a process input (secure parameter) until this is resolved |
| 6 | Create-path `contractId` source is unconfirmed — the design doc's Create/Update Contract call is a single PATCH to an existing `{contractId}`, which presupposes CPD already issued one on Create Person | `CreatePersonAndContract_Process` |
| 7 | Search Person match criteria — this build uses **Ext Id / VINCI ID** per project decision (not name+DOB). Known limitation carried from the design doc: this can't by itself distinguish a genuine new hire from a prior-run assignment that hasn't propagated yet | `SearchPerson_Process`, `SvcConn_Get_Post_Cpd` `SearchPersonByExtId` action |
| 8 | Error-alert recipients and format are unconfirmed, **and no Email/notification connector exists yet in this project** to reference | `ErrorAlert_Process` only formats the alert text (`out_Subject`/`out_Body`); it does not send anything yet — add a `<service>` step once an Email connector exists |
| 9 | `LastRunTimestamp` persistence across scheduled runs (the Orchestrator takes it as an input/emits `out_CurrentRunTimestamp` as an output, but the actual storage — a cache table, a file, a custom parameter — isn't wired up) | `Prcs_Dayforce_CPD`, `GetChangedEmployees_Process` |
| 10 | The Schedule object (build guide 4.16) is an IICS Administrator config, not part of this Explore export — must be created separately once the process is deployed | N/A |

## Process inventory

| Process | Role |
|---|---|
| `Prcs_Dayforce_CPD` | Orchestrator: get changed employees, get a run-scoped Dayforce token, loop `ProcessOneEmployee_Process` over each XRefCode |
| `GetChangedEmployees_Process` | Dayforce delta query (build guide 4.4) |
| `ProcessOneEmployee_Process` | **New in this pass.** Per-employee body: detail + search + create-or-update decision + phone reconciliation, fault-wrapped. Looped by the Orchestrator via `runForEach` |
| `GetEmployeeDetail_Process` | Dayforce employee detail, expanded (build guide 4.5) |
| `SearchPerson_Process` | CPD search by Ext Id/VINCI ID (build guide 4.6) |
| `CreatePersonAndContract_Process` | New-hire path (build guide 4.8) |
| `UpdatePersonAndContract_Process` | Existing-employee path (build guide 4.9) |
| `WriteVinciId_Process` | VINCI ID writeback, create path only (build guide 4.10) |
| `AddPhoneNumber_Process` / `UpdatePhoneNumber_Process` / `DeletePhoneNumber_Process` | Phone reconciliation actions (build guide 4.12), looped by `ProcessOneEmployee_Process` |
| `ErrorAlert_Process` | Formats a fault notification (build guide 4.13-4.14) — sending is not yet wired, see Open Item 8 |

## Regenerating the zip

After editing files under `Explore/`, rebuild `IICSExport.zip` and refresh `exportPackage.chksum` (plain uppercase-hex SHA-256 per file, same format Informatica's export produces):

```bash
cd /home/user/IICS
rm -f IICSExport.zip
zip -r IICSExport.zip Explore exportMetadata.v2.json ContentsofExportPackage_IICSExport.csv exportPackage.chksum
```

Recompute `exportPackage.chksum` first if any tracked file changed (plain SHA-256 hex, uppercased, of each file's bytes, keyed by its repo-relative path with spaces escaped as `\ `).
