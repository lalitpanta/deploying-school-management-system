# Requirements Document

## Introduction

This requirements document describes the Enhanced Day Assignments Workflow feature for the School MIS Calendar module.

The current Day Assignments feature in CalendarSettings requires users to select days through a bulk selection grid followed by assigning a single classification type. This workflow is inefficient when working with mixed assignment patterns (weekdays + specific dates).

This feature introduces a new, parallel workflow that allows users to:
1. Select days through TWO independent methods simultaneously (weekday dropdown + specific date selection)
2. Select a classification/day type
3. Assign all selected days at once

The new workflow coexists with the existing bulk grid interface, providing users with options based on their assignment patterns.

## Glossary

- **Day Assignments Tab**: The "Day Assignments" section within CalendarSettings component where users assign day types to calendar days
- **Calendar Day**: An individual day entry in the calendar_day table, uniquely identified by UUID
- **Day Type / Classification**: A category or classification assigned to a calendar day (e.g., "Public Holiday", "Saturday")
- **Weekday**: Day of week designation - Sunday, Monday, Tuesday, Wednesday, Thursday, Friday, Saturday
- **Specific Date**: A numeric day number (1-based) within a selected month (e.g., 5 Baisakh = day 5)
- **Selection State**: The current set of days the user has selected for assignment
- **Day Classification Selector**: The UI component displaying available classifications from the backend
- **Bulk Assign API**: The `/v1/calendar-days/bulk-assign` endpoint used to assign multiple days at once
- **Assign by Weekday API**: The `/v1/calendar-days/assign-by-weekday` endpoint used to assign days matching a specific weekday
- **Manual Assign API**: The `/v1/calendar-days/manual-assign` endpoint used to assign specific day numbers
- **Calendar Module**: The view displaying the academic calendar with assignments
- **Calendar Sidebar**: The sidebar panel in the Calendar view that displays assigned day types
- **Backend System**: The Node.js/Express API serving calendar operations
- **Frontend System**: The React frontend component CalendarSettings.jsx

## Requirements

### Requirement 1: Year and Month Selection

**User Story:** As a calendar administrator, I want to select an academic year and month before assigning days, so that I can organize assignments by academic periods.

#### Acceptance Criteria

1. WHEN the user opens the Day Assignments tab, THE CalendarSettings.jsx component SHALL display a Year dropdown selector
2. WHEN the user selects a Year from the dropdown, THE CalendarSettings.jsx component SHALL load available months for that year from the backend
3. WHEN the user selects a Month from the dropdown, THE CalendarSettings.jsx component SHALL populate the Day Selection interface with available dates
4. WHEN no year is selected, THE Month dropdown SHALL remain disabled and display "Select year first"
5. WHEN no month is selected, THE Day Selection interface (both Weekday and Specific Date sections) SHALL remain disabled

### Requirement 2: Weekday Selection Interface

**User Story:** As a calendar administrator, I want to assign a classification to all occurrences of a specific weekday in the selected month, so that I can quickly apply bulk assignments like "All Mondays are working days".

#### Acceptance Criteria

1. WHEN the user selects a Month, THE Day Selection interface SHALL display a "Weekday Selection" dropdown
2. THE Weekday Selection dropdown SHALL contain seven options: Sunday, Monday, Tuesday, Wednesday, Thursday, Friday, Saturday
3. WHEN a user selects a weekday from the dropdown, THE Weekday Selection dropdown value SHALL update to show the selected weekday name
4. WHEN a weekday is selected, THE Day Selection interface SHALL maintain the selection until the user explicitly clears it or selects a different weekday
5. WHEN a weekday is already selected and the user selects a different weekday, THE previous weekday selection SHALL be replaced by the new selection
6. THE Weekday Selection dropdown SHALL display "Clear weekday" when a weekday is currently selected, allowing the user to deselect it

### Requirement 3: Specific Date Selection Interface

**User Story:** As a calendar administrator, I want to assign classifications to specific dates in the month, so that I can handle exceptions and individual day assignments.

#### Acceptance Criteria

1. WHEN the user selects a Month, THE Day Selection interface SHALL display a "Specific Dates" section with a list of available date numbers
2. THE list of available date numbers SHALL be generated dynamically based on the total number of days in the selected month (1 through N)
3. WHEN the user clicks a date number, THE clicked date SHALL be added to the current Selection State
4. WHEN the user clicks an already-selected date number, THE date SHALL be removed from the current Selection State
5. WHEN multiple dates are selected, THE selected dates SHALL be visually highlighted (e.g., different background color)
6. THE Selection State for Specific Dates SHALL be independent and cumulative—the user can select multiple dates without clearing previous selections
7. WHEN the user clicks a "Clear All" or similar button next to the Specific Dates section, THE entire Specific Dates Selection State SHALL be emptied

### Requirement 4: Combined Day Selection State

**User Story:** As a calendar administrator, I want to combine weekday and specific date selections in a single assignment operation, so that I can handle complex assignment patterns efficiently.

#### Acceptance Criteria

1. WHEN the user has selected a weekday AND selected specific dates, THE combined Selection State SHALL include both weekday occurrences and the specific dates
2. WHEN the user assigns with both weekday and specific dates selected, THE backend SHALL receive both types of selections in a single API request
3. THE Day Selection interface SHALL display a summary (e.g., "3 weekday occurrences + 2 specific dates = 5 total days")
4. WHEN the user clears the weekday selection, THE specific dates selection SHALL remain unchanged
5. WHEN the user clears the specific dates selection, THE weekday selection SHALL remain unchanged

### Requirement 5: Day Classification Selection

**User Story:** As a calendar administrator, I want to select from available day classifications, so that I can assign the correct type to the selected days.

#### Acceptance Criteria

1. WHEN the Day Selection interface is active with days selected, THE Classification Selection dropdown SHALL display all available day types loaded from the backend
2. WHEN the backend returns classifications, THE Classification Selection dropdown SHALL populate with classification names (e.g., "Public Holiday", "Working Day")
3. THE Classification Selection dropdown SHALL require a user selection before assignment is allowed
4. WHEN the user selects a classification, THE selected classification name SHALL be displayed in the dropdown
5. IF the backend returns no classifications, THE Classification Selection dropdown SHALL display "No classifications available" and assignment SHALL be disabled

### Requirement 6: Assignment Operation

**User Story:** As a calendar administrator, I want to confirm and save my day and classification selections, so that the assignments are persisted to the database.

#### Acceptance Criteria

1. WHEN the user has selected days AND selected a classification AND clicked the "Assign" button, THE system SHALL prepare an assignment request containing all selected days and the classification ID
2. WHEN the user clicks "Assign", THE system SHALL validate that at least one day is selected
3. WHEN the user clicks "Assign", THE system SHALL validate that a classification is selected
4. IF validation fails, THE system SHALL display an error message specific to the failure (e.g., "Please select at least one day" or "Please select a classification")
5. IF validation passes, THE system SHALL send an API request to the backend combining:
   - Weekday assignments via the `/v1/calendar-days/assign-by-weekday` endpoint
   - Specific date assignments via the `/v1/calendar-days/manual-assign` endpoint
6. WHEN the backend returns a success response, THE system SHALL display a success message (e.g., "5 days assigned successfully")
7. WHEN the backend returns an error response, THE system SHALL display the error message and allow the user to retry

### Requirement 7: Selection UI Feedback

**User Story:** As a calendar administrator, I want immediate visual feedback about my selections, so that I can verify correctness before assignment.

#### Acceptance Criteria

1. WHEN the user selects a weekday, THE UI SHALL display a preview (e.g., "Monday: 4 occurrences in this month")
2. WHEN the user selects specific dates, THE UI SHALL display the count of selected dates (e.g., "Selected 3 dates")
3. WHEN both weekday and specific dates are selected, THE UI SHALL display a combined preview (e.g., "4 Mondays + 3 specific dates = 7 total days")
4. WHEN no days are selected, THE "Assign" button SHALL be disabled with a visual indicator
5. WHEN days are selected but no classification is chosen, THE "Assign" button SHALL be disabled with a visual indicator

### Requirement 8: Sidebar Calendar Integration

**User Story:** As a calendar user, I want the Calendar module sidebar to automatically reflect newly assigned day types, so that I can see assignments immediately after saving.

#### Acceptance Criteria

1. WHEN the user successfully assigns days in CalendarSettings, THE Calendar module sidebar SHALL be updated to show the newly assigned classifications
2. THE Calendar sidebar SHALL display the assigned day type name and its visual color/indicator on the correct calendar dates
3. IF the Calendar module is already open in another tab/view, THE sidebar update MAY require a page refresh (acceptable constraint for MVP)
4. WHEN the user navigates back to the Calendar view after assignment, THE assigned days SHALL be visible with their classifications

### Requirement 9: Existing Functionality Preservation

**User Story:** As a system administrator, I want to ensure that existing calendar features remain functional, so that I don't break any workflows currently in production.

#### Acceptance Criteria

1. THE existing bulk grid assignment interface (Requirement 2 in CalendarSettings) SHALL remain fully operational
2. WHEN the user uses the bulk grid interface to select and assign days, THE assignment SHALL save correctly and persist in the database
3. THE Import/Export CSV functionality (Requirement 3 in CalendarSettings) SHALL remain fully operational
4. WHEN the user exports a calendar month to CSV, THE export SHALL include all days and their classifications regardless of how they were assigned
5. WHEN the user imports a CSV file, THE import SHALL update day classifications correctly
6. THE existing Year, Category, and Classification setup functionality (Requirement 1 in CalendarSettings) SHALL remain fully operational

### Requirement 10: API Integration

**User Story:** As an API consumer, I want to use existing backend endpoints for day assignment, so that I don't require new backend development.

#### Acceptance Criteria

1. WHEN assigning days by weekday, THE system SHALL call the `/v1/calendar-days/assign-by-weekday` endpoint with the correct payload
2. THE assign-by-weekday request payload SHALL include: day_of_week (string), day_type_id (UUID), month_id (UUID)
3. WHEN assigning specific dates, THE system SHALL call the `/v1/calendar-days/manual-assign` endpoint with the correct payload
4. THE manual-assign request payload SHALL include: month_id (UUID), assignments array with day_number and day_type_id pairs
5. IF both weekday and specific dates are selected, THE system SHALL make separate API calls to each endpoint for each assignment type
6. WHEN the backend returns success for both API calls, THE system SHALL treat the operation as complete
7. IF one API call fails while the other succeeds, THE system SHALL display a partial success message (e.g., "Assigned 4 weekday occurrences, but failed to assign 2 specific dates")

### Requirement 11: Error Handling

**User Story:** As a calendar administrator, I want clear error messages when assignments fail, so that I can understand and correct problems.

#### Acceptance Criteria

1. IF the API request fails with a 4xx error, THE system SHALL display a user-friendly error message extracted from the response
2. IF the API request fails with a 5xx error, THE system SHALL display a generic error message (e.g., "Server error occurred. Please try again later.")
3. IF network connectivity is lost, THE system SHALL display a connectivity error message
4. WHEN an error occurs, THE selected days and classification SHALL remain in the UI for the user to retry without re-selecting

### Requirement 12: Performance and Usability

**User Story:** As a calendar administrator, I want the interface to remain responsive with large month selections, so that I can work efficiently regardless of the calendar size.

#### Acceptance Criteria

1. WHEN the user selects 20+ specific dates, THE UI rendering SHALL complete within 200ms
2. WHEN the user switches between months, THE interface SHALL reload within 500ms
3. WHEN the assignment API is processing, THE system SHALL display a loading indicator (e.g., spinner or "Saving...")
4. WHEN the API call completes, THE loading indicator SHALL disappear and the result message SHALL display

