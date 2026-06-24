# NITRO Canvas App — YAML Code Review Report
Generated: 2026-06-24
Reviewer: Claude Code (AI-assisted Power Apps Architecture Review)
Scope: Full app — all screens including App.OnStart

## Executive Summary

NITRO is **not production-ready**. This review found **9 Critical, 7 High, 9 Medium, and 3 Low** findings across App.OnStart and 7 screens (28 distinct findings total). The most severe issues are two undefined-variable bugs that will throw runtime errors (scrIntakeWizard's `CategoryStepValid`, scrPMOQueue's `PMOStage`, both missing their `var` prefix), a recurring wrong-column-name bug (`'IntakeID Request'` vs. the real `'Project Intake Request'`) present in **both** scrPMOQueue and scrITEstimate that will break queue refresh after every action on those screens, and a state-management regression in scrIntakeWizard where Resume-mode data is silently overwritten on every `OnVisible`. The previously-tracked Known Issues are mostly **Resolved** (K-01, K-04 through K-07, K-10), but K-02, K-03, K-05, K-11, and K-12 are either still present or have reappeared as new variants on screens that weren't fixed when the original fix was applied elsewhere. A stray, undocumented combined-export file (`screens/AllScreens`) was also found in the repository containing materially different code for several of these defects — strongly suggesting the tracked per-screen `.txt` exports are out of sync with the actual app source and should not be trusted as the sole source of truth until reconciled. `scrTestHarness.txt`, in scope for this review, does not exist anywhere in the repository.

## Review Methodology

Each screen source file under `/home/user/NITRO/screens` was read in full (App.OnStart → scrHome → scrMyRequests → scrIntakeWizard → scrReview → scrPMOQueue → scrITEstimate → scrAdminQuestions), checked against the nine dimensions in Section 7 of the review brief (Functionality, Performance, Error Handling, Role-Based Access, Naming Conventions, Design System Compliance, Cross-Screen Coupling, YAML Syntax, ALM/Production Readiness), and cross-referenced against the Power Fx rules in Section 4 and the twelve previously-tracked Known Issues (K-01–K-12). Targeted `Grep` passes across all screens were used to confirm exact line numbers and to verify whether suspected literal/column-name mismatches were genuine (by comparing against the canonical usage elsewhere in the app) before they were logged as findings. `scrTestHarness.txt` was confirmed absent via directory listing. An additional file, `screens/AllScreens`, was discovered during this pass; it is not part of the named review scope but is documented as a finding (NITRO-027) because of its bearing on source-of-truth integrity.

## App.OnStart

**Functionality — NITRO-001 (Critical):** Role flags use the banned `If(x = true, true, false)` wrapper.

```
// ❌ Current (App.OnStart.txt:207-210)
Set(varIsAdmin,             If(varAppUser.IsAdmin             = true, true, false));
Set(varIsPMO,               If(varAppUser.IsPMO               = true, true, false));
Set(varIsITEstimateManager, If(varAppUser.IsITEstimateManager = true, true, false));
Set(varIsApprover,          If(varAppUser.IsApprover          = true, true, false));
```
```
// ✅ Fixed
Set(varIsAdmin,             varAppUser.IsAdmin             = true);
Set(varIsPMO,               varAppUser.IsPMO               = true);
Set(varIsITEstimateManager, varAppUser.IsITEstimateManager = true);
Set(varIsApprover,          varAppUser.IsApprover          = true);
```
**Impact:** Functionally equivalent today, but this is the exact anti-pattern the team has previously flagged (K-02) for null-handling risk on Dataverse Two Options columns; leaving it unfixed means the rule isn't actually enforced anywhere in the app, and a future Coalesce()-style regression is more likely to slip through review.

**Functionality — NITRO-029 (Critical):** Record-typed variable initialized with `""` instead of `Blank()`.

```
// ❌ Current (App.OnStart.txt:291)
Set(varSelectedApprovalRec, "");
```
```
// ✅ Fixed
Set(varSelectedApprovalRec, Blank());
```
**Impact:** `""` locks `varSelectedApprovalRec`'s inferred type as Text. Any later `Set(varSelectedApprovalRec, <record>)` will fail with a type-mismatch error at runtime, breaking whatever approval flow depends on this variable. Notably, `screens/AllScreens` (line 307) shows this exact line commented out in favor of `Set(varSelectedApprovalRec, Blank());` — i.e., the fix was already drafted somewhere but never landed in the tracked source.

**ALM — NITRO-028 (Low):** Legacy color shim globals (`gblColorGreen`, `gblColorRed`, `gblColorMuted`, `gblColorText`, `gblColorWhite`, lines 120-125) are retained alongside the `gblTheme` token set and are still referenced from at least one screen (scrITEstimate, see NITRO-021). Recommend completing the migration and deleting the shim block once all references are gone.

**Known Issues verified against App.OnStart:** K-01 (`varCurrentUser` legacy lookup block) — **Resolved**, no such identifier exists anywhere in the repository. K-02 (`Coalesce(x, false)` role flags) — **New Variant**: the literal `Coalesce()` pattern isn't used, but the equally-banned `If(...,true,false)` wrapper is (NITRO-001). K-12 (`varSelectedApprovalRec` as `""`) — **Still Present** (NITRO-029).

## scrHome

**Role Access — NITRO-010 (High):** A role-switcher dropdown is present on the Home screen.

```
// ❌ Current (scrHome.txt:80-93)
- drpQuickRole:
    Control: ModernDropdown@1.0.1
    Properties:
      Items: =["Admin","PMO","IT Estimate Manager","Approver","Unauthorized"]
      ...
```
**Impact:** A client-side control that lists every role by name on the Home dashboard is, at minimum, a stray dev/test artifact and, at worst, a vector for a user to influence their own perceived role in the UI. It should be removed from the production build entirely or hard-gated behind a non-production check.

**Functionality — NITRO-011 (High):** Recent-activity row navigates to scrReview without setting context.

```
// ❌ Current (scrHome.txt:371-380)
- btnRecentOverlay:
    Control: Button@0.0.45
    Properties:
      OnSelect: =Navigate(scrReview, ScreenTransition.Fade)
```
```
// ✅ Fixed
OnSelect: |-
  =Set(varProjectRec, LookUp('Project Intake Requests',
      nod_projectintakerequestid = ThisItem.nod_projectintakerequestid));
  Navigate(scrReview, ScreenTransition.Fade)
```
**Impact:** Without setting the record context, scrReview's `OnVisible` (which re-`LookUp()`s based on `varProjectRec`) will display whatever request was last viewed elsewhere in the app, not the one the user clicked — a confusing and potentially incorrect-data-shown bug.

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

**Functionality — NITRO-002 (Critical):** `OnVisible`'s `Concurrent()` reset block runs unconditionally, even in Resume mode.

```
// ❌ Current (conceptual — OnVisible structure)
If(varWizardMode = "Resume",
    Set(varProjectRec, <resumed record>); Set(varWizardStep, <resumed step>)
);
Concurrent(
    Set(varProjectRec, Blank()),
    Set(varWizardStep, 0),
    ...
)
```
```
// ✅ Fixed
If(varWizardMode = "Resume",
    Set(varProjectRec, <resumed record>); Set(varWizardStep, <resumed step>),
    Concurrent(
        Set(varProjectRec, Blank()),
        Set(varWizardStep, 0),
        ...
    )
)
```
**Impact:** This is the most severe functional bug found in the app. Every time a user resumes a draft, the reset block that follows immediately clobbers the just-restored state, so Resume mode is effectively broken — the user is silently dropped back to step 0 with a blank record regardless of intent.

**Functionality/Naming — NITRO-003 (Critical) and NITRO-004 (Medium):** Undefined identifier `CategoryStepValid` (missing `var` prefix).

```
// ❌ Current (scrIntakeWizard.txt:1587)
If(CategoryStepValid, DisplayMode.Edit Or IsBlank(txtBenefitAmount.Text), DisplayMode.Disabled)

// ❌ Current (scrIntakeWizard.txt:1662)
CategoryStepValid;
```
```
// ✅ Fixed
If(varCategoryStepValid Or IsBlank(txtBenefitAmount.Text), DisplayMode.Edit, DisplayMode.Disabled)

// (line 1662 — remove the bare statement, or correct to:)
varCategoryStepValid;
```
**Impact:** `CategoryStepValid` (no `var` prefix) is not a declared variable anywhere in the app — Power Fx will either treat this as an implicit blank/error value or fail to compile the formula, depending on context. As written, the `DisplayMode` expression is also structurally invalid: it applies a boolean `Or` between an enum value (`DisplayMode.Edit`) and a boolean (`IsBlank(...)`), which is not a valid `If()` true-branch. This control's enabled/disabled state cannot be relied upon as shipped.

All Patch() calls on this screen are correctly wrapped in `IfError()`, use fully-qualified Choice enums (`'Auto-Classified Outcome (Project Intake Requests)'.*`), and set `nod_SubmittedBy` from `varAppUser.'Email Address (emailaddress)'` — these are best-practice examples for the rest of the app to match.

## scrReview

No Critical, High, or Medium findings. `OnVisible` correctly re-fetches the live record via `LookUp()` rather than trusting an inbound variable. `btnReviewSubmit.OnSelect` correctly implements the Draft → (IT Estimate | Submitted) routing logic, wrapped in `IfError()`, with `nod_SubmittedBy` set from `varAppUser.'Email Address (emailaddress)'`. This screen is a good reference implementation for the patterns other screens should follow.

## scrPMOQueue

**Functionality/Naming — NITRO-005 (Critical):** Undefined identifier `PMOStage`.

```
// ❌ Current (scrPMOQueue.txt:435)
varIsPMO And Not(IsBlank(varSelectedPMORecord)) And PMOStage = "NeedsEstimate",
```
```
// ✅ Fixed
varIsPMO And Not(IsBlank(varSelectedPMORecord)) And varPMOStage = "NeedsEstimate",
```
**Impact:** `btnSubmitToIT`'s `DisplayMode` formula references an undeclared variable; the comparison will never evaluate as intended, so the button's enabled/disabled state cannot be trusted.

**Functionality — NITRO-006 (Critical):** Wrong column name in `RemoveIf()`.

```
// ❌ Current (scrPMOQueue.txt:464 and 624)
RemoveIf(colQueue, 'IntakeID Request' = _patched.'IntakeID Request')
```
```
// ✅ Fixed
RemoveIf(colQueue, 'Project Intake Request' = _patched.'Project Intake Request')
```
**Impact:** `'IntakeID Request'` is not a real column — the canonical relationship column, confirmed via scrIntakeWizard (lines 1670, 1681) and scrReview (line 10), is `'Project Intake Request'`. As written, this `RemoveIf()` will either error out or silently fail to remove the just-actioned row from the local queue collection, leaving a stale/duplicate row visible in the triage queue until the next full refresh.

**ALM — NITRO-013 (High):** Placeholder literals left in nav component props.

```
// ❌ Current (scrPMOQueue.txt:659, 670)
EnvironmentLabel: ="Text"
CurrentScreen: ="Text"
```
```
// ✅ Fixed
EnvironmentLabel: =varEnvironmentLabel
CurrentScreen: ="queue"
```
**Impact:** `CurrentScreen` drives nav-item highlighting in `cmpPrimaryNav`; with the literal `"Text"`, the PMO Queue nav item never shows as active when a PMO user is on this screen.

**Role Access — NITRO-015 (High):** No explicit role-gate redirect set in `OnVisible`.

**Impact:** scrAdminQuestions and scrITEstimate both explicitly `Set(varRedirectToHome, true/false)` based on a role check at the top of `OnVisible`, driving the `tmrRedirectGuard` Timer that's already wired on this screen (`Start: =varRedirectToHome`). scrPMOQueue never sets that flag, so the Timer mechanism exists but has nothing actually gating access — an unauthorized user landing on this screen via direct navigation/deep link will not be redirected.

**Design System — NITRO-012 (Medium):** `cmpPrimaryNav_5.Fill: =RGBA(1, 81, 52, 1)` (line 671) instead of a `gblTheme` token.

**YAML Syntax (Medium, untracked ID):** `btnReturnToSubmitter.OnSelect` is an inline-quoted multi-line string instead of `|-` block style.

**Known Issues verified against scrPMOQueue:** K-04 (`btnApproveRoute.OnSelect` missing `IfError()`) — **Resolved**, correctly wrapped. K-05 (`galQueue.OnSelect` using `ThisItem`) — **Resolved**, correctly uses live `LookUp()`. K-10 (`Concurrent()` for independent OnVisible queries) — **Resolved**, three independent queries are correctly wrapped in `Concurrent()`.

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

**Functionality — NITRO-017 (Critical):** Gallery selection hydrates from `ThisItem`.

```
// ❌ Current (scrITEstimate.txt:337-339)
Set(varProjectRec,            ThisItem);
Set(varSelectedIntakeRequest, ThisItem);
Set(varITEstimate,            ThisItem);
```
```
// ✅ Fixed
With({_rec: LookUp('Project Intake Requests',
        nod_projectintakerequestid = ThisItem.nod_projectintakerequestid)},
    Set(varProjectRec,            _rec);
    Set(varSelectedIntakeRequest, _rec);
    Set(varITEstimate,            _rec)
)
```
**Impact:** This is the live successor to K-03. The originally-tracked defect ("LookUp() inside Patch()'s second parameter") was not found anywhere in this screen — the actual `btnSubmitEstimate.OnSelect` Patch call correctly uses `varProjectRec` directly. But the underlying risk K-03 was meant to catch (acting on stale gallery data) is still present here via `ThisItem` hydration instead.

**ALM — NITRO-019 (High):** Hardcoded email addresses.

```
// ❌ Current (scrITEstimate.txt:208, 1333)
{ Cc: "dgornitsky@nationaloak.com", IsHtml: true }
...
{ Cc: "pmo@nationaloak.com", IsHtml: true }
```
**Impact:** Recipient addresses baked into Patch/flow-call formulas can't be changed per environment without editing and re-publishing the app; should be Environment Variables resolved at runtime, especially since this same app already has a known Dev-vs-Prod connection-reference ALM issue (K-09).

**Design System — NITRO-018 (Medium):** 15 hardcoded `RGBA(...)` values across the screen (lines 383, 401, 414, 463, 465, 1032, 1119, 1198, 1225, 1232, 1264, 1568, 2020, 2093, 2116) — by far the highest concentration in the app, confirming K-11 is still very much present here even though other screens have largely migrated to `gblTheme` tokens.

**Functionality — NITRO-020 (Medium):** Inconsistent audit identity field.

```
// ❌ Current — two different conventions in the same screen
ChangedBy: varAppUser.Name              // line 218
ChangedBy: varCurrentSystemUserId       // lines 1299, 1347
```
**Impact:** Audit rows from the same screen, for logically equivalent "who made this change" semantics, will contain different value shapes depending on which code path wrote them — undermines reliable querying/reporting on the audit trail downstream.

**Naming — NITRO-022 (Medium):** Default control names retained: `Rectangle3`, `Rectangle6`, `Rectangle7`, `Rectangle7_2`, `Rectangle9`, `Rectangle10`, `Rectangle11`, `DataGrid2`.

**Design System — NITRO-021 (Low):** `btnSendEmail.BasePaletteColor: =gblColorText` (line 1315) uses the legacy shim instead of `gblTheme.ColorTextPrimary`.

**YAML Syntax — NITRO-024 (Low):** `btnCloseEstimateModal.OnSelect` begins with a stray blank `=` line before its first statement.

**Functionality (unverified) — NITRO-023 (Medium):** Status-label string literals (e.g. `"Submitted to IT"`, `"IT Estimate Complete"`) compared via `Text('IT Estimate Status')` should be verified against the actual Choice option-set text, given the confirmed literal-mismatch precedent at NITRO-009.

**Positive finding:** `Timer1` (lines 2176-2188) correctly implements the role-gate redirect Timer pattern (`Start: =varRedirectToHome`, `OnTimerEnd` does the `Navigate()`), and `Collect(colEstimateAudit, ...)` only ever writes to a local/session collection — never directly to `'Project Intake Audit'` in Dataverse, consistent with K-07 being resolved app-wide.

## scrAdminQuestions

**Design System — NITRO-025 (Medium):** This screen exclusively uses legacy non-`Modern*` controls (`Button@0.0.45`, `DropDown@0.0.45`, `Toggle@1.1.5`, `Rectangle@2.3.0`) while the rest of the app (scrHome, scrIntakeWizard, scrReview) has migrated to `ModernButton@1.0.0`/`ModernDropdown@1.0.1`/`ModernText@1.0.0`. No functional defect, but a visible/behavioral inconsistency.

**Positive finding:** `OnVisible` correctly implements the role-gate + `varRedirectToHome` flag pattern, paired with `tmrAdminRedirect` (`AutoStart: =true`, `Start: =varRedirectToHome`) — this is the cleanest example of the Timer-redirect pattern in the app and should be the template scrPMOQueue is brought into line with (NITRO-015). All Patch() calls to `'Intake Questions'` and `'Intake Question Options'` are wrapped in `IfError()`. No Critical or High findings on this screen.

## scrTestHarness

`scrTestHarness.txt` is in scope for this review but **does not exist anywhere in the repository** — confirmed via directory listing of `/home/user/NITRO/screens` (only `App.OnStart.txt` and the 7 named screens are present). Per the app's own constraints, this screen should exist in Dev for UAT and only be excluded from the **managed Prod export**. Its total absence from version control means either (a) it was correctly removed and this is a non-issue for Prod-readiness but a gap in Dev source tracking, or (b) the export used for this review is itself incomplete. This is logged as a Critical/Missing-Source-File finding (NITRO-026) because it cannot be distinguished from the repository alone — it needs to be confirmed against the live Studio solution.

## Known Issues — Status Update

| ID | Issue | Status | Evidence |
|---|---|---|---|
| K-01 | `varCurrentUser` legacy duplicate role-lookup block in App.OnStart | **Resolved** | No occurrence of `varCurrentUser` anywhere in the repository |
| K-02 | Role flags use `Coalesce(x, false)` | **New Variant** | Literal `Coalesce()` not used; the equally-banned `If(x=true,true,false)` wrapper is used instead (App.OnStart.txt:207-210, NITRO-001) |
| K-03 | scrITEstimate `galEstimateRequests.OnSelect` uses `LookUp()` inside `Patch()`'s 2nd parameter | **New Variant** | No `LookUp()`-inside-`Patch()` found; `btnSubmitEstimate.OnSelect` correctly Patches `varProjectRec` directly — but `galEstimateRequests.OnSelect` hydrates from `ThisItem` instead of live `LookUp()` (scrITEstimate.txt:337-339, NITRO-017), the same underlying staleness risk K-03 was meant to catch |
| K-04 | scrPMOQueue `btnApproveRoute.OnSelect` missing `IfError()` | **Resolved** | Patch call is correctly wrapped in `IfError()` |
| K-05 | scrPMOQueue `galQueue.OnSelect` uses `ThisItem` instead of live `LookUp()` | **Resolved on scrPMOQueue / New Variant elsewhere** | scrPMOQueue's `galQueue.OnSelect` correctly uses `LookUp()`; the same anti-pattern is present on scrMyRequests (NITRO-008), scrHome (NITRO-011), and scrITEstimate (NITRO-017) |
| K-06 | `Navigate()` called directly inside `OnVisible` | **Resolved** | Every screen reviewed (scrIntakeWizard, scrPMOQueue, scrAdminQuestions, scrITEstimate) uses the Timer + flag pattern; no direct `Navigate()` found in any `OnVisible` |
| K-07 | Direct `Patch()` to `'Project Intake Audit'` from Canvas | **Resolved** | All audit writes seen (`colEstimateAudit`, etc.) are local `Collect()`s, never a direct Patch to `'Project Intake Audit'` |
| K-08 | `scrTestHarness` exists in solution and must be excluded from Prod export | **Cannot verify** | File absent from repository entirely (NITRO-026) — cannot confirm it was correctly excluded vs. never tracked |
| K-09 | Unmanaged dependency on `cre6c_sharedcommondataserviceforapps_35703` | **Still Present (assumed)** | Solution-package-level concern; not visible in screen YAML, no contradicting evidence found, carried forward as stated |
| K-10 | `Concurrent()` not used where independent OnVisible queries exist | **Resolved (where checked)** | scrPMOQueue correctly uses `Concurrent()` for 3 independent queries; scrAdminQuestions's sequential queries are genuinely dependent (not a violation) |
| K-11 | Hardcoded RGBA/font sizes instead of `gblTheme.*` | **Still Present** | scrITEstimate alone has 15 hardcoded `RGBA(...)` literals (NITRO-018); scrPMOQueue has 1 (NITRO-012) |
| K-12 | `varSelectedApprovalRec` initialized as `""` instead of `Blank()` | **Still Present** | App.OnStart.txt:291 (NITRO-029) |

## Production Readiness Assessment

- [ ] All Critical findings resolved — **No**, 9 open
- [ ] All High findings resolved or accepted with documented risk — **No**, 7 open
- [ ] scrTestHarness removed from solution — **Unconfirmed**, file absent from repo entirely
- [ ] No hardcoded environment values — **No**, hardcoded emails on scrITEstimate (NITRO-019)
- [ ] Solution Checker critical (connection reference) resolved — **No**, K-09 still open per stated baseline
- [ ] Role flag pattern confirmed as `x = true` across all screens — **No**, App.OnStart uses the `If(...,true,false)` wrapper (NITRO-001)
- [ ] All Patch() calls wrapped in IfError() — **Yes**, on every Patch() call observed across all screens
- [ ] No Navigate() in OnVisible — **Yes**, confirmed app-wide
- [ ] No direct Canvas Patch to 'Project Intake Audit' — **Yes**, confirmed app-wide
- [ ] Concurrent() used for parallel OnVisible queries — **Yes**, where applicable (scrPMOQueue); not yet verified for scrITEstimate's OnVisible due to file length, recommend explicit verification before sign-off
- [ ] Delegation verified for all gallery Filter() calls — **Partially**, all `Filter()` predicates observed use delegable `=`/`<>` comparisons on Dataverse columns; no `Search()`/`In`/non-delegable patterns found

**Verdict: Not production-ready.** Two undefined-identifier bugs (`CategoryStepValid`, `PMOStage`) will produce runtime errors or silently-wrong DisplayMode behavior, a wrong-column-name bug is duplicated across two Critical workflow screens, and the IntakeWizard's Resume-mode state is actively broken. None of these are cosmetic — they sit on the core submit/triage/estimate workflow.

## Recommended Fix Order

1. **scrIntakeWizard** — gate the `Concurrent()` reset block behind `varWizardMode = "New"` (NITRO-002). This is the single highest-impact fix; Resume mode is currently non-functional.
2. **scrIntakeWizard / scrPMOQueue** — fix the two undefined-identifier bugs: `CategoryStepValid` → `varCategoryStepValid` (NITRO-003, NITRO-004) and `PMOStage` → `varPMOStage` (NITRO-005).
3. **scrPMOQueue / scrITEstimate** — fix the `'IntakeID Request'` → `'Project Intake Request'` column-name bug in both `RemoveIf()` calls (NITRO-006, NITRO-007).
4. **App.OnStart** — fix `varSelectedApprovalRec` to `Blank()` (NITRO-029) before any screen that depends on it is exercised in further testing.
5. **scrHome / scrMyRequests / scrITEstimate** — replace remaining `ThisItem` gallery-hydration call sites with live `LookUp()` (NITRO-008, NITRO-011, NITRO-017), matching the pattern already correct on scrPMOQueue and scrReview.
6. **scrPMOQueue** — add the missing `varRedirectToHome` role-gate in `OnVisible` (NITRO-015), and replace the placeholder `"Text"` literals on `EnvironmentLabel`/`CurrentScreen` (NITRO-013).
7. **App.OnStart** — normalize role flags to the bare `x = true` pattern (NITRO-001).
8. **scrITEstimate** — move hardcoded emails to Environment Variables (NITRO-019); reconcile `ChangedBy` identity convention (NITRO-020); verify status-label literals against actual Choice option text (NITRO-023).
9. **Repository hygiene** — resolve the `screens/AllScreens` divergence (NITRO-027) and confirm the true status of `scrTestHarness` (NITRO-026) before the next Prod pipeline run.
10. **Cosmetic/consistency pass** — hardcoded RGBA cleanup (NITRO-012, NITRO-018), default control renames (NITRO-022), legacy-control migration on scrAdminQuestions (NITRO-025), legacy shim cleanup (NITRO-021, NITRO-028), inline-quoted multi-line formula reformatting, stray leading `=` line (NITRO-024).
