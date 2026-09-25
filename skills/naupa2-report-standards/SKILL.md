---
name: naupa2-report-standards
description: Reference and self-audit checklist for generating correct NAUPA II unclaimed-property holder report files (625-byte fixed-width electronic format) and the human-readable report/cover-sheet package that accompanies them. Use whenever building, editing, or reviewing code that produces NAUPA II output, holder report PDFs, or due-diligence filing packages — or when asked to judge/audit an existing NAUPA report generator against the real standard.
---

# NAUPA II Report Generation Standard

Compiled from the official NAUPA (National Association of Unclaimed Property
Administrators, unclaimed.org) specification and cross-referenced against
live research (state treasury sites, reporting-software vendor docs, and
the official 2019 and 2025 spec PDFs, which are word-for-word identical
except two citation URLs). Sources are cited inline; treat every claim here
as sourced, not assumed.

## 1. What NAUPA II actually is

A **625-byte fixed-width record format** for reporting unclaimed property
electronically. First byte of every record = `TR-CODE` (1 digit), remaining
624 bytes hold that record type's fields.

| TR-CODE | Record | Purpose |
|---|---|---|
| 1 | HOLDER | One per report — the reporting company |
| 2 | PROPERTY | One per property/owner |
| 3 | PROPADD | Additional owner beyond the primary (shares `PROP-SEQUENCE-NUMBER`) |
| 4, 7, 8 | *(reserved)* | Not used |
| 5 | SECURITIES | Optional detail attached to a PROPERTY record (CUSIP, delivery method) |
| 6 | TANGIBLE | Optional detail attached to a PROPERTY record (safe-deposit box, etc.) |
| 9 | SUMINFO | One per report, last record — totals for reconciliation |

Rules that apply file-wide:
- Numeric (`N`) fields: **right-justified, zero-filled**, decimal point *assumed*, never entered (`$253.73` → `0000025373`).
- Character (`C`) fields: **left-justified, space-filled**.
- Records separated by **CR/LF**.
- File is plain ASCII, no embedded control characters.
- A file may hold **multiple Holder Reports** (Option 1: one file per report; Option 2: multiple HOLDER→SUMINFO blocks concatenated) — this mechanism is for **multiple holder entities reporting to the same state**, never for combining multiple states into one file. (Confirmed directly in spec text.)

Source: official spec, https://unclaimed.org/wp-content/uploads/NAUPAStandardElectronicFileFormat-11.20.19.pdf (2019) — verified identical to the 2025 revision except two citation-URL updates (ISO country-code link, census.gov NAICS link).

## 2. Field acceptable-values — the rule that's easy to get wrong

Every name field (`HOLDER-NAME`, `PROP-OWNER-NAME-LAST/FIRST/MIDDLE/PREFIX/SUFFIX/TITLE`, `PADD-OWNER-NAME-*`) has the *exact same* acceptable-value annotation in the spec:

> **(V) = A-Z / 0-9 / Space / &**

That's it. **No periods, no hyphens, no apostrophes, no commas.** Concretely:

- **Punctuation is never allowed** — spec: *"Punctuation should never be used under any circumstances (periods, commas, apostrophes, etc.)"*
- **"The" moves to the end**: `"The Smith Company"` → `"Smith Company The"` (verbatim rule, applies to HOLDER-NAME, PROP-OWNER-NAME-LAST, and PADD-OWNER-NAME-LAST identically).
- **Don't abbreviate the first word**: "American" not "Amer.", "National" not "Natl.", "first" never "1st" — except when the number is part of a registered name/logo ("A1 Inc", "84 Lumber").
- **Space out initials**: "J J Reynolds", not "JJ Reynolds".
- **Aggregate records**: owner name literally `"AGGREGATE"`; unknown owners literally `"UNKNOWN"`.

A generic sanitizer that strips *most* punctuation but leaves periods/hyphens/dashes through is **non-compliant** — the allowed set is exactly `A-Z0-9 &`, nothing else.

## 3. Filing convention: one file per state, always

Confirmed by direct research, not assumption:

- The multi-holder-report mechanism in §1 is scoped to **one destination state**; it has never been a mechanism for spanning states.
- **Reciprocity agreements** exist (a holder files to its domicile state, which forwards property to ~20 partner states in some cases) but are optional, state-specific, and declining in use — not a substitute for per-state filing as the default assumption.
- **SURCH** (States' Unclaimed Retirement Clearing House, NAST/NAUPA-backed) is a genuine one-filing-for-many-states clearinghouse, but scoped **only** to retirement-plan uncashed distribution checks ≤$1,000 — not general holder reporting (payroll, AP, dormant accounts, gift cards, etc.).
- **A general-corporation NAUPA file must never mix properties from multiple states.** If your data spans states, group by effective state and emit one file (and one cover sheet / PDF) per state.

State submission is increasingly portal-based, not mail-in media: NY (secure upload / manual web app, mag tape and email explicitly rejected as of Aug 2025), OH (unclaimedfunds.ohio.gov, paperless), TX (claimittexas.gov), DE (unclaimedproperty.delaware.gov), KY (dedicated portal), CA (GoReport.sco.ca.gov). Each state runs its own portal — there is no shared multi-state upload platform for general reporting.

Sources: unclaimed.org spec (multi-holder-report text); Wisconsin DOR Pub. 82 (reciprocity); nast.org/urrp (SURCH); NY OSC, Ohio DOR, TX Comptroller, DE UP portal, KY Treasury (per-state portal pages).

## 4. Aggregate reporting — thresholds vary by state, this is not a fixed rule

"Collapsing" small properties into one `AGGREGATE` PROPERTY record (owner name `"AGGREGATE"`, `PROP-OWNER-TYPE-CODE = "AP"`) is real and spec-supported, but it is **gated by a per-state dollar threshold**, not a blanket choice:

- **$50 is the modal threshold** (AL, AZ, CT, DE, GA, HI, IN, IA, MI, MO, MT, NJ, NM, OH, PA, RI, SC, UT, WI, WY).
- **$100**: AK, KS, MD, MA, MN, MS, VA.
- **Lower/stricter**: IL $5, FL <$10, LA $10, CO/VT/ND/TX/CA $25.
- **Nevada disallows aggregation entirely** — full owner detail required regardless of amount.
- **Oregon** requires state authorization before aggregating at all; **Idaho** has no aggregation provision.

A generator that aggregates "everything matching state+property-code" without checking the destination state's threshold (or without checking that the state allows it at all) will produce a non-compliant file for states like Nevada. This must be surfaced clearly if the actual thresholds aren't modeled per state.

Source: https://unclaimed.org/property-type-aggregate-amount/, NJ Treasury FAQ, cross-checked against Indiana/Illinois/Florida state guidance.

## 5. Code tables (confirmed current as of this research)

- **Relationship codes**: AD, AG, AF, AN, BF, CP, CN, CF, DF, ES, EX, FB, GR, HE, IN, JT, JS, TC, JE, OR, OT, PD, PA, PO, RE, SO, TE, UG, UT, UN, UF (effective 09/26/2013, unchanged).
- **Ownership codes**: `AP` = Aggregate Property, `OT` = all owners except aggregate/unknown, `UN` = Unknown Owner.
- **Deduction/Withholding**: DW, IW, MC, SW, TW, ZZ. **Addition**: DR, DV, IN, ME, SP, ZZ. **Paid/Deletion**: ER, RO, RS, ZZ.
- **Security delivery**: ACCOUNT, DTC, PHYSICAL, UNT.
- **Property type codes**: ~330 codes (AC/CK/MS/SC/IN/IR/MI/CT/HS/CS/SD/TR/UT families, ZZZZ = unidentified) — unchanged in NAUPA II as of this research.

**Forward note (NAUPA III, not yet required)**: draft NAUPA III material (v1.1 Aug 2025 → v1.4 May 2026) is being partly backported into NAUPA II already in some states: `VC01/VC02/VC03` (virtual currency — native units / liquidated / cash balance at exchange), `MS012` (sports betting), `CS01-07` (expanded college savings), `DC01-12` (SURCH/retirement distributions). **Not universally accepted** — a Tennessee chart updated June 2026 still shows the classic code set with no VC codes. Don't assume these codes are safe to emit without confirming the destination state accepts them.

## 6. Common real-world rejection reasons

- Missing mandatory (`M`) fields — files are rejected/returned whole, not partially accepted.
- State-specific rules layered on top of the shared format (e.g., some states require SSN on payroll-type properties where the national spec only marks it "required if known").
- Numeric fields not zero-filled/right-justified, character fields not space-filled/left-justified.
- Wrong property-type or relationship codes.
- Missed due-diligence-letter timing relative to the report due date.

Source: UPPO ("Identify and eliminate common reporting errors"), Eisen "How to Create a NAUPA File", Florida Treasure Hunt spec notes.

## 7. Self-audit checklist

When reviewing or building a NAUPA II generator, check each of these explicitly:

- [ ] **One file/report per effective state** — never combined across states.
- [ ] Name fields restricted to exactly `A-Z 0-9 space &` — no periods, hyphens, apostrophes, commas.
- [ ] `"The X"` companies re-ordered to `"X The"`.
- [ ] Aggregate (`AP`) records only used where the destination state actually permits aggregation, and (if modeled) only below that state's specific dollar threshold — otherwise flag it as unverified rather than silently aggregating.
- [ ] Numeric fields zero-filled + right-justified; character fields space-filled + left-justified; every field padded to its exact spec width (verify byte length after building each record).
- [ ] `TR-CODE` values match the table in §1; SUMINFO's 6 dollar fields (Reported/Deduction/Advertised/Addition/Deletion/Remitted) reconcile (`Advertised = Reported − Deduction`; `Remitted = Advertised + Addition − Deletion`).
- [ ] Mandatory (`M`) fields are never blank/zero unless the spec's own conditional logic says so (e.g., deduction-type is only required *if* deduction amount > 0).
- [ ] Any code emitted (property type, relationship, ownership, deduction/addition/deletion) is drawn from the current official table, and anything NAUPA-III-only (VC/MS012/CS0x/DC0x) is flagged, not assumed accepted.
- [ ] A human-readable cover sheet/report accompanying the file includes: holder name, FEIN, contact, report year/type, property count, total amount, negative-report indicator, and a certification/signature block — these are state-required even though NAUPA itself doesn't publish one canonical cover-sheet template (each state has its own, e.g. FL DFS-UP-111, CA UFS-1).
