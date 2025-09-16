# Manual Test Cases: BE | Scenario Builder | "Select All" filter values cause Constraint to not be saved

- **Feature area**: Constraint Builder in Scenario Builder
- **Bug summary**: When using "Select All" for a multi-value field (e.g., `SCAC`) in constraints, saving fails with backend validation error: `filters[0].value_string_array must not be empty`.
- **Severity**: High
- **Environments**: Test in QA and Staging; verify API and UI.

---

## Preconditions and test data
- **Preconditions**
  - An event with Lane Responses Collected and ability to create Scenarios.
  - User role with permission to create/edit scenarios and constraints.
- **Test data**
  - A multi-value participant field (e.g., `SCAC`) with at least 5+ values across multiple pages if pagination exists.
  - Lanes available to apply lane-level filters (All lanes and subsets).
  - Other constraint locations supporting "Select All":
    - Only Award to Specified Participants
    - Never Award to Specified Participants
    - Favor Specified Participants
    - Apply constraint to a subset of lanes
    - Apply constraint to a subset of participants

---

## Test matrix

### TC-01 Reproduce current bug (Goal > Only Award > Select All)
- **Purpose**: Confirm current failure with "Select All".
- **Steps**
  1. Create a Custom Scenario.
  2. Add a Constraint in Goal: select "Only Award to Specified Participants".
  3. Field: choose `SCAC` (or any multi-value field).
  4. In Value dropdown, choose "Select All".
  5. Save Goal.
  6. In Lane filter, select "Apply constraint to All Lanes" and Save Lane Filter.
  7. Click Save and Validate on the Constraint.
- **Expected (after fix)**: Constraint saves; applies to all `SCAC` values; no validation error.
- **Actual (current)**: Error returned with `"filters[0].value_string_array" must not be empty.` and `code: BAD_USER_INPUT`.
- **Result**: Should fail pre-fix, pass post-fix.

### TC-02 Control: Manual multi-select values (no "Select All")
- **Purpose**: Ensure baseline save works when explicitly picking values.
- **Steps**: Same as TC-01 but manually select 3+ individual `SCAC` values (do not use "Select All"), then save and validate.
- **Expected**: Constraint saves successfully; values persisted in UI and API.
- **Result**: Pass (both pre- and post-fix).

### TC-03 Toggle from subset to "Select All"
- **Purpose**: Verify switching to "Select All" generates the correct payload.
- **Steps**
  1. Configure as in TC-02 with 2–3 values selected; save.
  2. Edit the constraint; change to "Select All".
  3. Save and validate.
- **Expected**: Saves successfully; API reflects either all values enumerated or a dedicated server-side "all" handling without empty arrays.
- **Result**: Pass post-fix.

### TC-04 "Select All" then deselect one value
- **Purpose**: Ensure "Select All" state transitions to specific values correctly.
- **Steps**
  1. Choose "Select All".
  2. Deselect one value from the set.
  3. Save and validate.
- **Expected**: Saves with all-minus-one; UI displays partial selection; API includes explicit list excluding the deselected value.
- **Result**: Pass post-fix.

### TC-05 Lane scope: Apply to All Lanes
- **Purpose**: Validate lane filter scope with "Select All".
- **Steps**: Create constraint as in TC-01 with "Select All"; set Lane filter "Apply to All Lanes"; save and validate.
- **Expected**: Saves; constraint applies across all lanes; no validation error.
- **Result**: Pass post-fix.

### TC-06 Lane scope: Apply to subset of lanes
- **Purpose**: Validate "Select All" with lane subset scoping.
- **Steps**
  1. Create constraint with "Select All".
  2. In Lane filter, choose "Apply constraint to a subset of lanes", select 3+ lanes.
  3. Save and validate.
- **Expected**: Saves; constraint applies only to chosen lanes; no error.
- **Result**: Pass post-fix.

### TC-07 "Never Award to Specified Participants" with "Select All"
- **Purpose**: Regression across other constraint types.
- **Steps**: Create this constraint; choose `SCAC`; pick "Select All"; save and validate.
- **Expected**: Saves; applies to all participants; no error.
- **Result**: Pass post-fix.

### TC-08 "Favor Specified Participants" with "Select All"
- **Purpose**: Regression on favor-type constraint.
- **Steps**: Same pattern as TC-07.
- **Expected**: Saves; applies to all; no error.
- **Result**: Pass post-fix.

### TC-09 "Apply constraint to a subset of participants" with "Select All"
- **Purpose**: Regression on participant subset scoping.
- **Steps**
  1. Create constraint type that targets participant subset.
  2. Choose `SCAC`; "Select All".
  3. Save and validate.
- **Expected**: Saves; applies to all; no error.
- **Result**: Pass post-fix.

### TC-10 API payload validation when using "Select All"
- **Purpose**: Ensure backend receives a valid payload.
- **Steps**
  1. With network inspector or logs, perform TC-01 post-fix.
  2. Capture request body for `createConstraint`.
- **Expected**:
  - Request contains either:
    - `value_string_array` populated with all values; or
    - a supported flag/indicator representing "all" that backend translates into values; but not an empty `value_string_array`.
  - Response success; persisted constraint id returned.
- **Result**: Pass post-fix.

### TC-11 Pagination: "Select All" across multiple pages
- **Purpose**: Ensure "Select All" truly includes off-screen values.
- **Steps**
  1. Open values list where `SCAC` spans multiple pages.
  2. Click "Select All" (global select-all, not page-only, if supported).
  3. Save and validate.
- **Expected**: All values across pages included; no omission; saves successfully.
- **Result**: Pass post-fix.

### TC-12 Re-open/edit persistence
- **Purpose**: Ensure persistence after save.
- **Steps**
  1. Save a constraint with "Select All".
  2. Re-open scenario later; edit the constraint.
- **Expected**: UI shows "Select All" state accurately; counts match current catalog; saving again continues to succeed.
- **Result**: Pass post-fix.

### TC-13 Removal and re-creation
- **Purpose**: Ensure no residual state issues.
- **Steps**
  1. Delete a "Select All" constraint.
  2. Recreate it with "Select All".
- **Expected**: Both delete and re-create succeed.
- **Result**: Pass post-fix.

### TC-14 Mixed field types sanity check
- **Purpose**: Ensure fix generalizes beyond `SCAC`.
- **Steps**: Repeat TC-01 for another multi-value field (e.g., Region, Mode).
- **Expected**: Saves; no error.
- **Result**: Pass post-fix.

### TC-15 Permission/role check
- **Purpose**: Ensure behavior isn’t role-dependent.
- **Steps**: Run TC-01 as another role permitted to edit scenarios.
- **Expected**: Same results; no validation error post-fix.
- **Result**: Pass post-fix.

---

## Acceptance criteria (post-fix)
- Saving any constraint that uses "Select All" never produces `value_string_array must not be empty`.
- Constraint correctly applies to all values in scope and persists across re-open/edit.
- API request is valid: does not send empty `value_string_array` for "Select All"; either enumerates all values or uses a supported "all" indicator handled server-side.
- Works consistently in:
  - Only Award to Specified Participants
  - Never Award to Specified Participants
  - Favor Specified Participants
  - Apply constraint to a subset of lanes
  - Apply constraint to a subset of participants
- "Select All" covers paginated values, not just those on the current page.

---

## Notes for investigation
- If UI uses a compact payload for "all", ensure backend translates it to a non-empty list or supports the compact flag; avoid sending empty arrays.
- Verify that value catalogs are fetched prior to save so the client can enumerate when required.

## References
- Source issue: BE | Scenario Builder | "Select All" filter values cause Constraint to not be saved.
- Related repository placeholder: [RFP-10434](https://github.com/olevkivska-emerge/RFP-10434)
