# Implementation Plan: Enhanced Day Assignments Workflow

## Overview

This task list provides a complete implementation roadmap for the Enhanced Day Assignments Workflow feature. The implementation is divided into 6 phases:

1. **State Management Setup** (Tasks 1-2): Initialize all required state variables and helper functions
2. **UI Component Development** (Tasks 3-8): Build weekday, date, and classification selectors + preview
3. **Core Logic & Validation** (Tasks 9-12): Implement selection logic, validation, and preview calculation
4. **API Integration** (Tasks 13-16): Orchestrate assignment API calls and handle responses
5. **Testing & Verification** (Tasks 17-28): Property-based tests and integration tests
6. **Integration & Finalization** (Tasks 29-31): Sidebar updates, accessibility, and deployment

---

## Tasks

### Phase 1: State Management Setup

- [x] 1. Initialize Advanced Assignment State Variables
  - Add `advancedSelectedWeekday`, `advancedSelectedDates`, `advancedSelectedClassification`, `advancedIsAssigning`, and `advancedMessage` state variables to CalendarSettings component
  - Add TypeScript/JSDoc comments documenting each state variable's type and purpose
  - Initialize states with proper defaults (null for weekday/classification, empty Set for dates, empty message object)
  - _Requirements: 2.1, 2.3, 3.3, 5.1, 6.1_
  - **Files to Modify**: `frontend/src/components/settings/CalendarSettings.jsx`

- [x] 2. Implement Helper Functions for Advanced Workflow
  - Implement `countWeekdayOccurrences()` - returns count of calendar days matching selected weekday
  - Implement `generateAssignmentSummary()` - formats combined preview text (e.g., "4 Mondays + 3 specific dates = 7 total days")
  - Implement `canAssign()` - boolean check for assignment eligibility
  - Implement `getValidationErrors()` - returns array of validation error messages
  - Implement `extractErrorMessage()` - extracts user-friendly error from API response
  - _Requirements: 4.3, 6.2, 6.4, 7.2, 7.3, 11.1_
  - **Files to Modify**: `frontend/src/components/settings/CalendarSettings.jsx`

---

### Phase 2: UI Component Development

- [ ] 3. Build Weekday Selection Dropdown Component
  - Create dropdown element with label "Weekday Selection"
  - Render 7 weekday options (Sunday through Saturday) with occurrence counts
  - Implement "Clear weekday" functionality when a weekday is selected
  - Add disabled state when no month is loaded
  - Add ARIA labels for accessibility (`aria-label`, `aria-describedby`)
  - Ensure responsive styling matches existing `.cal-grid` pattern
  - _Requirements: 2.1, 2.2, 2.3, 2.4, 2.5, 2.6, 7.1_
  - **Files to Modify**: `frontend/src/components/settings/CalendarSettings.jsx`, `frontend/src/components/settings/CalendarSettings.css`

- [ ] 4. Build Specific Dates Multi-Select Grid Component
  - Create grid container with CSS Grid layout (7 columns for weekdays)
  - Render date buttons (1 through max days in month) as interactive toggle buttons
  - Implement selection state toggling (Set-based) with visual highlight on selected dates
  - Add "Selected X dates" counter display below grid
  - Add "Clear All" button to empty date selection set
  - Add CSS classes for selected/unselected states (`.selected` modifier on `.cal-grid-item`)
  - Ensure performance: memoize component, render 31 items within 200ms target
  - _Requirements: 3.1, 3.2, 3.3, 3.4, 3.5, 3.6, 3.7, 7.1, 12.1_
  - **Files to Modify**: `frontend/src/components/settings/CalendarSettings.jsx`, `frontend/src/components/settings/CalendarSettings.css`

- [ ] 5. Build Classification Selection Dropdown Component
  - Create dropdown with label "Classification"
  - Display all available day types from `dayTypes` array
  - Show placeholder text "Select a classification" when nothing selected
  - Disable dropdown when no days are selected (weekday and dates both empty)
  - Add error state display when backend returns no classifications
  - Populate dropdown values with classification UUIDs
  - _Requirements: 5.1, 5.2, 5.3, 5.4, 5.5_
  - **Files to Modify**: `frontend/src/components/settings/CalendarSettings.jsx`

- [ ] 6. Build Combined Selection Preview Box Component
  - Create `.preview-box` container with dashed border (matching existing style)
  - Display main preview text with dynamic weekday/date counts and total
  - Display selected classification name in sub-text if classification selected
  - Update preview in real-time on any state change (weekday, dates, classification)
  - Show "No days selected" message when empty state
  - Format summary text: "4 Mondays + 3 specific dates = 7 total days" (or variations)
  - _Requirements: 4.3, 7.1, 7.2, 7.3_
  - **Files to Modify**: `frontend/src/components/settings/CalendarSettings.jsx`

- [ ] 7. Build Assignment Button and Status Message Display Component
  - Create "Assign Days" button with proper disabled/enabled states
  - Show loading spinner during API call (`advancedIsAssigning = true`)
  - Change button text to "Assigning..." during loading
  - Create status message container with role="status" aria-live="polite"
  - Style for success (green), error (red), warning (yellow) messages
  - Display error/warning messages with auto-dismiss logic for success (5s)
  - Show message timestamp for audit trail
  - _Requirements: 6.1, 6.6, 6.7, 7.1, 11.1, 11.2, 11.3, 12.3_
  - **Files to Modify**: `frontend/src/components/settings/CalendarSettings.jsx`, `frontend/src/components/settings/CalendarSettings.css`

- [ ] 8. Assemble Advanced Assignments UI Section
  - Wrap all components from Tasks 3-7 in a new `.panel` container
  - Add panel header with title "Advanced Day Assignments" and description
  - Position section below existing bulk grid interface in "Day Assignments" tab
  - Ensure all components are vertically stacked with proper spacing
  - Add disabled state overlay when month is not loaded
  - Verify responsive layout: stacks on mobile, grid on desktop
  - _Requirements: 1.5, 3.1, 5.1, 6.1, 9.1_
  - **Files to Modify**: `frontend/src/components/settings/CalendarSettings.jsx`, `frontend/src/components/settings/CalendarSettings.css`

---

### Phase 3: Core Logic & Validation

- [ ] 9. Implement Weekday Selection Handler
  - Create `handleWeekdayChange()` function that updates `advancedSelectedWeekday` state
  - Handle clearing: if current weekday is selected again, clear to null
  - On weekday selection, compute weekday count via `countWeekdayOccurrences()`
  - Verify specific date selection is NOT cleared (independent state)
  - Add event listener to weekday dropdown
  - _Requirements: 2.3, 2.4, 2.5, 2.6, 4.4_
  - **Files to Modify**: `frontend/src/components/settings/CalendarSettings.jsx`

- [ ] 10. Implement Specific Date Selection Handler
  - Create `handleToggleDate()` function that adds/removes date from `advancedSelectedDates` Set
  - Create `handleClearAllDates()` function that empties the Set
  - Verify idempotence: clicking same date twice returns to original state
  - Verify weekday selection is NOT affected by date operations
  - Update "Selected X dates" counter on each toggle
  - Add click event listeners to date button grid
  - _Requirements: 3.3, 3.4, 3.5, 3.6, 3.7, 4.5_
  - **Files to Modify**: `frontend/src/components/settings/CalendarSettings.jsx`

- [ ] 11. Implement Classification Selection Handler
  - Create `handleClassificationChange()` function that updates `advancedSelectedClassification` state
  - Extract UUID from dropdown selection
  - Verify weekday and date selections are NOT affected
  - Enable/disable dropdown based on whether any day is selected
  - Show error if no classifications available
  - _Requirements: 5.1, 5.4, 5.5_
  - **Files to Modify**: `frontend/src/components/settings/CalendarSettings.jsx`

- [ ] 12. Implement Validation Logic
  - Create comprehensive validation check in `getValidationErrors()`:
    - Verify at least one day selected (weekday OR dates)
    - Verify classification is selected
    - Verify month is loaded (gridMonthId is set)
  - Return array of specific error messages for each failure case
  - Call validation before any assignment attempt
  - Display all errors to user if validation fails
  - _Requirements: 6.2, 6.3, 6.4, 11.1_
  - **Files to Modify**: `frontend/src/components/settings/CalendarSettings.jsx`

---

### Phase 4: API Integration

- [ ] 13. Implement Weekday Assignment API Call
  - Create function to call `assignByWeekday()` with correct payload: `{ day_of_week, day_type_id, month_id }`
  - Pass `advancedSelectedWeekday`, `advancedSelectedClassification`, and `gridMonthId` to API
  - Handle success response: capture count of assigned days
  - Handle error response: extract user-friendly message via `extractErrorMessage()`
  - Return structured result: `{ success: boolean, count?: number, error?: string }`
  - Avoid making call if no weekday selected (check before call)
  - _Requirements: 10.1, 10.2, 13._
  - **Files to Modify**: `frontend/src/components/settings/CalendarSettings.jsx`

- [ ] 14. Implement Specific Date Assignment API Call
  - Create function to call `manualAssignDayTypes()` with correct payload: `{ month_id, assignments: [{ day_number, day_type_id }, ...] }`
  - Build assignments array from `advancedSelectedDates` Set
  - Pass `advancedSelectedClassification` as `day_type_id` for each date
  - Handle success response: capture count of assigned dates
  - Handle error response: extract user-friendly message
  - Return structured result: `{ success: boolean, count?: number, error?: string }`
  - Avoid making call if no dates selected (check before call)
  - _Requirements: 10.3, 10.4, 10.5_
  - **Files to Modify**: `frontend/src/components/settings/CalendarSettings.jsx`

- [ ] 15. Implement Combined Assignment Orchestration
  - Create `performAssignment()` async function that:
    1. Sets `advancedIsAssigning = true` (show spinner, disable button)
    2. Calls weekday assignment if weekday selected
    3. Calls date assignment if dates selected (may run in parallel or sequence based on design)
    4. Collects results from both calls
    5. Handles partial success: one succeeds, one fails
    6. Handles both success: clear selections, show success message
    7. Handles both fail: show combined error, retain selections
    8. Sets `advancedIsAssigning = false` (hide spinner, enable button)
  - Include error handling for network failures
  - Return combined result with success/error/partial flags
  - _Requirements: 6.1, 6.5, 10.5, 10.7_
  - **Files to Modify**: `frontend/src/components/settings/CalendarSettings.jsx`

- [ ] 16. Implement Assignment Click Handler and Message Management
  - Create `handleAssignClick()` function:
    1. Call `getValidationErrors()` to check prerequisites
    2. If errors, set `advancedMessage` with error type and display errors
    3. If valid, call `performAssignment()`
    4. Handle success: set message with success type, count of assigned days
    5. Handle error: set message with error type and details
    6. Handle partial success: set message with warning type and breakdown
  - Implement auto-dismiss logic: success messages disappear after 5s via setTimeout
  - Error messages persist until user dismisses or retries
  - Set message `timestamp` for tracking
  - Clear previous message before new operation
  - _Requirements: 6.6, 6.7, 9.1, 9.2, 9.3, 11.1, 11.2, 11.3, 12.4_
  - **Files to Modify**: `frontend/src/components/settings/CalendarSettings.jsx`

---

### Phase 5: Testing & Verification

- [ ] 17. Write Unit Tests for State Management
  - Test initial state: all variables set to correct defaults (null, empty Set, null message)
  - Test state updates when weekday selected
  - Test state updates when dates toggled
  - Test state updates when classification selected
  - Test state updates when assignment completes
  - Create test file: `frontend/src/components/settings/__tests__/CalendarSettings.advanced.state.test.js`
  - _Requirements: 2.3, 3.3, 5.1, 6.1_

- [ ] 18. Write Unit Tests for Weekday Selection
  - Test weekday count accuracy: verify `countWeekdayOccurrences()` matches loaded calendar data
  - Test weekday dropdown renders all 7 days
  - Test selecting different weekdays replaces previous selection
  - Test clearing weekday sets state to null
  - Test occurrence count displayed correctly in dropdown
  - Test weekday count with various month lengths (28, 29, 30, 31 days)
  - _Requirements: 2.1, 2.2, 2.3, 2.4, 2.5, 2.6, 7.1_

- [ ] 19. Write Unit Tests for Specific Date Selection
  - Test date button renders for all days in month (1 through N)
  - Test toggling date adds to selection set
  - Test toggling same date removes from set
  - Test clicking date twice returns to original state (idempotence)
  - Test multiple dates can be selected independently
  - Test "Clear All" empties selection set
  - Test no duplicate dates in selection set
  - Test visual highlight applied to selected dates
  - _Requirements: 3.1, 3.2, 3.3, 3.4, 3.5, 3.6, 3.7, 7.1_

- [ ] 20. Write Unit Tests for Classification Selection
  - Test classification dropdown displays all available types
  - Test selecting classification updates state
  - Test dropdown disabled when no days selected (weekday and dates both empty)
  - Test dropdown enabled when at least one day selected (weekday or date)
  - Test error state when backend returns no classifications
  - Test selected classification shown in preview
  - _Requirements: 5.1, 5.2, 5.3, 5.4, 5.5_

- [ ] 21. Write Unit Tests for Combined Selection State
  - Test weekday and date selections are independent (selecting weekday doesn't clear dates)
  - Test weekday and date selections are independent (selecting dates doesn't clear weekday)
  - Test classification selection doesn't affect weekday/date selections
  - Test combined count calculation: total = weekday count + date count
  - Test preview displays correct combined count
  - Test clearing weekday retains date selections
  - Test clearing dates retains weekday selection
  - _Requirements: 4.1, 4.2, 4.3, 4.4, 4.5_

- [ ] 22. Write Unit Tests for Validation
  - Test validation fails when no days selected
  - Test validation fails when no classification selected
  - Test validation fails when no month loaded
  - Test validation passes when weekday selected AND classification selected
  - Test validation passes when dates selected AND classification selected
  - Test validation passes when both weekday and dates selected AND classification selected
  - Test error messages are specific to the failure
  - _Requirements: 6.2, 6.3, 6.4, 11.1, 11.2_

- [ ] 23. Write Unit Tests for API Call Building
  - Test weekday assignment payload: `{ day_of_week, day_type_id, month_id }`
  - Test manual assignment payload: `{ month_id, assignments: [{ day_number, day_type_id }, ...] }`
  - Test date numbers are correct 1-based values
  - Test payload correctness with single and multiple selections
  - Test UUID extraction from classification dropdown
  - Test payload doesn't include null/undefined values
  - _Requirements: 10.1, 10.2, 10.3, 10.4_

- [ ] 24.* Write Property-Based Test for Weekday Count Accuracy (Property 1)
  - **Property 1: Weekday Selection Count Accuracy**
  - For any selected month and any valid weekday, the count displayed SHALL match the actual count of calendar days with that day_of_week in loaded data
  - Use fast-check or jest-property to generate random weekday indices (0-6) and calendar data
  - Load calendar days with known weekday distribution
  - Assert `countWeekdayOccurrences(weekday) === calDays.filter(d => d.day_of_week === weekday).length`
  - Test multiple months with varying day lengths (28, 29, 30, 31 days)
  - Test with weekdays that don't occur in month (count = 0)
  - **Validates: Requirements 2.1, 2.2, 2.3, 7.1**

- [ ] 25.* Write Property-Based Test for Date Selection Idempotence (Property 2)
  - **Property 2: Specific Dates Toggle Idempotence**
  - For any date in the selectable range, clicking a date twice SHALL return to original state (no duplicates)
  - Use fast-check to generate random day numbers (1-31)
  - Assert that toggleDate(N) twice returns to initial selection state
  - Test with empty initial state, single date, multiple dates
  - Test that toggle order doesn't matter: toggle(5), toggle(8), toggle(5) leaves only 8 selected
  - **Validates: Requirements 3.3, 3.4, 3.5, 3.6**

- [ ] 26.* Write Property-Based Test for Selection State Independence (Property 3)
  - **Property 3: Selection State Independence**
  - For any valid month, selecting weekday and selecting dates simultaneously SHALL result in both being retained
  - Use fast-check to generate random weekday and date combinations
  - Select random weekday, assert date state unchanged
  - Add random dates to selection, assert weekday unchanged
  - Test all combinations: weekday only, dates only, both, neither
  - Assert neither operation clears the other selection
  - **Validates: Requirements 4.1, 4.2, 4.4, 4.5**

- [ ] 27.* Write Property-Based Test for Combined Count Correctness (Property 4)
  - **Property 4: Combined Count Correctness**
  - For any valid combination of selected weekday and specific dates, the total count SHALL equal (weekday count) + (specific date count)
  - Use fast-check to generate random weekday selections, date selections, and their combinations
  - Assert `computeTotal() === countWeekdayOccurrences() + advancedSelectedDates.size`
  - Test with 0 weekday occurrences (weekday not in month)
  - Test with 0 dates selected
  - Test with max dates (31) + multiple weekday occurrences
  - **Validates: Requirements 4.3, 7.2, 7.3**

- [ ] 28.* Write Property-Based Test for Validation Before Assignment (Property 5)
  - **Property 5: Validation Before Assignment**
  - For any state where days or classification are missing, assignment SHALL NOT proceed, and error message SHALL display
  - Use fast-check to generate invalid states: no weekday, no dates, no classification, no month
  - Assert `canAssign() === false` for all invalid states
  - Assert assignment function returns early without API call
  - Assert error message is displayed for each failure case
  - **Validates: Requirements 6.2, 6.3, 6.4**

- [ ] 29.* Write Integration Test for Complete Assignment Flow
  - Test end-to-end: load month → select weekday → select dates → select classification → click Assign → verify API call payload → verify success message → verify selections cleared
  - Mock API endpoints for weekday assignment and manual assignment
  - Verify both API calls made with correct payloads
  - Verify success message shows total count (weekday + dates)
  - Verify selections cleared after success
  - Mock partial success scenario: one API succeeds, one fails
  - Verify warning message displays with breakdown
  - Verify selections retained for retry
  - _Requirements: 6.1, 6.5, 6.6, 6.7, 10.5, 10.7_

- [ ] 30.* Write Integration Test for Partial Success and Error Handling
  - Test weekday assignment fails, date assignment succeeds
  - Test date assignment fails, weekday assignment succeeds
  - Test both assignments fail with different error messages
  - Test network error during assignment
  - Verify error messages are user-friendly and specific
  - Verify selections retained after error
  - Verify user can retry without re-selecting
  - Verify button re-enabled after error
  - _Requirements: 6.7, 10.7, 11.1, 11.2, 11.3_

- [ ] 31.* Write Integration Test for Message Lifecycle (Property 9)
  - Test success message displays immediately after successful assignment
  - Test success message auto-dismisses after 5 seconds
  - Test error message persists (user must close or retry)
  - Test multiple messages don't overlap (new message replaces old)
  - Test message timestamp tracked
  - Test role="status" and aria-live="polite" attributes for screen readers
  - _Requirements: 6.6, 6.7, 9.1, 9.2, 9.3, 11.1_

---

### Phase 6: Integration & Finalization

- [ ] 32. Test Property 6: API Payload Correctness (Weekday)
  - **Property 6: API Payload Correctness (Weekday)**
  - When assigning days by weekday, the API request payload SHALL include exactly: `{ day_of_week, day_type_id, month_id }`
  - Mock the `assignByWeekday()` call and capture the payload
  - Assert payload structure contains all required fields with correct types
  - Assert day_of_week is a string matching selected weekday
  - Assert day_type_id is a valid UUID
  - Assert month_id matches loaded month
  - Test with all 7 weekdays
  - **Validates: Requirements 10.1, 10.2**

- [ ] 33. Test Property 7: API Payload Correctness (Manual)
  - **Property 7: API Payload Correctness (Manual)**
  - When assigning specific dates, the API request payload SHALL include exactly: `{ month_id, assignments: [{ day_number, day_type_id }, ...] }`
  - Mock the `manualAssignDayTypes()` call and capture the payload
  - Assert payload structure contains all required fields
  - Assert assignments array length matches selected dates count
  - Assert each assignment has day_number (1..31) and day_type_id (UUID)
  - Assert day_number values match selected dates exactly
  - Test with 1, 5, 10, 31 selected dates
  - **Validates: Requirements 10.3, 10.4**

- [ ] 34. Test Property 8: Partial Success Handling
  - **Property 8: Partial Success Handling**
  - When one API call succeeds and another fails, the system SHALL display a partial success message indicating which assignments succeeded and which failed
  - Test scenario: weekday succeeds (4 days), manual fails
  - Verify message displays: "Assigned 4 weekday occurrences, but failed to assign specific dates: [error detail]"
  - Test scenario: weekday fails, manual succeeds (3 days)
  - Verify message displays: "Failed to assign weekday occurrences: [error detail], but assigned 3 specific dates"
  - Test with actual error responses from mock API
  - **Validates: Requirements 6.7, 10.7**

- [ ] 35. Test Property 10: Sidebar Refresh After Assignment
  - **Property 10: Sidebar Refresh After Assignment**
  - When the user completes a successful assignment, the Calendar module sidebar SHALL reflect newly assigned day types
  - Setup: Create mock Calendar component with sidebar
  - Perform successful assignment in CalendarSettings
  - Verify sidebar data refreshed (mock data fetch)
  - Verify sidebar displays newly assigned day types with visual indicators
  - Test with weekday-only, date-only, and combined assignments
  - Verify sidebar updates on same page navigation (if implemented) or after page refresh (MVP acceptable)
  - **Validates: Requirements 8.1, 8.2, 8.4**

- [ ] 36. Verify Accessibility Compliance (WCAG 2.1 AA)
  - Run accessibility audit on Advanced Assignments section
  - Verify color contrast meets WCAG AA standards (4.5:1 for text, 3:1 for graphics)
  - Verify keyboard navigation works: Tab order is logical (Month → Weekday → Dates → Classification → Assign)
  - Verify all buttons, dropdowns, and inputs are keyboard accessible (Enter, Space to activate)
  - Verify focus outline visible on all interactive elements
  - Verify form labels associated with inputs via `<label htmlFor>` or aria-label
  - Verify error messages have `role="alert"` for screen reader announcement
  - Verify status messages have `role="status"` aria-live="polite" for live region updates
  - Verify date button touch targets are minimum 44x44px on mobile
  - Test with screen reader (NVDA or JAWS): all UI elements announce correctly
  - _Requirements: 1.4.3, 2.1.1, 2.4.3, 3.3.1, 3.3.2, 4.1.2, 4.1.3 (WCAG)_

- [ ] 37. Verify Performance Targets
  - Date grid rendering: Measure render time with 31 date buttons, target < 200ms
  - Month switch: Measure time to reload calendar days and re-render, target < 500ms
  - Use React DevTools Profiler to identify bottlenecks
  - Verify no unnecessary re-renders of date grid or other components
  - Verify memoization is applied to prevent re-renders
  - Test on low-end devices (simulate slow CPU in DevTools)
  - _Requirements: 12.1, 12.2_

- [ ] 38. Run Full Feature Regression Test
  - Verify existing bulk grid interface still functions correctly
  - Verify existing Import/Export CSV functionality works
  - Verify existing Year, Category, Classification setup unchanged
  - Verify no state leakage between bulk grid and advanced workflow
  - Verify Calendar sidebar updates after advanced assignments
  - Test on different browsers: Chrome, Firefox, Safari, Edge
  - Test on mobile devices: iPhone, Android (responsive layout)
  - Verify no console errors or warnings
  - _Requirements: 9.1, 9.2, 9.3, 9.4, 9.5, 9.6, 9.7_

- [ ] 39. Update Component Documentation and Code Comments
  - Add JSDoc comments to all new functions (description, parameters, return types)
  - Add inline comments explaining complex logic (validation, API orchestration, message handling)
  - Document state structure in header comment
  - Document helper functions in separate section
  - Add comments explaining error handling strategy
  - Add README.md section documenting Advanced Assignments feature (user guide)
  - Document API payload structures as inline comments
  - Add comments about performance optimizations (memoization, debounce)
  - _Requirements: Documentation requirement_

- [ ] 40. Create Test Documentation and User Guide
  - Document all unit test suites and their coverage
  - Document property-based tests with examples of generated test cases
  - Create user guide: how to use Advanced Assignments workflow (with screenshots/mockups)
  - Document error scenarios and recovery procedures
  - Document keyboard shortcuts and accessibility features
  - Document performance characteristics and optimization techniques
  - Create troubleshooting guide for common issues
  - _Requirements: Documentation requirement_

- [ ] 41. Prepare for Code Review
  - Ensure all tests pass locally (unit, integration, property-based)
  - Run linter (ESLint) and fix any violations
  - Check code coverage: target > 80% for new code
  - Verify all requirements covered by implementation
  - Prepare PR description with summary of changes
  - Create checklist of items reviewed: functionality, accessibility, performance, testing
  - Note any known limitations or future enhancements
  - _Requirements: All requirements, code quality standards_

- [ ] 42. Deploy and Monitor
  - Merge PR to main branch after approval
  - Deploy to staging environment
  - Run smoke tests in staging (manual spot-checks)
  - Deploy to production
  - Monitor error logs for any new exceptions
  - Monitor analytics for feature usage (optional)
  - Prepare rollback plan in case of issues
  - Announce feature to team/users
  - _Requirements: Deployment and monitoring_

---

## Notes

### Optional Testing Tasks

Tasks marked with `*` are property-based and integration tests. These are critical for correctness verification but can be scheduled separately if needed:

- **Must implement**: Tasks 24-28 (core property tests)
- **Must implement**: Tasks 29-31 (integration tests and error handling)
- **Must implement**: Tasks 32-35 (advanced property tests)
- Optional optimizations: Tasks 37-40 (can be deferred if MVP time-constrained)

### Task Dependencies Summary

- **Phase 1** (Tasks 1-2): No dependencies → START HERE
- **Phase 2** (Tasks 3-8): Depends on Phase 1
- **Phase 3** (Tasks 9-12): Depends on Phases 1-2
- **Phase 4** (Tasks 13-16): Depends on Phases 1-3
- **Phase 5** (Tasks 17-42): Depends on Phases 1-4 (can begin in parallel after Phase 3 complete)
- **Phase 6** (Tasks 39-42): Final phase, after all testing complete

### Key Implementation Patterns

1. **State Management**: All state centralized in CalendarSettings component
2. **API Calls**: Use existing `assignByWeekday()` and `manualAssignDayTypes()` endpoints
3. **Error Handling**: Extract user-friendly messages, retain selections for retry
4. **Validation**: Comprehensive client-side checks before API calls
5. **Testing**: Property-based tests verify invariants, integration tests verify workflows
6. **Accessibility**: All interactive elements keyboard-accessible, screen-reader compatible
7. **Performance**: Memoization for date grid, debounce for month switches

### Estimated Effort (T-Shirt Sizing)

- Phase 1 (State Setup): XS
- Phase 2 (UI Components): S
- Phase 3 (Core Logic): S
- Phase 4 (API Integration): M
- Phase 5 (Testing): L (most effort here)
- Phase 6 (Integration & Finalization): M

---

## Task Dependency Graph

```json
{
  "waves": [
    { "id": 0, "tasks": ["1", "2"] },
    { "id": 1, "tasks": ["3", "4", "5", "6", "7"] },
    { "id": 2, "tasks": ["8", "9", "10", "11", "12"] },
    { "id": 3, "tasks": ["13", "14", "15", "16"] },
    { "id": 4, "tasks": ["17", "18", "19", "20", "21", "22", "23"] },
    { "id": 5, "tasks": ["24", "25", "26", "27", "28"] },
    { "id": 6, "tasks": ["29", "30", "31"] },
    { "id": 7, "tasks": ["32", "33", "34", "35"] },
    { "id": 8, "tasks": ["36", "37"] },
    { "id": 9, "tasks": ["38"] },
    { "id": 10, "tasks": ["39", "40"] },
    { "id": 11, "tasks": ["41", "42"] }
  ]
}
```

