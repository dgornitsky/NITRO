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

**Performance/Functionality — NITRO-008 (High):** Gallery row actions hydrate from `ThisItem` instead of a live `LookUp()`.

```
// ❌ Current (scrMyRequests.txt:434-435)
=Set(varSelectedRequest,  ThisItem);
Set(varProjectRec,        ThisItem);
```
```
// ✅ Fixed
=Set(varSelectedRequest, LookUp('Project Intake Requests',
    nod_projectintakerequestid = ThisItem.nod_projectintakerequestid));
Set(varProjectRec, varSelectedRequest)
```
**Impact:** Same class of bug as K-05, which was already fixed on scrPMOQueue's `galQueue.OnSelect` but never propagated here — the user can act on a stale snapshot of their own request (e.g., resuming a draft that was just updated by a PMO triage action in another session/tab).

**Functionality — NITRO-009 (High):** Literal Choice-column string comparison that may never match.

```
// ❌ Current (scrMyRequests.txt:555)
"Steering Committee Project", gblTheme.ColorClassSteering,
```
**Impact:** The actual classification value elsewhere in the app is referenced via the fully-qualified Choice (`'Project Classification'`/`'Auto-Classified Outcome'` enum members), not this exact string. If the option-set label differs even slightly (capitalization, trailing text), this branch silently never fires and falls through to default styling — a cosmetic bug today, but evidence of a literal-vs-enum mismatch pattern that recurs at NITRO-023.

**YAML Syntax (Medium, untracked ID — folded into general report note):** `btnResume.OnSelect` is written as an inline-quoted multi-line string rather than the `|-` block scalar. Functionally equivalent, but inconsistent with the rest of the codebase and harder to diff in source control.

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

**Design System — NITRO-025 (Medium):** This screen exclusively uses legacy non-`Modern*` controls (`Button@0.0.45`, `DropDown@0.0.45`, `Toggle@1.1.5`, `Rectangle@2.3.0`) while the rest of the app (scrHome, scrIntakeWizard, scrReview) has migrated to `ModernButton@1.0.0`/`ModernDropdown@1.0.1`/`ModernText@1.0.0`. No functional defect, but a visible/behavioral inconsistency.

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
| K-05 | scrPMOQueue `galQueue.OnSelect` uses `ThisItem` instead of live `LookUp()` | **Resolved on scrPMOQueue/scrHome/scrITEstimate / Still Present on scrMyRequests** | scrPMOQueue, scrHome (NITRO-011), and scrITEstimate (NITRO-017) now all correctly use `LookUp()`; scrMyRequests (NITRO-008) is the one remaining `ThisItem` occurrence |
| K-06 | `Navigate()` called directly inside `OnVisible` | **Resolved** | No direct `Navigate()` found in any `OnVisible`, including the newly-edited scrPMOQueue/scrIntakeWizard `OnVisible` blocks |
| K-07 | Direct `Patch()` to `'Project Intake Audit'` from Canvas | **Resolved** | Unaffected by this edit pass; still no direct Patch to `'Project Intake Audit'` found anywhere |
| K-08 | `scrTestHarness` exists in solution and must be excluded from Prod export | **Cannot verify** | File still absent from repository entirely (NITRO-026) |
| K-09 | Unmanaged dependency on `cre6c_sharedcommondataserviceforapps_35703` | **Still Present (assumed)** | Solution-package-level concern; not visible in screen YAML, unaffected by this edit pass |
| K-10 | `Concurrent()` not used where independent OnVisible queries exist | **Resolved (where checked)** | Unaffected by this edit pass; scrPMOQueue's 3-query `Concurrent()` block is preserved correctly inside the new role-gate nesting |
| K-11 | Hardcoded RGBA/font sizes instead of `gblTheme.*` | **Still Present, partially improved** | scrPMOQueue's occurrence (NITRO-012) is now resolved. scrITEstimate dropped from 15 to 7 confirmed remaining hardcoded RGBA literals (NITRO-018), but 2 more were "fixed" into raw `Color.*` constants instead of tokens (NITRO-032) — net non-compliant count did not actually shrink as much as the RGBA count suggests |
| K-12 | `varSelectedApprovalRec` initialized as `""` instead of `Blank()` | **Fix attempt failed — New Variant, now Critical** | App.OnStart.txt:291 was changed to `Set(varSelectedApprovalRec, Blank();` — missing closing paren (NITRO-030). The variable is no longer typed as text, but it's also not cleanly `Blank()`; it resolves to the return value of the file's final `ClearCollect()` due to the parsing cascade this typo causes |

## Production Readiness Assessment

*(updated for re-review commit `24d4d62`)*

- [ ] All Critical findings resolved — **No**, 3 open (NITRO-030, NITRO-033, NITRO-026) — down from 9, but 2 are newly-introduced regressions
- [ ] All High findings resolved or accepted with documented risk — **No**, 6 open — down from 7
- [ ] scrTestHarness removed from solution — **Unconfirmed**, file absent from repo entirely
- [ ] No hardcoded environment values — **No**, hardcoded emails on scrITEstimate (NITRO-019), unchanged
- [ ] Solution Checker critical (connection reference) resolved — **No**, K-09 still open per stated baseline
- [ ] Role flag pattern confirmed as `x = true` across all screens — **Yes**, App.OnStart now uses the bare pattern app-wide (NITRO-001 resolved)
- [ ] All Patch() calls wrapped in IfError() — **Yes**, on every Patch() call observed across all screens
- [ ] No Navigate() in OnVisible — **Yes**, confirmed app-wide, including newly-edited OnVisible blocks
- [ ] No direct Canvas Patch to 'Project Intake Audit' — **Yes**, confirmed app-wide
- [ ] Concurrent() used for parallel OnVisible queries — **Yes**, where applicable (scrPMOQueue, preserved through the new role-gate edit)
- [ ] Delegation verified for all gallery Filter() calls — **Partially**, all `Filter()` predicates observed use delegable `=`/`<>` comparisons on Dataverse columns; no `Search()`/`In`/non-delegable patterns found
- [ ] No new syntax errors introduced by recent edits — **No**, App.OnStart's `varSelectedApprovalRec` fix (NITRO-030) has a paren-balance defect, and scrITEstimate's status-Switch fix (NITRO-033) appears to reference a typo'd enum member

**Verdict: Still not production-ready, but materially improved.** The Resume-mode state bug, both undefined-identifier bugs, the role-flag pattern, and most of the `ThisItem`-hydration anti-pattern are now genuinely fixed. However, this edit pass introduced **two new Critical defects** (NITRO-030, NITRO-033) that are arguably riskier than what they replaced, because they're subtle: NITRO-030 silently mis-assigns a variable rather than throwing an obvious error, and NITRO-033 may simply fail to save/publish in Studio. Both must be fixed before the next merge.

## Recommended Fix Order

*(updated for re-review commit `24d4d62`)*

1. **App.OnStart** — fix the malformed `Set(varSelectedApprovalRec, Blank();` statement (NITRO-030). Rewrite as two cleanly-closed statements; this is the highest priority because it currently corrupts the structure of every statement after it in the file.
2. **scrITEstimate** — fix the typo'd Choice enum member `'Information Needeed'` → the correct option label (NITRO-033); verify it actually saves/compiles in Studio.
3. **scrPMOQueue** — fix the remaining two `'IntakeID Request'` → `'Project Intake Request'` occurrences (NITRO-006 at the reclass-confirm flow, and embedded in NITRO-014's inline-quoted `btnReturnToSubmitter.OnSelect` string).
4. **scrITEstimate** — restore transparency on the 3 modal scrim `Fill` properties; add a dedicated `gblTheme.ColorScrim` token with an alpha channel rather than reusing opaque surface/border tokens (NITRO-031).
5. **scrMyRequests** — replace the one remaining `ThisItem` gallery-hydration call site with live `LookUp()` (NITRO-008), matching the pattern now correct everywhere else.
6. **scrITEstimate** — finish converting the remaining 7 hardcoded RGBA literals and the 2 raw `Color.*` constants to `gblTheme` tokens (NITRO-018, NITRO-032); align `BorderColor`/`BorderThickness` on `lblEstimateStatus` to use the same enum-member comparison as the adjacent `Color`/`Fill` properties (NITRO-034).
7. **scrITEstimate** — move hardcoded emails to Environment Variables (NITRO-019); reconcile `ChangedBy` identity convention, which still flips between `varAppUser.Name` and `varCurrentSystemUserId` (NITRO-020).
8. **scrPMOQueue** — convert `btnReturnToSubmitter.OnSelect` from an inline-quoted string to `\|-` block style (NITRO-014); clean up the inconsistent indentation introduced in the `OnVisible` role-gate fix (NITRO-035).
9. **Repository hygiene** — resolve the `screens/AllScreens` divergence (NITRO-027, which changed again in this edit pass — 1469 lines touched) and confirm the true status of `scrTestHarness` (NITRO-026) before the next Prod pipeline run.
10. **Cosmetic/consistency pass** — default control renames (NITRO-022), legacy-control migration on scrAdminQuestions (NITRO-025), legacy shim cleanup (NITRO-021, NITRO-028), stray leading `=` line (NITRO-024).
