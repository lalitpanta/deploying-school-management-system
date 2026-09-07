# Design Document: Enhanced Day Assignments Workflow

## Overview

The Enhanced Day Assignments Workflow introduces a new, parallel assignment interface to the CalendarSettings component. Rather than replacing the existing bulk grid interface, this new workflow provides an alternative for users who need to combine weekday assignments (e.g., "all Mondays") with specific date selections in a single operation.

### Current State Problem
- Users working with mixed patterns (weekdays + specific dates) must choose one approach
- The existing bulk grid requires sequential operations: select all Mondays, assign, then select specific dates, assign again
- No preview of combined selections before assignment

### Solution
- New UI section: "Advanced Day Assignments" with independent weekday + specific date selection
- Single combined preview showing total days to be assigned
- One API request cycle combining both assignment types
- Non-destructive coexistence with existing bulk grid interface
- Immediate feedback and error recovery

### Key Constraints
- Backend already supports `/v1/calendar-days/assign-by-weekday` and `/v1/calendar-days/manual-assign` endpoints
- Must preserve all existing functionality (bulk grid, import/export, year/category setup)
- Calendar sidebar must update after successful assignment
- Performance: 20+ date selections must render within 200ms

---

## Architecture

### Component Structure

```
CalendarSettings (existing root component)
├── Tab Navigation (existing)
│   ├── Setup tab (existing)
│   ├── Day Assignments tab (existing bulk grid)
│   └── Import/Export tab (existing)
├── Setup Section (existing - Requirement 1, 5, 9 coverage)
│   ├── Year/Month/Category setup
│   ├── Classifications management
│   └── Grid bulk assignment interface
│
└── [NEW] Advanced Day Assignments Section (this feature)
    ├── Year/Month selector (disabled until prerequisites)
    ├── Weekday selection dropdown
    ├── Specific dates multi-select list
    ├── Classification selector
    ├── Combined selection preview
    ├── Assign button + validation
    └── Status messages (loading, success, error)
```

**Placement**: New "Advanced" subsection within the "Day Assignments" tab, positioned below or alongside existing bulk grid interface. Uses `.panel` class to match existing styling.

---

## Components and Interfaces

### 1. State Variables

```javascript
// ── ADVANCED ASSIGNMENT STATE ──

// Selection state
const [advancedSelectedWeekday, setAdvancedSelectedWeekday] = useState(null);
  // null | "Sunday" | "Monday" | ... | "Saturday"

const [advancedSelectedDates, setAdvancedSelectedDates] = useState(new Set());
  // Set<number> containing 1..N day numbers for selected month

const [advancedSelectedClassification, setAdvancedSelectedClassification] = useState(null);
  // null | UUID (string)

// UI state
const [advancedIsAssigning, setAdvancedIsAssigning] = useState(false);
  // true during API call, disables button and shows spinner

const [advancedMessage, setAdvancedMessage] = useState({
  type: null,       // null | "success" | "error" | "warning"
  text: "",         // Human-readable message
  timestamp: null   // For auto-dismiss logic
});

// Month context (from existing grid state)
// Reuse: gridMonthId, calDays, availableMonths, dayTypes, mode
```

### 2. Computed Properties / Helpers

```javascript
// Count weekday occurrences in selected month
const countWeekdayOccurrences = () => {
  if (!advancedSelectedWeekday || !gridMonthId) return 0;
  return calDays.filter(
    d => d.day_of_week === advancedSelectedWeekday
  ).length;
};

// Generate summary preview
const generateAssignmentSummary = () => {
  const weekdayCount = countWeekdayOccurrences();
  const dateCount = advancedSelectedDates.size;
  const total = weekdayCount + dateCount;
  
  if (total === 0) return "No days selected";
  
  const parts = [];
  if (weekdayCount > 0) parts.push(`${weekdayCount} ${advancedSelectedWeekday}s`);
  if (dateCount > 0) parts.push(`${dateCount} specific date${dateCount > 1 ? 's' : ''}`);
  
  return `${parts.join(' + ')} = ${total} total days`;
};

// Determine if assignment is possible
const canAssign = () => {
  const hasDaysSelected = advancedSelectedDates.size > 0 || advancedSelectedWeekday;
  const hasClassification = !!advancedSelectedClassification;
  return hasDaysSelected && hasClassification && !advancedIsAssigning;
};

// Validation errors
const getValidationErrors = () => {
  const errors = [];
  const hasDaysSelected = advancedSelectedDates.size > 0 || advancedSelectedWeekday;
  
  if (!hasDaysSelected) {
    errors.push("Please select at least one day (weekday or specific date)");
  }
  if (!advancedSelectedClassification) {
    errors.push("Please select a classification");
  }
  if (!gridMonthId) {
    errors.push("Please select a month first");
  }
  
  return errors;
};
```

### 3. Weekday Selection Dropdown

**UI Component Requirements**:
- Location: Below Month selector in Advanced section
- Label: "Weekday Selection"
- Options: 
  - "None" (placeholder, value: null)
  - "Sunday", "Monday", "Tuesday", "Wednesday", "Thursday", "Friday", "Saturday"
- Display count: Show "Monday (4 occurrences in Baisakh)" next to each option
- Clear button: When weekday selected, show "Clear weekday" text or icon

**Behavior**:
- Single select (replaces previous selection)
- Shows occurrence count dynamically based on loaded calendar days
- Visual feedback: highlight selected weekday with accent color
- Independent from specific dates (can select both simultaneously)

**Accessibility**:
- `<select>` element with proper labels
- ARIA attributes: `aria-label`, `aria-describedby` for help text
- Keyboard navigation: arrow keys, enter to confirm

### 4. Specific Dates Multi-Select List

**UI Component Requirements**:
- Location: Beside or below Weekday dropdown
- Display: Grid of numbered buttons (1-31, depending on month length)
- Selection state: 
  - Unselected: `.cal-grid-item` (existing style, border: `--border`)
  - Selected: `.cal-grid-item.selected` (gradient background, accent color)
- Clear All button: Below list, triggers `setAdvancedSelectedDates(new Set())`
- Layout: 7 columns (7 days per week) responsive to mobile (4 columns on small screens)

**Behavior**:
- Toggle selection: click date number to add/remove
- Multiple selection: each click adds or removes independently
- Cumulative: previous selections persist when selecting different dates
- Visual count: "Selected X dates" display above Clear All button
- Max performance: Render 31 items within 200ms (CSS grid, React keys)

**Accessibility**:
- Buttons with semantic role="button"
- `aria-pressed="true|false"` on each date button
- `aria-label="Day 5, currently selected"` pattern
- Keyboard: Tab navigation, Space/Enter to toggle

### 5. Classification Selector Dropdown

**UI Component Requirements**:
- Label: "Classification"
- Display: All available day types from backend (`dayTypes` array)
- Placeholder: "Select a classification"
- Disabled state: When no days selected
- Options format: "Public Holiday", "Working Day", etc.

**Behavior**:
- Single select
- Disable dropdown until at least one day is selected
- Show error state if backend returns no classifications
- Selected value stored as UUID

**Accessibility**:
- `<select>` with `aria-label`
- Disabled state communicated visually and via `disabled` attribute

### 6. Combined Selection Preview

**UI Component Requirements**:
- Container: `.preview-box` class (existing style, dashed border)
- Content areas:
  - Main line: "4 Mondays + 3 specific dates = 7 total days"
  - Sub-text (optional): "Assigned classification: Public Holiday"
- Empty state: "No days selected"
- Update trigger: Any change to selectedWeekday, selectedDates, selectedClassification

**Dynamic Elements**:
- Weekday section (if selected): "Mondays: X occurrences"
- Dates section (if selected): "Dates: 5, 8, 12, 15, 20"
- Total: Color-accented number
- Classification name: Shown in summary if selected

### 7. Assignment Operation (Assign Button + Status)

**Button States**:
- Disabled: No days selected OR no classification selected OR API in-flight
- Loading: `advancedIsAssigning = true`, show spinner, text "Assigning..."
- Ready: All validation passed, clickable, text "Assign Days"

**Status Message Display**:
- Success message (green): "7 days assigned successfully. 4 Mondays + 3 specific dates."
- Error message (red): "Failed to assign days: [backend error message]"
- Warning message (yellow): "Partially assigned: 4 Mondays assigned, but 3 specific dates failed. Please retry."
- Auto-dismiss: Success/warning after 5 seconds, error persistent (user must close)
- Position: Below assignment button, inside `.panel`

**Validation on Click**:
```javascript
const handleAssignClick = () => {
  const errors = getValidationErrors();
  if (errors.length > 0) {
    setAdvancedMessage({
      type: "error",
      text: errors.join("\n"),
      timestamp: Date.now()
    });
    return;
  }
  performAssignment();
};
```

---

## Data Models

### Calendar Day Structure (from backend)
```javascript
{
  id: "uuid",                    // Unique identifier
  month_id: "uuid",              // Reference to month
  day_number: 1-31,              // 1-based day of month
  day_of_week: "Monday",         // Sunday through Saturday
  day_type_id: "uuid" | null,    // Current assignment
  day_type: "Public Holiday" | null  // Populated from join
}
```

### Month Structure (from backend)
```javascript
{
  id: "uuid",
  year_id: "uuid",
  month_name: "Baisakh",
  bs_month_index: 1,
  total_days: 31,
  created_at: "2025-...",
  updated_at: "2025-..."
}
```

### Day Type / Classification Structure (from backend)
```javascript
{
  id: "uuid",
  day_type: "Public Holiday",
  category_id: "uuid" | null,
  created_at: "2025-...",
  updated_at: "2025-..."
}
```

---

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Property 1: Weekday Selection Count Accuracy

*For any* selected month and any valid weekday, the count of weekday occurrences displayed in the UI SHALL match the actual count of calendar days with that day_of_week in the loaded calendar data.

**Validates: Requirements 2.1, 2.2, 2.3, 7.1**

### Property 2: Specific Dates Toggle Idempotence

*For any* date in the selectable range, clicking a date that is already selected SHALL remove it from the selection state, and clicking it again SHALL add it back (no duplicates, no extra/missing states).

**Validates: Requirements 3.3, 3.4, 3.6**

### Property 3: Selection State Independence

*For any* valid month, selecting a weekday and selecting specific dates simultaneously SHALL result in both being retained in separate state variables (neither clears the other).

**Validates: Requirements 4.1, 4.4, 4.5**

### Property 4: Combined Count Correctness

*For any* valid combination of selected weekday and specific dates, the displayed total count SHALL equal (weekday occurrence count) + (number of selected specific dates).

**Validates: Requirements 4.3, 7.2, 7.3**

### Property 5: Validation Before Assignment

*For any* state where either days or classification are missing, the assignment operation SHALL not proceed, and a validation error message SHALL be displayed.

**Validates: Requirements 6.2, 6.3, 6.4**

### Property 6: API Payload Correctness (Weekday)

*When* assigning days by weekday, the API request payload SHALL include exactly: `{ day_of_week, day_type_id, month_id }` with correct data types and values matching user selections.

**Validates: Requirements 10.1, 10.2**

### Property 7: API Payload Correctness (Manual)

*When* assigning specific dates, the API request payload SHALL include exactly: `{ month_id, assignments: [{ day_number, day_type_id }, ...] }` with day_number values (1..N) matching user selections.

**Validates: Requirements 10.3, 10.4**

### Property 8: Partial Success Handling

*When* one API call succeeds and another fails (e.g., weekday succeeds but manual-assign fails), the system SHALL display a partial success message indicating which assignments succeeded and which failed.

**Validates: Requirements 6.7, 10.7**

### Property 9: Message Lifecycle

*For any* successful assignment, a success message SHALL be displayed immediately, and SHALL auto-dismiss after 5 seconds. *For any* error, the error message SHALL persist until dismissed by user action.

**Validates: Requirements 6.6, 6.7, 11.1, 11.2, 11.3**

### Property 10: Sidebar Refresh After Assignment

*When* the user completes a successful assignment, the Calendar module sidebar (if present or on next navigation) SHALL reflect the newly assigned day types with visual indicators.

**Validates: Requirements 8.1, 8.2, 8.4**

---

## Error Handling

### Validation Errors (Client-Side)

| Error | Trigger | Message | Recovery |
|-------|---------|---------|----------|
| No days selected | User clicks Assign with no weekday and no dates | "Please select at least one day (weekday or specific date)" | User selects a weekday or date, retries |
| No classification | User clicks Assign without selecting classification | "Please select a classification" | User selects a classification, retries |
| No month loaded | User tries to assign without loading a month | "Please select a month first" | User loads a month, retries |

### API Errors (Server Response)

| Error Type | Status | Message | Recovery |
|-----------|--------|---------|----------|
| Weekday assignment fails | 4xx/5xx | Extract from `response.data.error` or fallback to "Failed to assign weekday occurrences" | Show error, retain selection, user can retry |
| Manual assign fails | 4xx/5xx | Extract from `response.data.error` or fallback to "Failed to assign specific dates" | Show error, retain selection, user can retry |
| Both fail | 4xx/5xx | "Failed to complete assignment. Weekday: [error1]. Specific dates: [error2]" | Show combined error, retain selection, user can retry |
| Network error | Network | "Connection lost. Please check your internet and retry." | User retries when connection restored |

### Partial Success Scenario

**Condition**: Weekday assignment succeeds, manual-assign fails (or vice versa)

**Handler**:
```javascript
const response = {
  weekday: { success: true, count: 4 },
  manual: { success: false, error: "Month not found" }
};

const message = buildPartialSuccessMessage(response);
// "Assigned 4 weekday occurrences, but failed to assign specific dates: Month not found. Please retry the specific date assignment."
```

**User Action**: Retry button retries only failed portion, or clear and start over.

---

## API Integration Strategy

### Request Sequence

```
User clicks "Assign"
│
├─ Validate client-side (days selected, classification selected, month loaded)
│  └─ If invalid: show error, return (don't proceed)
│
├─ Set advancedIsAssigning = true (show spinner, disable button)
│
├─ IF weekday selected:
│  ├─ Call assignByWeekday({
│  │    day_of_week: advancedSelectedWeekday,
│  │    day_type_id: advancedSelectedClassification,
│  │    month_id: gridMonthId
│  │  })
│  ├─ Capture response/error
│  └─ Store result in weekdayResult
│
├─ IF specific dates selected:
│  ├─ Build assignments array:
│  │  {
│  │    day_number: 5,
│  │    day_type_id: advancedSelectedClassification
│  │  },
│  │  { day_number: 8, day_type_id: advancedSelectedClassification },
│  │  ...
│  │
│  ├─ Call manualAssignDayTypes(gridMonthId, assignments)
│  ├─ Capture response/error
│  └─ Store result in manualResult
│
├─ Set advancedIsAssigning = false (hide spinner, enable button)
│
├─ Evaluate results:
│  ├─ Both succeeded: Show success message, clear selections, refresh grid
│  ├─ One succeeded: Show partial success message, refresh grid, retain selections
│  ├─ Both failed: Show combined error message, retain selections
│  └─ Network error: Show connectivity error, retain selections
│
└─ Trigger sidebar refresh (Calendar module data fetch)
```

### Error Extraction

```javascript
const extractErrorMessage = (error) => {
  // Priority 1: Backend error response
  if (error.response?.data?.error) {
    return error.response.data.error;
  }
  if (error.response?.data?.message) {
    return error.response.data.message;
  }
  
  // Priority 2: HTTP status message
  if (error.response?.status) {
    const statusMap = {
      400: "Bad request. Check your input.",
      401: "Unauthorized. Please log in again.",
      403: "You don't have permission to perform this action.",
      404: "Resource not found.",
      409: "Conflict. The data may have changed.",
      500: "Server error. Please try again later.",
      503: "Service unavailable. Please try again later."
    };
    return statusMap[error.response.status] || "Request failed.";
  }
  
  // Priority 3: Network error
  if (!error.response) {
    return "Connection lost. Please check your internet and retry.";
  }
  
  // Fallback
  return "An unexpected error occurred.";
};
```

### Sidebar Calendar Update

**Approach 1: Callback/Event** (recommended if parent component available)
```javascript
// After successful assignment
if (typeof onAssignmentComplete === 'function') {
  onAssignmentComplete({ monthId: gridMonthId, count: totalAssigned });
}
```

**Approach 2: Direct Fetch** (if Calendar module is separate)
```javascript
// In effect dependency of assignment success
useEffect(() => {
  if (assignmentSuccessful && monthId) {
    // Trigger Calendar sidebar data refresh
    // (URL or method depends on Calendar component implementation)
    refreshCalendarSidebar(monthId);
  }
}, [assignmentSuccessful, monthId]);
```

**Approach 3: Force Full Page Refresh** (MVP acceptable per Requirement 8.3)
```javascript
// Only if other approaches not available
// After successful assignment with user acceptance
if (confirm("Assignment complete. Refresh to see updates in Calendar?")) {
  window.location.reload();
}
```

---

## Testing Strategy

### Unit Tests

**Test Suite: Weekday Selection**
- ✓ Weekday dropdown renders all 7 options
- ✓ Selecting weekday updates state
- ✓ Weekday count computed correctly for loaded month
- ✓ Clearing weekday sets state to null

**Test Suite: Specific Dates Selection**
- ✓ Date buttons render for all days in month (1 to N)
- ✓ Clicking date adds to selection set
- ✓ Clicking selected date removes from set
- ✓ Multiple selections work independently
- ✓ Clear All empties selection set

**Test Suite: Classification Selection**
- ✓ Classifications dropdown displays all available types
- ✓ Selecting classification updates state
- ✓ Dropdown disabled when no days selected
- ✓ Classification shows in preview when selected

**Test Suite: Validation**
- ✓ Cannot assign with no days selected
- ✓ Cannot assign with no classification selected
- ✓ Cannot assign with no month loaded
- ✓ Validation error messages are specific

**Test Suite: Preview**
- ✓ Preview shows "No days selected" when empty
- ✓ Preview shows correct weekday count
- ✓ Preview shows correct date count
- ✓ Preview shows correct total
- ✓ Preview updates when selection changes

**Test Suite: API Calls**
- ✓ Weekday-only assignment sends correct payload to `/assign-by-weekday`
- ✓ Dates-only assignment sends correct payload to `/manual-assign`
- ✓ Combined assignment makes both calls in sequence
- ✓ API errors are caught and displayed

**Test Suite: UI State During Assignment**
- ✓ Button disabled during API call
- ✓ Spinner displayed during API call
- ✓ Loading message shown
- ✓ Button re-enabled after success or error

**Test Suite: Messages**
- ✓ Success message displays with assignment count
- ✓ Error message displays with details
- ✓ Partial success message shows both outcomes
- ✓ Success message auto-dismisses after 5s
- ✓ Error message persists until dismissed

### Property-Based Tests

**Property 1: Weekday Count Invariant**
```javascript
// For any loaded month and any weekday
// countWeekdayOccurrences() === calDays.filter(d => d.day_of_week === weekday).length
test.prop([fc.integer({ min: 1, max: 7 })], (weekdayIndex) => {
  const weekday = WEEKDAYS_EN[weekdayIndex];
  const count = countWeekdayOccurrences(weekday);
  const expected = calDays.filter(d => d.day_of_week === weekday).length;
  expect(count).toEqual(expected);
});
```

**Property 2: Selection Toggle Idempotence**
```javascript
// For any date in the month, clicking it twice returns to original state
test.prop([fc.integer({ min: 1, max: 31 })], (dayNumber) => {
  const initial = new Set(advancedSelectedDates);
  toggleDate(dayNumber);
  toggleDate(dayNumber);
  expect(advancedSelectedDates).toEqual(initial);
});
```

**Property 3: Independent Selection**
```javascript
// Selecting weekday does not affect date selections
test(() => {
  const dateBefore = new Set(advancedSelectedDates);
  setAdvancedSelectedWeekday("Monday");
  expect(advancedSelectedDates).toEqual(dateBefore);
});
```

**Property 4: Combined Count Correctness**
```javascript
// For any valid combination, total = weekday count + date count
test.prop(
  [fc.boolean(), fc.array(fc.integer({ min: 1, max: 31 }))],
  (hasWeekday, dateNumbers) => {
    if (hasWeekday) setAdvancedSelectedWeekday("Monday");
    dateNumbers.forEach(d => setAdvancedSelectedDates(prev => new Set([...prev, d])));
    
    const weekdayCount = hasWeekday ? countWeekdayOccurrences("Monday") : 0;
    const dateCount = dateNumbers.length;
    const total = computeTotal();
    
    expect(total).toEqual(weekdayCount + dateCount);
  }
);
```

### Integration Tests

- ✓ Load month → select weekday → select dates → select classification → assign → verify API call payload
- ✓ Retry failed assignment without losing selection state
- ✓ Partial success: weekday succeeds, dates fail → user retries dates → both eventually succeed
- ✓ Network error → error shown → retry → succeeds
- ✓ Calendar sidebar updates after assignment (verify via Calendar module data)

### Performance Tests

- ✓ Render 31 date buttons within 200ms
- ✓ Month switch within 500ms (reload calendar days)
- ✓ API response time measured and logged
- ✓ No memory leaks during extended use (many assignments)

---

## Integration Points

### 1. Calendar Module Sidebar Refresh

**File**: `Calendar.jsx` (or calendar view component)

**Integration**:
- After successful assignment in CalendarSettings, sidebar should show newly assigned days
- Method A: Pass callback from parent: `<CalendarSettings onAssignmentComplete={refreshSidebar} />`
- Method B: Use shared state management (if Redux/Context available)
- Method C: Schedule sidebar fetch on next tab switch (user navigates to Calendar)
- Requirement 8.3 allows page refresh for MVP

**Expected Outcome**: 
- User assigns days in CalendarSettings
- Navigates to Calendar tab
- Sidebar reflects new assignments with visual indicators (colors, day type names)

### 2. Existing Bulk Grid Interface

**File**: `CalendarSettings.jsx` (same component)

**Integration**:
- New Advanced section does NOT modify existing grid state
- Both workflows can be used independently
- No state interference between bulk grid and advanced workflow
- Existing tab structure preserved

**Expected Outcome**:
- Users can still use bulk grid (Requirements 9.1, 9.2)
- Users can also use new advanced workflow
- Both coexist without conflict

### 3. Import/Export CSV

**File**: `CalendarSettings.jsx` (same component, existing implementation)

**Integration**:
- Advanced assignments produce same calendar_day records as bulk grid
- Export includes all days regardless of assignment method
- Import repopulates day types (no method tracking)
- No changes needed; existing code handles both sources

**Expected Outcome**:
- CSV export includes advanced assignments
- CSV import restores advanced assignments correctly
- No data loss or corruption

### 4. API Layer

**Files**: `calendarApi.js`

**Integration**:
- Use existing functions: `assignByWeekday()`, `manualAssignDayTypes()`
- Both endpoints already exist and tested
- New workflow = new orchestration of existing endpoints
- No backend changes required

**Expected Outcome**:
- Advanced workflow calls existing endpoints with correct payloads
- Backend processes requests as usual
- No new endpoints needed

---

## State Diagram

```
┌─────────────────────────────────────────────────────────────┐
│ Advanced Assignments State Machine                          │
└─────────────────────────────────────────────────────────────┘

[INITIAL] 
  └─ Month loaded: gridMonthId set, calDays populated
      └─ [READY_TO_SELECT]

[READY_TO_SELECT]
  ├─ User selects weekday
  │  └─ [WEEKDAY_SELECTED]
  ├─ User selects date(s)
  │  └─ [DATES_SELECTED]
  ├─ User selects classification
  │  └─ [CLASSIFICATION_SELECTED]
  └─ Any combination: [DAYS_SELECTED]

[DAYS_SELECTED]
  ├─ Classification NOT selected
  │  └─ [READY_FOR_CLASSIFICATION]
  ├─ Classification selected
  │  └─ [READY_TO_ASSIGN] ← Can click "Assign"
  ├─ User clears selection
  │  └─ [READY_TO_SELECT]
  └─ No changes
      └─ [READY_TO_ASSIGN]

[READY_TO_ASSIGN]
  └─ User clicks "Assign"
      └─ [ASSIGNING] (button disabled, spinner shown)

[ASSIGNING]
  ├─ Both API calls succeed
  │  └─ [SUCCESS] → clear selections → [READY_TO_SELECT]
  ├─ One succeeds, one fails
  │  └─ [PARTIAL_SUCCESS] → retain selections → [READY_TO_ASSIGN]
  ├─ Both fail
  │  └─ [ERROR] → retain selections → [READY_TO_ASSIGN]
  └─ Network error
      └─ [NETWORK_ERROR] → retain selections → [READY_TO_ASSIGN]
```

---

## Performance Considerations

### Optimization Goals

| Metric | Target | Method |
|--------|--------|--------|
| Date button render (31 items) | < 200ms | CSS Grid, React keys, memoization |
| Month switch | < 500ms | Debounce, preload on change |
| API response | < 2s | Use axios timeout, show spinner |
| Message auto-dismiss | 5s | setTimeout with cleanup |

### Implementation Details

```javascript
// Memoize date list to prevent unnecessary re-renders
const MemoizedDatesList = React.memo(({ 
  dates, 
  selectedDates, 
  onToggle 
}) => (
  <div className="cal-grid">
    {dates.map(dayNum => (
      <button
        key={dayNum}
        className={`cal-grid-item ${selectedDates.has(dayNum) ? 'selected' : ''}`}
        onClick={() => onToggle(dayNum)}
      >
        {dayNum}
      </button>
    ))}
  </div>
));

// Debounce month changes to prevent rapid API calls
const handleMonthChange = debounce((monthId) => {
  loadCalendarDays(monthId);
}, 300);

// Auto-dismiss messages with cleanup
useEffect(() => {
  if (advancedMessage.type && advancedMessage.type !== 'error') {
    const timer = setTimeout(() => {
      setAdvancedMessage({ type: null, text: '', timestamp: null });
    }, 5000);
    return () => clearTimeout(timer);
  }
}, [advancedMessage]);
```

---

## Constraints and Limitations

### MVP Constraints

| Constraint | Reason | Mitigation |
|-----------|--------|-----------|
| No real-time sidebar sync | Network complexity | Accept Requirement 8.3 (page refresh acceptable) |
| No undo/rollback | Complexity, not in requirements | Retain selection for retry |
| No bulk edit after assign | Out of scope | Users can reassign or use bulk grid |
| No conflict detection | Not specified | Accept overwrites (last write wins) |

### Technical Constraints

| Constraint | Impact | Mitigation |
|-----------|--------|-----------|
| Month must be pre-loaded | Weekday count calculation | Require month selection before assignment |
| Only one weekday at a time | Can't do "Mondays AND Fridays" | Out of scope; user makes two assignments |
| No date range selection | User must click each date | Out of scope; use bulk grid for range |
| Backend date format BS/AD only | Date display flexibility | Use mode flag to format dates correctly |

---

## Future Enhancements

Not in scope for MVP but worth noting:

1. **Date Range Selection**: "Select dates 5-15" instead of individual clicks
2. **Multiple Weekday Selection**: "Assign Mondays AND Fridays in one op"
3. **Recurring Assignments**: "Every Monday in all months"
4. **Undo/Rollback**: Track and revert assignments
5. **Real-time Sync**: WebSocket updates to sidebar
6. **Bulk Edit**: Select multiple months, apply assignments to all
7. **Templates**: Save and reuse assignment patterns

---

## Accessibility Compliance

### WCAG 2.1 Level AA

| Criterion | Implementation | Status |
|-----------|----------------|--------|
| 1.4.3 Contrast | Use existing CSS color scheme (--accent, --text, etc.) | ✓ |
| 2.1.1 Keyboard | All interactive elements accessible via keyboard (Tab, Enter, Space) | ✓ |
| 2.4.3 Focus Order | Logical tab order: Month → Weekday → Dates → Classification → Assign | ✓ |
| 3.3.1 Error Identification | Error messages specific, labeled with role="alert" | ✓ |
| 3.3.2 Labels/Instructions | All inputs have associated labels via `<label htmlFor>` | ✓ |
| 4.1.2 Name/Role/Value | Buttons, selects have proper roles; state communicated via aria attributes | ✓ |
| 4.1.3 Status Messages | Loading/success/error messages use role="status" with aria-live="polite" | ✓ |

### Screen Reader Support

```jsx
// Example: Assign button with status
<button 
  onClick={handleAssign}
  disabled={!canAssign()}
  aria-label={`Assign ${generateAssignmentSummary()} as ${advancedSelectedClassification?.name || 'unknown classification'}`}
>
  {advancedIsAssigning ? "Assigning..." : "Assign Days"}
</button>

// Status messages with live regions
<div role="status" aria-live="polite" aria-atomic="true">
  {advancedMessage.text}
</div>
```

### Mobile Accessibility

- Touch targets: Minimum 44x44px for date buttons
- Responsive layout: Single column on small screens
- Readable text: Minimum 16px font size on inputs
- No hover-only controls: All interactions available on touch

---

## Summary

The Enhanced Day Assignments Workflow provides a modern, intuitive alternative to the existing bulk grid, specifically designed for mixed assignment patterns. By leveraging existing backend APIs and preserving all current functionality, this feature can be implemented safely and incrementally. The design emphasizes user feedback, error recovery, and accessibility while maintaining performance constraints.

**Key Deliverables**:
1. New Advanced Assignments UI section with weekday + date selection
2. Combined preview and validation
3. Sequential API orchestration with partial success handling
4. Clear error messages and recovery paths
5. Integration with Calendar sidebar
6. Full accessibility compliance

**Implementation Path**:
1. Add state variables to CalendarSettings
2. Build Weekday dropdown component
3. Build Specific Dates grid component
4. Build Classification selector
5. Implement combined preview logic
6. Implement validation and assignment logic
7. Add error handling and messaging
8. Test with all combinations
9. Integrate with Calendar sidebar
10. Deploy and monitor

---

## Appendix: Code Structure Template

```javascript
// ── ADVANCED ASSIGNMENT STATE ──
const [advancedSelectedWeekday, setAdvancedSelectedWeekday] = useState(null);
const [advancedSelectedDates, setAdvancedSelectedDates] = useState(new Set());
const [advancedSelectedClassification, setAdvancedSelectedClassification] = useState(null);
const [advancedIsAssigning, setAdvancedIsAssigning] = useState(false);
const [advancedMessage, setAdvancedMessage] = useState({
  type: null,
  text: "",
  timestamp: null
});

// ── HELPER FUNCTIONS ──
const countWeekdayOccurrences = () => { /* ... */ };
const generateAssignmentSummary = () => { /* ... */ };
const canAssign = () => { /* ... */ };
const getValidationErrors = () => { /* ... */ };

// ── EVENT HANDLERS ──
const handleWeekdayChange = (e) => { /* ... */ };
const toggleDateSelection = (dayNum) => { /* ... */ };
const handleClearDates = () => { /* ... */ };
const handleClassificationChange = (e) => { /* ... */ };
const handleAssignClick = () => { /* ... */ };
const performAssignment = async () => { /* ... */ };

// ── RENDER SECTION (in JSX) ──
// <div className="panel">
//   <div className="panel-head">
//     <p className="panel-title">Advanced Day Assignments</p>
//   </div>
//   {/* Month selector (reuse existing) */}
//   {/* Weekday dropdown */}
//   {/* Specific dates grid */}
//   {/* Classification dropdown */}
//   {/* Preview box */}
//   {/* Assign button */}
//   {/* Status message */}
// </div>
```

