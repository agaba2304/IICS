# CPD-Dayforce Integration — Build Notes

This repo tracks the IICS Explore export for the **CPD-Dayforce Integration** project (`Prcs_Dayforce_CPD` orchestrator + supporting connectors/sub-processes), built against `Dayforce_CPD_Functional_Design_Build_Guide.docx` v1.0.

`IICSExport.zip` at the repo root is the re-importable package (see "Regenerating the zip" below). Everything under `Explore/` is the same content extracted to individual files so changes are reviewable in git diffs/PRs — **edit the extracted files, not the zip directly.**

## How this was built

The original repo only contained `Prcs_Dayforce_CPD` as an empty `Start -> End` process plus two connectors (`cnctr_cpd_get`, `cnctr-test-dayforce`) captured from live test calls, including several hardcoded secrets (a Dayforce password and long-lived Bearer JWTs, and a CPD Bearer token). This pass:

- Removed every hardcoded secret and replaced it with a `{curly-brace}` bind parameter (`access_token`, `cpd_token`, `df_username`, `df_password`) — **the exposed credentials were live at the time this repo was read and should be rotated regardless of this change.**
- Extended both connectors with every REST action listed in the design doc's API reference (section 3).
- Built the orchestrator (`Prcs_Dayforce_CPD`) and 10 sub-processes implementing the design doc's 16-step build guide (section 4).

## Important: validate node types in IICS Process Designer before deploying

The only authentic example available of this repo's process XML schema (`avosScreenflow.xsd` / `taskflowModel.xsd`) was the original **trivial** `Start -> End` process — no populated example with real service calls, decisions, loops, or fault handlers was available to model from. The process files here were hand-authored using:
- That one working example, for the `<flow>`/`<start>`/`<end>`/`<link>` graph pattern (a link is a child of the node it originates from, pointing at a `targetId`).
- General knowledge of IICS Cloud Application Integration / ActiveVOS's BPEL-derived activity model, for the higher-level constructs introduced here: `<invoke>` (call a connector action), `<invokeProcess>` (call a sub-process), `<decision>`/`<case>` (branch), `<forEach>` (loop over a list), `<dataProcess>`/`<expression>` (mapper/expression logic), `<scope>`/`<faultHandlers>`/`<catchAll>` (fault handling).

**This has not been validated against a live IICS org or Process Designer.** Before treating any of these processes as build-ready:
1. Open each `.PROCESS.xml` in IICS Process Designer (or re-import the zip into an Explore dev project) and confirm every step type resolves to a real palette item (Service Call, Sub-Process/Call a Process, Decision, Repeat, Data Process, Scope/Try-Catch).
2. Expect to re-wire steps using the visual designer where the hand-authored XML doesn't match your tenant's exact schema — the *logic* (call order, field mappings, branching, fault routing) is the deliverable here, not a guarantee of byte-for-byte importability.
3. Re-save from Process Designer once validated so the file picks up your IICS version's canonical XML (visual layout coordinates, generated IDs, etc.), then re-export.

## Open items (carried over from the design doc, not introduced by this build)

| # | Item | Where it shows up |
|---|------|-------------------|
| 1 | CPD hostname is a placeholder (`{cpd-host}`) | `cnctr_cpd_get` — every new CPD action |
| 2 | Dayforce EmployeeProperties XRefCode for the VINCI ID field is unconfirmed | `cnctr-test-dayforce` `WriteVinciIdToDayforce` action, `WriteVinciId_Process` |
| 3 | Dayforce org (LedgerCode/XRefCode) → CPD `ouId` cross-reference table doesn't exist yet | `CreatePersonAndContract_Process` / `UpdatePersonAndContract_Process` (`dp-mapOrg` / `resolved_COR_ouId`) |
| 4 | Dayforce contact/phone type → CPD `phoneTypeCode` mapping is unconfirmed, and no sample of the expanded `Contacts.Items` shape was available | Orchestrator `dp-comparePhones` step |
| 5 | CPD authentication mechanism is unconfirmed (no CPD equivalent of Dayforce's "Get token" action exists yet) | `cnctr_cpd_get` — `cpd_token` is sourced from a process parameter until this is resolved |
| 6 | Create-path `contractId` source is unconfirmed — the design doc's Create/Update Contract call is a single PATCH to an existing `{contractId}`, which presupposes CPD already issued one on Create Person | `CreatePersonAndContract_Process` |
| 7 | Search Person match criteria — this build uses **Ext Id / VINCI ID** per project decision (not name+DOB). Known limitation carried from the design doc: this can't by itself distinguish a genuine new hire from a prior-run assignment that hasn't propagated yet | `SearchPerson_Process`, `cnctr_cpd_get` `SearchPersonByExtId` action |
| 8 | Error-alert recipient list and per-record vs. digest format are unconfirmed | `ErrorAlert_Process` |
| 9 | `LastRunTimestamp` persistence across scheduled runs (the Orchestrator takes it as an input/emits `CurrentRunTimestamp` as an output, but the actual storage — a cache table, a file, a custom parameter — isn't wired up) | `Prcs_Dayforce_CPD`, `GetChangedEmployees_Process` |
| 10 | The Schedule object (build guide 4.16) is an IICS Administrator config, not part of this Explore export — must be created separately once the process is deployed | N/A |

## Regenerating the zip

After editing files under `Explore/`, rebuild `IICSExport.zip` and refresh `exportPackage.chksum` (plain uppercase-hex SHA-256 per file, same format Informatica's export produces):

```bash
cd /home/user/IICS
rm -f IICSExport.zip
zip -r IICSExport.zip Explore exportMetadata.v2.json ContentsofExportPackage_IICSExport.csv exportPackage.chksum
```

Recompute `exportPackage.chksum` first if any tracked file changed (see the hashing script used in this build's git history for the exact format).
