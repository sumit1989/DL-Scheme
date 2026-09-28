# DL Partner / Scheme Configuration – Frontend Jira Stories

**Epic:** DL-EPIC – Migrate DL partner/scheme configuration from Oracle (FinnOne) to MongoDB, provide maker-checker UI, and sync every approved change to FinnOne through Kafka.

**Reference:** `DL_Migration_Tables_and_Questions.xlsx`
- The field list comes from the "Additional Fields (Fin1 Api)" sheet.
- Field order comes from "Field Sequence in UI (Fin1 API)".
- Dropdown sources are in "Dropdown Masters & Queries".

**UI tabs:** New Partner · Partner Configuration · Partner Details · Servicing

**Principles**
- The UI is **driven by metadata**. Fields, order, mandatory flags and conditional rules come from the backend (BE-04), not hardcoded screens.
- Only **Maker Input** fields are editable.
- **Autopopulate** fields are shown read-only.
- **Default Value** and **Backend** fields are not shown.
- Every change goes through maker-checker. Nothing reaches FinnOne until a checker approves it.

**Story points** are suggested values and should be re-estimated in grooming.

---

## FE-01 · Scheme list / search screen

**As a** user **I want** to search DL schemes **so that** I can open, copy or check them.

**Scope**
- Filters:
  - partner code / name
  - scheme ID
  - product
  - journey code
  - maker-checker status
  - sync status
- Grid columns: scheme ID, partner, product, journey, last updated by/on, maker-checker status, sync status badge (PENDING / SYNCED / FAILED).
- Actions: View, Edit, Copy, History, and Change Scheme ID (UAT only).
- Paginated, sortable.

**Acceptance criteria**
- Filters can be combined.
- Each action button is shown only to roles allowed to use it.

**API:** BE-13  **SP:** 3

---

## FE-02 · Tabbed form shell driven by metadata

**As a** developer **I want** one reusable form engine **so that** new fields or rule changes don't need UI code changes.

**Scope**
- Render the 4 tabs and their fields from the metadata API (label, type, order, M/NM/CM, length, default).
- Supported input types:
  - Checkbox
  - Numeric
  - Alphanumeric
  - Drop Down
  - Multi-select
  - Semicolon-separated list input (for due dates)
- Rule engine for conditional mandatory, show/hide and enable/disable (see FE-03 to FE-05).
- Field-level validation: mandatory, max length, numeric range, IFSC format. Show backend field errors next to the field they belong to.
- Mark a tab with an error indicator when it has invalid fields.
- Warn the user about unsaved changes when they leave the page.

**Acceptance criteria**
- Changing metadata or rules in backend config changes the UI without a release.
- A save error from the backend highlights the correct field.

**API:** BE-04  **SP:** 8

---

## FE-03 · Partner Configuration tab

**Scope:** the maker-input fields of `FINTECH_PARTNER_CONFIG_MST`. Examples:
- Reference Scheme ID, Source Name, Product Flag, Journey Code, Customer Type
- Disbursal Stage, Upfront Validation and its checks (Maturity, Debit Freeze, LAN Status, NPA)
- Max/Min Loan, Parent Scheme ID
- State Business Head, Sales Officer, RO
- BSR Code, Partner Pool Account and IFSC, Disbursal Bank ID, Disbursal To, Auto Disbursal
- Mandate Merchant Code, Mandate Status, Part Payment API, KYC cumulative amount/flag, Auth API
- Offer, Tenure 60, Mandate vs Disburse Account check, Customer Delinquency, Subvent Interest
- Backdated Foreclosure, Max CRN DPD, Full Cancellation Auto Receipt, Non-Partner Txn callback
- DPD counters, Interest Charged From, Part Cancellation and its Impact On, Excess Amount Adjustment
- Age Flag with Min/Max Age, Tenure in Days, Auto process EOD, Settlement Callback
- Asset Details API, Penny Reversal, Pennydrop API

**Conditional rules**

| Condition | Effect |
|---|---|
| Disbursal To = P | Partner Pool Account and IFSC become mandatory, and the IFSC must be valid |
| Disbursal To = M | Mandate Merchant Code becomes mandatory |
| Upfront Validation flag = Y | Debit Freeze Validation becomes mandatory |
| Product = RC | Parent Scheme ID becomes mandatory |
| Part Cancellation Applicable = Y | Part Cancellation Impact On becomes mandatory |
| Age Flag = Y | Min Age and Max Age become mandatory |
| Backdated Foreclosure = Y | Its dependent field becomes mandatory |

**Cascading dropdowns**
- Sales Officer → State Business Head and RO. The RO list is refreshed when the SO changes.
- BSR Code → Industry and Sub-industry, filled automatically and read-only.

**Acceptance criteria:** every rule in the table works both ways. Unchecking a flag clears or hides its dependent fields.  **SP:** 8

---

## FE-04 · Partner Details tab

**Scope**
- **Due dates:** a semicolon-separated list with a chip preview, one chip per due date.
- **Cut-off Date:** dropdown with values 1–31.
- **Pre-disbursal Charge ID:** multi-select. For each selected charge, the user sets Charge sourced from Partner, Disbursement Including Charges and APR Flag.
- **SMS checkboxes:** Loan Agreement SMS, Sanction Letter SMS, Loan Disbursement SMS, Sanction SMS, plus Template Type and Partner Name.
- **Payment flags:**
  - Amount Disbursal at Mandate Account
  - Disbursement in Bank ID other than 121, which shows Product Code and Disbursal Bank ID (autopopulated)
  - Payment allowed within 6 months
  - Scheme Description (autopopulated)
- **Other Fields:** Bureau Reporting Product (dropdown, mandatory) and Partner Name in Bureau Reports (mandatory).

**Acceptance criteria**
- Removing a charge or unchecking an SMS box shows a "will be removed" note in the review summary before saving.
- Invalid due-date formats are rejected before saving.

**SP:** 5

---

## FE-05 · Servicing tab

**Scope**
- Partner Name.
- **Auto refund:** Initiate Refund API Flag and Auto Refund Flag. When Auto Refund = Y, the AR Bank Account, AR IFSC, AR MICR and AR Bank ID fields are enabled.
- **Charge posting:** Charge Posting API flag. When it is Y, the Post Disbursal Charge ID dropdown is shown.
- **Collections:** Payment Posting API, Subvention Collection Bank ID (mandatory if Subvent flag = Y), Source System (autopopulated), Partner's Legal Name, Collection Bank ID.
- **Recon:** Recon Report (1055) checkbox.

**Acceptance criteria**
- The AR bank fields are disabled and cleared when Auto Refund = N.
- The rules match the "Additional Validation" column in the reference sheet.

**SP:** 5

---

## FE-06 · New Partner tab – clone XML config

**Scope**
- Inputs: Source Name, Journey Code (CO-BNPL / CO-BNPL-EMI / CO-EMI / DR-EMI), Customer Type (NTB / ETB), and a "Copy from partner" dropdown.
- Before saving, show how many XML validation rows will be cloned.

**Acceptance criteria:** the backend's duplicate-clone error is shown clearly.  **API:** BE-12  **SP:** 2

---

## FE-07 · Reusable dropdown components

**Scope**
- One generic dropdown component that supports:
  - DB master data (with search)
  - static lookup lists
  - multi-select
  - cascading (reloads when its parent field changes)
- Show code and description (e.g. `BANKID – BANKDESC`), and save only the code.
- Handle loading, empty-list and error states.
- Cache master data for the length of the session.

**Acceptance criteria**
- All 27 dropdowns in the reference sheet use this component.
- A dependent dropdown resets when its parent value changes.

**API:** BE-03  **SP:** 3

---

## FE-08 · Copy scheme (Reference Scheme ID)

**As a** maker **I want** to prefill a new scheme from an existing one.

**Scope**
- On Partner Configuration, selecting a Reference Scheme ID calls the copy-template API.
- That call fills the Partner Configuration, Partner Details and Servicing tabs.
- Show a banner: "Copied from scheme X – review before saving".
- Cleared fields are left empty for the user to enter: new scheme details and any status fields.
- If the form already has data, ask the user to confirm before overwriting it.

**Acceptance criteria**
- All copyable fields are prefilled.
- Saving creates a new scheme and leaves the source scheme unchanged.

**API:** BE-10  **SP:** 3

---

## FE-09 · Maker-checker screens

**Scope**
- **Maker:** Save as draft and Submit for approval. Show the review summary (added / changed / removed rows per tab) before submitting.
- **Checker:**
  - a pending approvals queue;
  - a comparison view (live vs proposed) with changed fields highlighted;
  - Approve;
  - Reject with mandatory remarks.
- The maker cannot see Approve on their own submissions.
- Show a rejected draft back to the maker with the checker's remarks.

**Acceptance criteria:** the comparison view shows every table-level change, including rows added or removed by checkboxes and multi-selects.  **API:** BE-07  **SP:** 5

---

## FE-10 · Sync status and retry

**Scope**
- A sync status badge on the list and on the scheme header: PENDING / SYNCED / FAILED.
- When the status is FAILED, show the error message from FinnOne and a Retry button (authorised roles only).
- While the status is PENDING, block edits or show a warning, depending on the rule decided in BE-09.

**Acceptance criteria:** the status refreshes on its own (polling or manual refresh) after approval.  **API:** BE-09  **SP:** 2

---

## FE-11 · Change Scheme ID screen – UAT only

**Scope**
- Visible only when the feature flag is on (UAT) and the user has the UAT-Admin role. Hidden in PROD.
- Fields: Old Scheme ID (dropdown), New Scheme ID (text), Reason (mandatory).
- Check the new ID as the user types: the backend confirms it is **not present in LEA_SCHEME_M** or in any related table.
- A preview lists the affected tables and row counts (12 tables).
- Submit for checker approval, then track sync status.
- Show a confirmation dialog warning that the change updates all related tables.

**Acceptance criteria**
- The screen is not shown in PROD builds.
- Submit stays disabled until validation passes.

**API:** BE-11  **SP:** 3

---

## FE-12 · Audit history view

**Scope**
- Per scheme: a timeline of changes (who, when, maker/checker, old → new values per field), with the sync result for each event.
- Scheme ID renames are included.

**API:** BE-07  **SP:** 2

---

## FE-13 · Roles, access and masking

**Scope**
- Show or hide actions by role: Maker, Checker, Viewer, UAT-Admin.
- Mask account numbers on view screens (e.g. `XXXXXX1234`) unless the user is in edit mode and has permission.

**Acceptance criteria:** menus and buttons match the role matrix, and bank account numbers are masked in every read-only view.  **SP:** 2

---

### Suggested order
FE-02 → FE-07 → FE-03 → FE-04 → FE-05 → FE-01 → FE-09 → FE-08 → FE-06 → FE-10 → FE-11 → FE-12 → FE-13

### Definition of Done (all frontend stories)
- Works against the UAT APIs.
- Field rules match the reference sheet.
- Unit tests for the rule engine and the dropdown component.
- Responsive at 1366px and wider.
- Accessibility basics: labels and keyboard navigation.
- Code reviewed.
- Demoed to the BA.
