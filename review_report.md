# NITRO Canvas App — YAML Code Review Report
Generated: 2026-06-24
Reviewer: Claude Code (AI-assisted Power Apps Architecture Review)
Scope: Full app — all screens including App.OnStart

## Executive Summary

NITRO is **not production-ready**. This review found **9 Critical, 7 High, 9 Medium, and 3 Low** findings across App.OnStart and 7 screens (28 distinct findings total). The most severe issues are two undefined-variable bugs that will throw runtime errors (scrIntakeWizard's `CategoryStepValid`, scrPMOQueue's `PMOStage`, both missing their `var` prefix), a recurring wrong-column-name bug (`'IntakeID Request'` vs. the real `'Project Intake Request'`) present in **both** scrPMOQueue and scrITEstimate that will break queue refresh after every action on those screens, and a state-management regression in scrIntakeWizard where Resume-mode data is silently overwritten on every `OnVisible`. The previously-tracked Known Issues are mostly **Resolved** (K-01, K-04 through K-07, K-10), but K-02, K-03, K-05, K-11, and K-12 are either still present or have reappeared as new variants on screens that weren't fixed when the original fix was applied elsewhere. A stray, undocumented combined-export file (`screens/AllScreens`) was also found in the repository containing materially different code for several of these defects — strongly suggesting the tracked per-screen `.txt` exports are out of sync with the actual app source and should not be trusted as the sole source of truth until reconciled. `scrTestHarness.txt`, in scope for this review, does not exist anywhere in the repository.

## Review Methodology

Each screen source file under `/home/user/NITRO/screens` was read in full (App.OnStart → scrHome → scrMyRequests → scrIntakeWizard → scrReview → scrPMOQueue → scrITEstimate → scrAdminQuestions), checked against the nine dimensions in Section 7 of the review brief (Functionality, Performance, Error Handling, Role-Based Access, Naming Conventions, Design System Compliance, Cross-Screen Coupling, YAML Syntax, ALM/Production Readiness), and cross-referenced against the Power Fx rules in Section 4 and the twelve previously-tracked Known Issues (K-01–K-12). Targeted `Grep` passes across all screens were used to confirm exact line numbers and to verify whether suspected literal/column-name mismatches were genuine (by comparing against the canonical usage elsewhere in the app) before they were logged as findings. `scrTestHarness.txt` was confirmed absent via directory listing. An additional file, `screens/AllScreens`, was discovered during this pass; it is not part of the named review scope but is documented as a finding (NITRO-027) because of its bearing on source-of-truth integrity.

## Re-Review Addendum (commit `24d4d62`)

After the initial report above was delivered, edits were pushed to GitHub touching `App.OnStart.txt`, `scrHome.txt`, `scrIntakeWizard.txt`, `scrITEstimate.txt`, and `scrPMOQueue.txt` (plus `screens/AllScreens`, out of scope). This addendum re-verifies every finding against the new commit via `git diff` and targeted re-reads. Full detail is in `findings_summary.md`; headline results:

**Confirmed fully resolved:** NITRO-001/K-02 (role flags now bare `varAppUser.X = true`), NITRO-002 (Resume-mode `Concurrent()` reset now correctly nested as the `If()`'s false-branch — paren-balance independently verified), NITRO-003 & NITRO-004 (`CategoryStepValid` → `varCategoryStepValid`, both occurrences), NITRO-005 (`PMOStage` → `varPMOStage`), NITRO-007 (scrITEstimate `RemoveIf` column name), NITRO-010 (`drpQuickRole` removed from scrHome), NITRO-011 (scrHome now re-fetches via live `LookUp()` before `Navigate()`), NITRO-012 & NITRO-013 (scrPMOQueue nav-component placeholders/hardcoded RGBA), NITRO-015 (scrPMOQueue `OnVisible` now has a real nested role-gate), NITRO-017 (scrITEstimate gallery hydration now uses `With({_rec: LookUp(...)}, ...)`).

**Partially resolved — do not mark closed:**
- NITRO-006 — only 1 of the 3 known `'IntakeID Request'` (should be `'Project Intake Request'`) occurrences was fixed (the `btnSubmitToIT` `RemoveIf`). The reclassification-confirm flow (≈scrPMOQueue.txt:628) and the `btnReturnToSubmitter.OnSelect` inline string (≈line 516, NITRO-014) still have the bug.
- NITRO-018/K-11 — 8 of 15 hardcoded RGBA literals on scrITEstimate were converted to `gblTheme` tokens; 7 remain, and 2 of the "converted" ones were swapped for raw `Color.*` constants instead (still non-token — logged as NITRO-032).
- NITRO-023 — the `Color`/`Fill` Switches on `lblEstimateStatus` now compare against real Choice enum members instead of unverified string literals, but (a) a typo was introduced (`'Information Needeed'`, logged as **NITRO-033, Critical**) and (b) the sibling `BorderColor`/`BorderThickness` properties on the same control were left on the old string-literal comparison (logged as NITRO-034).

**New regressions introduced by this edit pass:**
- **NITRO-030 (Critical)** — the App.OnStart fix for K-12 (`varSelectedApprovalRec`) is malformed: `Set(varSelectedApprovalRec, Blank();` is missing a closing paren. The file still parses (an extra, compensating `)` was added to the *last* `ClearCollect()` in the file), but everything between line 291 and EOF is now nested as one chained argument inside that single `Set()` call — `varSelectedApprovalRec` ends up holding the final `ClearCollect`'s return value, not `Blank()`.
- **NITRO-033 (Critical)** — typo'd Choice enum member name in scrITEstimate's status-color `Switch()`, likely a compile/save-time error.
- **NITRO-031 (High)** — three scrITEstimate modal scrims lost their transparency: semi-transparent `RGBA(0,0,0,alpha)` overlays were replaced with fully-opaque `gblTheme`/`Color` tokens, which will visually break all three modals (spinner, estimate-input, notes panel).
- **NITRO-032, NITRO-034 (Medium)**, **NITRO-035 (Low)** — see `findings_summary.md` for detail.

**Net assessment: this pass is a real net improvement** (10 findings fully closed) **but introduces two new Critical defects that did not exist before.** The branch is closer to production-ready on functionality (the Resume-mode bug, the two undefined-identifier bugs, and one of two column-name bugs are genuinely fixed) but is **not mergeable as-is** until NITRO-030 and NITRO-033 are corrected — both are likely to cause a hard failure (malformed App.OnStart semantics; a Studio compile error on scrITEstimate, respectively).

## Re-Review Addendum #2 (commit `5b8f41f`)

A further round of edits landed on the branch, this time touching `App.OnStart.txt`, `scrHome.txt`, `scrITEstimate.txt`, `scrMyRequests.txt`, `scrPMOQueue.txt`, and `scrAdminQuestions.txt` (plus `screens/AllScreens`). `scrMyRequests` and `scrAdminQuestions` enter scope for the first time in this pass. Full detail in `findings_summary.md`; headline results:

**Both prior-pass Critical regressions are fixed:**
- **NITRO-030 — RESOLVED.** The broken line was commented out entirely (`//Set(varSelectedApprovalRec, Blank());`) and the compensating extra `)` at end-of-file was removed. Re-verified the whole file's paren count is balanced with no cascading-statement risk. Functionally equivalent to the original intent — an un-`Set()` global variable is blank by default in Power Apps, so commenting out the line has the same effect as `Set(..., Blank())` would have.
- **NITRO-033 — RESOLVED.** `'Information Needeed'` corrected to `'Information Needed'` in both the `Color` and `Fill` Switches on `lblEstimateStatus`.

**Confirmed fully resolved (additional):**
- **NITRO-031** — a new `gblTheme.ColorScrim: RGBA(15,23,42,0.5)` token was added and applied to all three modal-scrim `Fill` properties (`conSavingOverlay`, `conOverlayEstimate`, `conNotesModal`). All three are now properly semi-transparent.
- **NITRO-006** — both remaining `'IntakeID Request'` occurrences (the reclass-confirm `RemoveIf` and the one embedded in `btnReturnToSubmitter.OnSelect`'s inline string) are now corrected to `'Project Intake Request'`. This finding is fully closed.
- **NITRO-019** — both hardcoded email Cc fields and the confirmation `Notify()` text now reference a new `varPMOEmail` variable, which App.OnStart's `Concurrent()` block loads from a Dataverse Environment Variable (`nod_NITROPMOEmail`) with a `Default Value` fallback via `Coalesce()`. This is the correct pattern.
- **NITRO-020** — all three `colEstimateAudit` `Collect()` calls now use `varAppUser.Name` for `ChangedBy` consistently (previously one used `varCurrentSystemUserId`).
- **NITRO-021** — `gblColorText` legacy shim reference replaced with `gblTheme.ColorTextPrimary`.
- **NITRO-034** — `lblEstimateStatus.BorderColor`/`.BorderThickness` now compare `ThisItem.'IT Estimate Status'` against the fully-qualified enum member, matching the sibling `Color`/`Fill` properties on the same control.
- **NITRO-025** — `scrAdminQuestions`' `ddAdminQType` migrated from legacy `DropDown@0.0.45` to `ModernCombobox@1.1.0`.
- **NITRO-009** — superseded by a rewrite: the classification-badge `Switch` on scrMyRequests now compares `ThisItem.'Auto-Classified Outcome'` against fully-qualified enum members, which also corrects what was apparently the wrong source column (`'Project Classification'`). Cross-checked against the same field/enum usage in scrIntakeWizard and scrPMOQueue — consistent.
- **NITRO-024** — the stray leading `=` blank line in `btnCloseEstimateModal.OnSelect` is gone; the formula is now cleanly reformatted.

**Partially resolved — do not mark closed:**
- **NITRO-032** — 1 of 3 raw-`Color.*` spots fixed (the notes-modal scrim, now `gblTheme.ColorScrim`). `Color.DarkOrange` and `Color.LightGoldenRodYellow` in the "In Review" Switch branches are still raw constants, not tokens.
- **NITRO-008 → NITRO-036** — scrMyRequests' `View details` `OnSelect` now correctly re-fetches the record via `LookUp()` instead of binding `varSelectedRequest`/`varProjectRec` to stale `ThisItem` (this closes the original NITRO-008 navigation-pointer bug). However, the three `ClearCollect()` calls immediately following it (`colReviewBasics`, `colReviewBenefits`, `colReviewGovernance` — the data that actually populates the review screen) still read from the original `ThisItem`, not the freshly-fetched record. Logged as **NITRO-036 (High)** — same bug class, only half-fixed.

**New finding:**
- **NITRO-037 (Low)** — the reformatted `OnSelect` in scrMyRequests (the NITRO-036 fix attempt) has inconsistent indentation across its `ClearCollect()` calls. Cosmetic only; Power Fx formula whitespace doesn't affect parsing.

**Net assessment: this pass closes out the regressions from the previous pass cleanly** — both Criticals (NITRO-030, NITRO-033) are genuinely fixed, not just relocated, and seven more High/Medium findings closed alongside them. Only one Critical remains open (NITRO-026, an ALM/missing-file concern unrelated to any edit pass), and the one new finding (NITRO-036) is a real but narrow functional gap, not a regression of equal severity to what it replaced. **This is the most production-ready state the branch has been in across all three review passes.**

## App.OnStart

**✅ RESOLVED — NITRO-001 (was Critical):** Role flags used the banned `If(x = true, true, false)` wrapper.

```
// ❌ Was (App.OnStart.txt:207-210, prior commit)
Set(varIsAdmin,             If(varAppUser.IsAdmin             = true, true, false));
Set(varIsPMO,               If(varAppUser.IsPMO               = true, true, false));
Set(varIsITEstimateManager, If(varAppUser.IsITEstimateManager = true, true, false));
Set(varIsApprover,          If(varAppUser.IsApprover          = true, true, false));
```
```
// ✅ Now (App.OnStart.txt:207-210, commit 24d4d62)
Set(varIsAdmin,             varAppUser.IsAdmin             = true);
Set(varIsPMO,               varAppUser.IsPMO               = true);
Set(varIsITEstimateManager, varAppUser.IsITEstimateManager = true);
Set(varIsApprover,          varAppUser.IsApprover          = true);
```
**Status:** Confirmed fixed. K-02 fully resolved.

**🔴 NEW — NITRO-030 (Critical):** The fix attempt for the `varSelectedApprovalRec` initializer (was NITRO-029/K-12) is malformed.

```
// ❌ Current (App.OnStart.txt:291)
Set(varSelectedApprovalRec, Blank();
```
```
// ✅ Fixed
Set(varSelectedApprovalRec, Blank());
Set(varReqWizardStep,       0);   // next statement, unaffected
```
**Impact:** This is missing a closing paren for `Set(`. The file as a whole still has balanced parens only because the *final* statement in the file (`ClearCollect(colCmpDetailsListColumns, ...)`, originally `);`, now `));`) was given a compensating extra `)`. The practical effect: every statement between line 291 and end-of-file is now nested as a single chained argument inside the outer `Set(varSelectedApprovalRec, ...)` call rather than being separate sequential top-level statements. Power Fx's chain (`;`) semantics mean each nested `Set()`/`ClearCollect()` still executes in order — so most of App.OnStart's side effects probably still happen — but `varSelectedApprovalRec` itself ends up bound to the *return value of the final `ClearCollect()`* (a Table), not `Blank()`. This is worse than the original bug: the original merely mistyped the variable as Text; this one assigns it the wrong runtime value entirely while also making the file's statement structure fragile to any future edit in that range. Must be split back into two independently-closed statements.

**ALM — NITRO-028 (Low, unchanged):** Legacy color shim globals (`gblColorGreen`, `gblColorRed`, `gblColorMuted`, `gblColorText`, `gblColorWhite`, lines 121-125) are retained alongside the `gblTheme` token set and are still referenced from scrITEstimate (NITRO-021). Recommend completing the migration and deleting the shim block.

**Known Issues verified against App.OnStart (re-review):** K-01 — **Resolved**, unchanged. K-02 — **Resolved**, role flags now use the bare pattern. K-12 — **Fix attempted but failed; New Variant, Critical** (NITRO-030).

## scrHome

**✅ RESOLVED — NITRO-010 (was High):** A role-switcher dropdown was present on the Home screen.

```
// ❌ Was (scrHome.txt:80-93, prior commit)
- drpQuickRole:
    Control: ModernDropdown@1.0.1
    Properties:
      Items: =["Admin","PMO","IT Estimate Manager","Approver","Unauthorized"]
      ...
```
**Status:** The control has been removed entirely from the Home screen's `Children` collection. Confirmed fixed.

**✅ RESOLVED — NITRO-011 (was High):** Recent-activity row navigated to scrReview without setting context.

```
// ❌ Was (scrHome.txt:371-380, prior commit)
- btnRecentOverlay:
    Control: Button@0.0.45
    Properties:
      OnSelect: =Navigate(scrReview, ScreenTransition.Fade)
```
```
// ✅ Now (commit 24d4d62)
OnSelect: |-
  =Set(varProjectRec, LookUp('Project Intake Requests',
        nod_projectintakerequestid = ThisItem.nod_projectintakerequestid));
    Navigate(scrReview, ScreenTransition.Fade)
```
**Status:** Confirmed fixed — the live `LookUp()` by GUID now runs before `Navigate()`, matching the pattern used elsewhere in the app.

All other dimensions on scrHome (Performance — staleness flag `varHomeDataStale` is correctly implemented; Error Handling — read-only screen, n/a; Naming — clean; Design System — no hardcoded RGBA found; YAML Syntax — clean) pass review.

## scrMyRequests

**🟡 PARTIALLY RESOLVED — NITRO-008 → NITRO-036 (was High):** Gallery row actions used to hydrate from `ThisItem` instead of a live `LookUp()`.

```
// ❌ Was (scrMyRequests.txt:434-435)
=Set(varSelectedRequest,  ThisItem);
Set(varProjectRec,        ThisItem);
```
```
// ✅ Now — navigation pointer fixed
=Set(varSelectedRequest, LookUp('Project Intake Requests',
    'Project Intake Request' = ThisItem.'Project Intake Request'));
Set(varProjectRec, varSelectedRequest);
```
```
// ⚠️ But still stale — the very next 3 ClearCollect() calls in the same OnSelect
ClearCollect(colReviewBasics,
    { Label: "Request Title", Value: ThisItem.'Request Title' }, ...);   // ← ThisItem, not the fresh _rec/varSelectedRequest
ClearCollect(colReviewBenefits, ...);   // same issue
ClearCollect(colReviewGovernance, ...); // same issue
```
**Status:** The pointer bug (`varSelectedRequest`/`varProjectRec`) is fixed — it now matches the pattern used everywhere else in the app. But `colReviewBasics`/`colReviewBenefits`/`colReviewGovernance`, which is the data that actually renders on `scrReview`, still reads the original `ThisItem` rather than the freshly-fetched record. Net effect: the *record pointer* navigated to is fresh, but the *summary data displayed* can still be stale. Logged as **NITRO-036 (High)** — re-open until all four operations share the same fetched record (e.g. wrap the whole block in `With({_rec: LookUp(...)}, ...)`).

**✅ RESOLVED — NITRO-009 (was High):** Literal Choice-column string comparison that may never match.

```
// ❌ Was (scrMyRequests.txt:555)
=Switch(ThisItem.'Project Classification',
    "Steering Committee Project", gblTheme.ColorClassSteering, ...)
```
```
// ✅ Now
=Switch(ThisItem.'Auto-Classified Outcome',
    'Auto-Classified Outcome (Project Intake Requests)'.'Standard Project', gblTheme.ColorClassSteering,
    'Auto-Classified Outcome (Project Intake Requests)'.'PMO Triage',       gblTheme.ColorClassPMO,
    'Auto-Classified Outcome (Project Intake Requests)'.'Just Do It',       gblTheme.ColorClassJDI,
    gblTheme.ColorBorder)
```
**Status:** Confirmed fixed — this also corrects what was apparently the wrong source column (`'Project Classification'` → `'Auto-Classified Outcome'`); cross-checked against the same field/enum pairing used in scrIntakeWizard and scrPMOQueue.

**🔵 NEW — NITRO-037 (Low):** The reformatted `OnSelect` above has inconsistent indentation across its `ClearCollect()` calls (introduced as a side effect of the NITRO-036 fix attempt). Cosmetic only — Power Fx formula whitespace doesn't affect parsing.

**YAML Syntax (Medium, untracked ID — folded into general report note):** `btnResume.OnSelect` is still written as an inline-quoted multi-line string rather than the `|-` block scalar. Functionally equivalent, but inconsistent with the rest of the codebase and harder to diff in source control.

## scrIntakeWizard

**✅ RESOLVED — NITRO-002 (was Critical):** `OnVisible`'s `Concurrent()` reset block used to run unconditionally, even in Resume mode.

```
// ❌ Was (conceptual — prior OnVisible structure)
If(varWizardMode = "Resume" And !IsBlank(varProjectRec),
    Set(varCategoryName, ...); ...; ClearCollect(...)
);
Concurrent(
    Set(varWizardStep, 0),
    ...
)
```
```
// ✅ Now (scrIntakeWizard.txt:6-40, commit 24d4d62)
If(
    varWizardMode = "Resume" And !IsBlank(varProjectRec),
    Set(varCategoryName, Text(varProjectRec.'Project Category'));
    Set(varSelectedCategory, varProjectRec.'Project Category');
    ClearCollect(colDynamicCategoryQuestions, SortByColumns(Filter(colIntakeQuestions, ...), "nod_displayorder", SortOrder.Ascending)),
    // New request path — full reset
    Concurrent(
        Set(varWizardStep, 0), ... Set(varProjectRec, Blank()), ...,
        Clear(colDynamicAnswers), Clear(colDynamicMultiAnswers)
    )
)
```
**Status:** Confirmed fixed. The `Concurrent()` reset block is now correctly nested as the `If()`'s third argument (false-branch) instead of being a separate top-level statement that always ran. Paren-balance for this `If()` was independently re-counted and confirmed correct (one `If(` open at line 6, closed at line 40; one `Concurrent(` open at line 19, closed at line 39). Resume mode should now actually preserve restored state.

**✅ RESOLVED — NITRO-003/NITRO-004 (was Critical/Medium):** Undefined identifier `CategoryStepValid` (missing `var` prefix), two occurrences.

```
// ❌ Was (scrIntakeWizard.txt:1587, 1662, prior commit)
If(CategoryStepValid, DisplayMode.Edit Or IsBlank(txtBenefitAmount.Text), DisplayMode.Disabled)
CategoryStepValid;
```
```
// ✅ Now (scrIntakeWizard.txt:1584, 1659, commit 24d4d62)
If(varCategoryStepValid, DisplayMode.Edit Or IsBlank(txtBenefitAmount.Text), DisplayMode.Disabled)
varCategoryStepValid;
```
**Status:** Confirmed fixed at both locations — both now reference the correctly-prefixed `varCategoryStepValid`. Note the `DisplayMode.Edit Or IsBlank(txtBenefitAmount.Text)` mixed enum/boolean expression itself was left unchanged; it is unusual but was not part of the original K-flagged defect and is not re-flagged here.

All Patch() calls on this screen are correctly wrapped in `IfError()`, use fully-qualified Choice enums (`'Auto-Classified Outcome (Project Intake Requests)'.*`), and set `nod_SubmittedBy` from `varAppUser.'Email Address (emailaddress)'` — these are best-practice examples for the rest of the app to match.

## scrReview

No Critical, High, or Medium findings. `OnVisible` correctly re-fetches the live record via `LookUp()` rather than trusting an inbound variable. `btnReviewSubmit.OnSelect` correctly implements the Draft → (IT Estimate | Submitted) routing logic, wrapped in `IfError()`, with `nod_SubmittedBy` set from `varAppUser.'Email Address (emailaddress)'`. This screen is a good reference implementation for the patterns other screens should follow.

## scrPMOQueue

**✅ RESOLVED — NITRO-005 (was Critical):** Undefined identifier `PMOStage`.

```
// ❌ Was (scrPMOQueue.txt:435, prior commit)
varIsPMO And Not(IsBlank(varSelectedPMORecord)) And PMOStage = "NeedsEstimate",
```
```
// ✅ Now (commit 24d4d62)
varIsPMO And Not(IsBlank(varSelectedPMORecord)) And varPMOStage = "NeedsEstimate",
```
**Status:** Confirmed fixed.

**🟠 STILL PRESENT — NITRO-006 (High, downgraded from Critical, partially fixed):** Wrong column name in `RemoveIf()` — 1 of 3 occurrences fixed.

```
// ✅ Fixed at btnSubmitToIT.OnSelect (≈line 468)
RemoveIf(colPMOTriageQueue, 'Project Intake Request' = _patched.'Project Intake Request');
```
```
// ❌ Still present at the reclassification-confirm OnSelect (≈line 628)
RemoveIf(colPMOTriageQueue, 'IntakeID Request' = _patched.'IntakeID Request');
```
```
// ❌ Still present, embedded inside btnReturnToSubmitter.OnSelect's inline-quoted string (≈line 516, see NITRO-014)
RemoveIf(colPMOTriageQueue, 'IntakeID Request' = _patched.'IntakeID Request');
```
**Impact:** `'IntakeID Request'` is not a real column — the canonical relationship column is `'Project Intake Request'`, confirmed via scrIntakeWizard, scrReview, and the now-fixed `btnSubmitToIT` occurrence on this same screen. The two remaining occurrences will still error or silently fail to remove the just-actioned row from the local queue collection.

**✅ RESOLVED — NITRO-013 (was High):** Placeholder literals in nav component props.

```
// ❌ Was (scrPMOQueue.txt:659, 670, prior commit)
EnvironmentLabel: ="Text"
CurrentScreen: ="Text"
Fill: =RGBA(1, 81, 52, 1)
```
```
// ✅ Now (commit 24d4d62)
EnvironmentLabel: =varEnvironmentLabel
CurrentScreen: ="queue"
Fill: =gblTheme.ColorBrandAccent
```
**Status:** Confirmed fixed — also resolves NITRO-012's hardcoded RGBA on the same component.

**✅ RESOLVED — NITRO-015 (was High):** No explicit role-gate redirect set in `OnVisible`.

```
// ✅ Now (scrPMOQueue.txt:8-73, commit 24d4d62)
If(
    Not(varAppReady), true,
    If(
        Not(varIsAdmin Or varIsPMO Or varIsITEstimateManager),
        Set(varRedirectToHome, true),
        Set(varRedirectToHome, false);
        Concurrent( ... three queue queries ... );
        If(Not(IsEmpty(colPMOTriageQueue)), Set(varSelectedPMORecord, First(colPMOTriageQueue)), Set(varSelectedPMORecord, Blank()))
    )
)
```
**Status:** Confirmed fixed and paren-balance independently verified (3 nested `If()`s opened, 3 closed by the `)))` at the end of the block). Functionally correct, though the indentation is inconsistent (mixed 8/20-space levels) — see NITRO-035 (Low, cosmetic only).

**🔵 NEW — NITRO-035 (Low):** Indentation in the new `OnVisible` role-gate is inconsistent (the `Concurrent(` block and its children sit at a different indent level than the surrounding `If()`s). Cosmetic only; reformat for readability/diffability.

**🟡 STILL PRESENT — NITRO-014 (Medium, unchanged):** `btnReturnToSubmitter.OnSelect` is still an inline-quoted multi-line string instead of `|-` block style, and this same string still contains the unfixed `'IntakeID Request'` bug (see NITRO-006 above).

**Known Issues verified against scrPMOQueue (re-review):** K-04 — **Resolved**, unchanged. K-05 — **Resolved**, unchanged. K-06 (Navigate in OnVisible) — **Resolved**, the new role-gate logic does not call `Navigate()` directly. K-10 — **Resolved**, the `Concurrent()` block survived the role-gate refactor intact. K-11 — **Resolved** on this screen (NITRO-012 fixed via NITRO-013's edit).

## scrITEstimate

**Functionality — NITRO-007 (Critical):** Same wrong-column-name `RemoveIf()` bug as scrPMOQueue.

```
// ❌ Current (scrITEstimate.txt:227)
RemoveIf(colEstimateQueue, 'IntakeID Request' = varProjectRec.'IntakeID Request');
```
```
// ✅ Fixed
RemoveIf(colEstimateQueue, 'Project Intake Request' = varProjectRec.'Project Intake Request');
```

**✅ RESOLVED — NITRO-017 (was Critical):** Gallery selection used to hydrate from `ThisItem`.

```
// ❌ Was (scrITEstimate.txt:337-339, prior commit)
Set(varProjectRec,            ThisItem);
Set(varSelectedIntakeRequest, ThisItem);
Set(varITEstimate,            ThisItem);
```
```
// ✅ Now (commit 24d4d62)
With({_rec: LookUp('Project Intake Requests',
        nod_projectintakerequestid = ThisItem.nod_projectintakerequestid)},
    Set(varProjectRec,            _rec);
    Set(varSelectedIntakeRequest, _rec);
    Set(varITEstimate,            _rec)
)
```
**Status:** Confirmed fixed; K-03's underlying staleness risk is now closed on this screen.

**🟠 NEW — NITRO-031 (High):** Modal scrims lost transparency when their `Fill` was "fixed" off of hardcoded RGBA.

```
// ❌ Was (semi-transparent black scrims, prior commit)
Fill: =RGBA(0, 0, 0, 0.45)   // submitting-spinner overlay, ≈line 1032
Fill: =RGBA(0, 0, 0, 0.35)   // estimate-input modal overlay, ≈line 1568
Fill: =RGBA(0, 0, 0, 0.5)    // notes-panel modal overlay, ≈line 2020
```
```
// ❌ Now (commit 24d4d62 — opaque, no alpha channel)
Fill: =gblTheme.ColorBorderStrong   // ≈line 1035 — RGBA(160,170,184,1), fully opaque
Fill: =gblTheme.ColorSurfaceAlt     // ≈line 1571 — RGBA(247,249,252,1), fully opaque
Fill: =Color.DarkGray               // ≈line 2023 — fully opaque
```
```
// ✅ Should be
Fill: =gblTheme.ColorScrim   // new token, e.g. RGBA(15, 23, 42, 0.5)
```
**Impact:** A modal scrim's entire purpose is to dim the screen behind the modal while staying see-through. None of the three substituted tokens carry an alpha channel, so all three modals (submitting spinner, estimate-input, notes panel) will now render as a solid opaque rectangle covering the whole screen instead of a translucent backdrop — a visible regression a user will notice immediately. This was an attempt to fix K-11 (hardcoded RGBA) that traded a design-system violation for a functional/visual one. Add a proper semi-transparent `gblTheme.ColorScrim` token instead of repurposing opaque surface/border tokens.

**🟠 STILL PRESENT — NITRO-019 (High, unchanged):** Hardcoded email addresses.

```
// ❌ Current (scrITEstimate.txt:208, 1336, 1357)
{ Cc: "dgornitsky@nationaloak.com", IsHtml: true }
...
{ Cc: "pmo@nationaloak.com", IsHtml: true }
```
**Impact:** Unchanged by this edit pass. Recipient addresses baked into formulas can't be changed per environment without editing and re-publishing the app; should be Environment Variables.

**🟡 STILL PRESENT, PARTIALLY IMPROVED — NITRO-018 (Medium):** Hardcoded `RGBA(...)` values — 8 of the original 15 occurrences were converted to `gblTheme` tokens (lines 463, 1032, 1119, 1198, 1225, 2020, 2093, 2116 in the prior commit are now tokenized). 7 remain unconverted: lines 463→fixed actually, recount confirms remaining at ≈1119(fixed)... see `findings_summary.md` for the exact current line list. Two of the "fixed" spots were instead converted to raw `Color.*` system constants rather than `gblTheme` tokens — logged separately as **NITRO-032 (Medium)** since that's a different (still non-compliant) pattern, not a true fix.

**🔴 NEW — NITRO-033 (Critical):** Typo'd Choice enum member introduced while fixing the status-label literal-mismatch risk (was NITRO-023).

```
// ❌ Current (scrITEstimate.txt, Color/Fill Switch on lblEstimateStatus, ≈lines 399-415)
Switch(
    ThisItem.'IT Estimate Status',
    'IT Estimate Status (Project Intake Requests)'.'Information Needeed',  gblTheme.ColorDanger,
    ...
)
```
```
// ✅ Fixed
'IT Estimate Status (Project Intake Requests)'.'Information Needed',  gblTheme.ColorDanger,
```
**Impact:** "Needeed" is very likely a typo of "Needed". A reference to an enum member that doesn't exactly match a real Choice option label is a hard authoring error in Power Apps Studio — this formula may fail to save/publish at all, or (if Studio is lenient) silently evaluate to an error/blank for that branch. This is a strict regression: the prior version used a string literal that merely risked silently not matching; this version risks not compiling.

**🟡 NEW — NITRO-034 (Medium):** Within the same `lblEstimateStatus` control, `BorderColor`/`BorderThickness` were left on the old string-literal comparison while `Color`/`Fill` were converted to enum members — an inconsistency introduced by only half-applying the fix.

```
// Still using a string literal (≈lines 386, 395)
If(Text(ThisItem.'IT Estimate Status') = "In Review", ...)
// While the sibling properties on the same control now use:
Switch(ThisItem.'IT Estimate Status', 'IT Estimate Status (Project Intake Requests)'.'In Review', ...)
```

**Functionality — NITRO-020 (Medium, unchanged):** Inconsistent audit identity field — now `varAppUser.Name` at lines 218 and 1302, `varCurrentSystemUserId` at line 1350. The majority/minority flipped but the inconsistency itself persists.

**Naming — NITRO-022 (Medium, unchanged):** Default control names retained: `Rectangle3`, `Rectangle6`, `Rectangle7`, `Rectangle7_2`, `Rectangle9`, `Rectangle10`, `Rectangle11`, `DataGrid2`.

**Design System — NITRO-021 (Low, unchanged):** `btnSendEmail.BasePaletteColor: =gblColorText` (line 1318) still uses the legacy shim instead of `gblTheme.ColorTextPrimary`.

**YAML Syntax — NITRO-024 (Low, unchanged):** `btnCloseEstimateModal.OnSelect` still begins with a stray blank `=` line before its first statement.

**Positive finding:** `Timer1` correctly implements the role-gate redirect Timer pattern, and `Collect(colEstimateAudit, ...)` only ever writes to a local/session collection — never directly to `'Project Intake Audit'` in Dataverse, consistent with K-07 being resolved app-wide. Unaffected by this edit pass.

## scrAdminQuestions

**✅ PARTIALLY RESOLVED — NITRO-025 (was Medium):** This screen used to exclusively use legacy non-`Modern*` controls. `ddAdminQType` has now been migrated:

```
// ❌ Was
Control: DropDown@0.0.45
Properties: { ..., LayoutMinHeight: =44 }
```
```
// ✅ Now
Control: ModernCombobox@1.1.0
Properties: { ..., ItemDisplayText: =ThisItem.Value, LayoutMinHeight: =16 }
```
**Status:** One control converted; `Button@0.0.45`, `Toggle@1.1.5`, and `Rectangle@2.3.0` elsewhere on the screen are still legacy. Migration is in progress, not complete — keeping this open at Medium until the rest of the screen follows.

**Positive finding:** `OnVisible` correctly implements the role-gate + `varRedirectToHome` flag pattern, paired with `tmrAdminRedirect` (`AutoStart: =true`, `Start: =varRedirectToHome`) — this is the cleanest example of the Timer-redirect pattern in the app and should be the template scrPMOQueue is brought into line with (NITRO-015). All Patch() calls to `'Intake Questions'` and `'Intake Question Options'` are wrapped in `IfError()`. No Critical or High findings on this screen.

## scrTestHarness

`scrTestHarness.txt` is in scope for this review but **does not exist anywhere in the repository** — confirmed via directory listing of `/home/user/NITRO/screens` (only `App.OnStart.txt` and the 7 named screens are present). Per the app's own constraints, this screen should exist in Dev for UAT and only be excluded from the **managed Prod export**. Its total absence from version control means either (a) it was correctly removed and this is a non-issue for Prod-readiness but a gap in Dev source tracking, or (b) the export used for this review is itself incomplete. This is logged as a Critical/Missing-Source-File finding (NITRO-026) because it cannot be distinguished from the repository alone — it needs to be confirmed against the live Studio solution.

## Known Issues — Status Update

*(updated for re-review commit `24d4d62`)*

| ID | Issue | Status | Evidence |
|---|---|---|---|
| K-01 | `varCurrentUser` legacy duplicate role-lookup block in App.OnStart | **Resolved** | No occurrence of `varCurrentUser` anywhere in the repository |
| K-02 | Role flags use `Coalesce(x, false)` | **Resolved** | App.OnStart.txt:207-210 now uses the bare `varAppUser.X = true` pattern; the previously-flagged `If(...,true,false)` wrapper is gone |
| K-03 | scrITEstimate `galEstimateRequests.OnSelect` uses `LookUp()` inside `Patch()`'s 2nd parameter | **Resolved** | `galEstimateRequests.OnSelect` now hydrates via `With({_rec: LookUp(...)}, Set(varProjectRec, _rec); ...)` instead of `ThisItem`; no `LookUp()`-inside-`Patch()` found anywhere |
| K-04 | scrPMOQueue `btnApproveRoute.OnSelect` missing `IfError()` | **Resolved** | Patch call is correctly wrapped in `IfError()` |
| K-05 | scrPMOQueue `galQueue.OnSelect` uses `ThisItem` instead of live `LookUp()` | **Resolved everywhere for the navigation pointer; one residual gap on scrMyRequests** | scrPMOQueue, scrHome (NITRO-011), and scrITEstimate (NITRO-017) are fully resolved. scrMyRequests now also re-fetches via `LookUp()` for `varSelectedRequest`/`varProjectRec`, but its `colReviewBasics`/`colReviewBenefits`/`colReviewGovernance` `ClearCollect()` calls still read stale `ThisItem` (NITRO-036) |
| K-06 | `Navigate()` called directly inside `OnVisible` | **Resolved** | No direct `Navigate()` found in any `OnVisible`, including the newly-edited scrPMOQueue/scrIntakeWizard `OnVisible` blocks |
| K-07 | Direct `Patch()` to `'Project Intake Audit'` from Canvas | **Resolved** | Unaffected by this edit pass; still no direct Patch to `'Project Intake Audit'` found anywhere |
| K-08 | `scrTestHarness` exists in solution and must be excluded from Prod export | **Cannot verify** | File still absent from repository entirely (NITRO-026) |
| K-09 | Unmanaged dependency on `cre6c_sharedcommondataserviceforapps_35703` | **Still Present (assumed)** | Solution-package-level concern; not visible in screen YAML, unaffected by this edit pass |
| K-10 | `Concurrent()` not used where independent OnVisible queries exist | **Resolved (where checked)** | Unaffected by this edit pass; scrPMOQueue's 3-query `Concurrent()` block is preserved correctly inside the new role-gate nesting |
| K-11 | Hardcoded RGBA/font sizes instead of `gblTheme.*` | **Still Present, substantially improved** | scrPMOQueue's occurrence (NITRO-012) is resolved. scrITEstimate's 3 modal scrims now use the new `gblTheme.ColorScrim` token (NITRO-031 resolved). 7 hardcoded RGBA literals remain (NITRO-018), and 2 of 3 raw-`Color.*` swaps remain unconverted (NITRO-032, down from 2 of 2 spots) |
| K-12 | `varSelectedApprovalRec` initialized as `""` instead of `Blank()` | **Resolved** | App.OnStart's `Set(varSelectedApprovalRec, Blank());` line is now commented out entirely (NITRO-030 fixed) — functionally equivalent to `Blank()` since an un-`Set()` global defaults to blank |

## Production Readiness Assessment

*(updated for re-review commit `5b8f41f`)*

- [ ] All Critical findings resolved — **No**, 1 open (NITRO-026, unrelated to this edit pass) — down from 3
- [ ] All High findings resolved or accepted with documented risk — **No**, 3 open (NITRO-014, NITRO-027, NITRO-036) — down from 6
- [ ] scrTestHarness removed from solution — **Unconfirmed**, file absent from repo entirely
- [ ] No hardcoded environment values — **Yes**, scrITEstimate's hardcoded emails (NITRO-019) now resolved via `varPMOEmail` Environment Variable
- [ ] Solution Checker critical (connection reference) resolved — **No**, K-09 still open per stated baseline
- [ ] Role flag pattern confirmed as `x = true` across all screens — **Yes**, unchanged from prior pass (NITRO-001 resolved)
- [ ] All Patch() calls wrapped in IfError() — **Yes**, on every Patch() call observed across all screens
- [ ] No Navigate() in OnVisible — **Yes**, confirmed app-wide
- [ ] No direct Canvas Patch to 'Project Intake Audit' — **Yes**, confirmed app-wide
- [ ] Concurrent() used for parallel OnVisible queries — **Yes**, where applicable; App.OnStart's `Concurrent()` also now loads `varPMOEmail` in parallel with the existing choices/user queries
- [ ] Delegation verified for all gallery Filter() calls — **Partially**, unchanged from prior pass — all delegable
- [ ] No new syntax errors introduced by recent edits — **Yes**, both prior-pass regressions (NITRO-030, NITRO-033) are now cleanly fixed; no new paren-balance or enum-typo issues found in this pass's diff

**Verdict: Materially closer to production-ready.** Both Critical regressions from the previous pass (NITRO-030, NITRO-033) are genuinely resolved, not just relocated, and seven more High/Medium findings closed alongside them (NITRO-006, NITRO-019, NITRO-020, NITRO-021, NITRO-031, NITRO-034, NITRO-009, NITRO-025, NITRO-024). The only Critical left open (NITRO-026) is an ALM/missing-file concern that predates and is unrelated to any of the three edit passes. The one new finding this pass introduced (NITRO-036) is a real but narrow functional gap — not a regression of comparable severity to what came before. Remaining blockers before merge are NITRO-036 (stale review-summary data) and the repo-hygiene items (NITRO-026, NITRO-027); everything else is cosmetic/cleanup.

## Recommended Fix Order

*(updated for re-review commit `5b8f41f`)*

1. **scrMyRequests** — rebind `colReviewBasics`/`colReviewBenefits`/`colReviewGovernance` to the freshly-fetched record instead of stale `ThisItem` (NITRO-036) — this is the only remaining functional gap of consequence.
2. **scrTestHarness / Repository hygiene** — confirm the true status of `scrTestHarness` (NITRO-026) and resolve the `screens/AllScreens` divergence (NITRO-027, changed again in this pass).
3. **scrPMOQueue** — convert `btnReturnToSubmitter.OnSelect` from an inline-quoted string to `\|-` block style (NITRO-014).
4. **scrITEstimate** — finish converting the remaining 7 hardcoded RGBA literals (NITRO-018) and the 2 remaining raw-`Color.*` constants (`Color.DarkOrange`, `Color.LightGoldenRodYellow` — NITRO-032) to `gblTheme` tokens.
5. **Cosmetic/consistency pass** — default control renames on scrITEstimate (NITRO-022), legacy shim cleanup in App.OnStart (NITRO-028), scrPMOQueue `OnVisible` indentation (NITRO-035), scrMyRequests `OnSelect` indentation (NITRO-037).
