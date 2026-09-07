# Bugfix Requirements Document

## Introduction

The CalendarSettings component's Setup tab stores academic year configuration, day categories, and day classifications exclusively in React state using `useState` hooks. This causes all setup data to be lost on page refresh, as there is no integration with the backend API to persist data to the PostgreSQL database. While the backend endpoints and database tables already exist, the frontend does not invoke these APIs for CRUD operations in the Setup tab, resulting in data volatility and a broken user experience.

This bugfix ensures the Setup tab data flows through the complete stack: Component → API Client → Backend Controller → Service → Database, and that existing data is loaded on component mount.

---

## Bug Analysis

### Current Behavior (Defect)

**1.1** WHEN a user adds an academic year (with label, start/end dates, current year flag) in the Setup tab THEN the system stores the data only in React state (`years` array) without making any API call to persist it to the database.

**1.2** WHEN a user adds a day category (e.g., "Holiday", "Working Day") in the Setup tab THEN the system stores the data only in React state (`categories` array) without making any API call to persist it to the database.

**1.3** WHEN a user adds a day classification (e.g., "Public Holiday", "Saturday") with a category mapping in the Setup tab THEN the system stores the data only in React state (`classifications` array) without making any API call to persist it to the database.

**1.4** WHEN a user refreshes the page or navigates away and returns THEN the system loses all Setup tab data because React state is reset and no data is fetched from the backend on component mount.

**1.5** WHEN a user deletes an academic year, day category, or day classification in the Setup tab THEN the system removes it from React state but does not delete it from the database (if it was ever saved).

**1.6** WHEN the CalendarSettings component mounts THEN the system does not fetch existing academic years, day categories, or day classifications from the backend, leaving the Setup tab empty even if data exists in the database.

### Expected Behavior (Correct)

**2.1** WHEN a user adds an academic year with label, start/end dates, and current year flag in the Setup tab THEN the system SHALL immediately call the backend API (`createYear` or `seedNepaliYear`) to persist the year to the database and update React state with the response data including the database-generated ID.

**2.2** WHEN a user adds a day category in the Setup tab THEN the system SHALL immediately call the backend API (`createDayCategory`) to persist the category to the database and update React state with the response data including the database-generated ID.

**2.3** WHEN a user adds a day classification with category mapping in the Setup tab THEN the system SHALL immediately call the backend API (`createDayType`) to persist the classification to the database and update React state with the response data including the database-generated ID.

**2.4** WHEN a user refreshes the page or navigates away and returns THEN the system SHALL preserve all Setup tab data because it is stored in the database and fetched on component mount via API calls.

**2.5** WHEN a user deletes an academic year, day category, or day classification in the Setup tab THEN the system SHALL call the appropriate backend API (`deleteYear`, `deleteDayCategory`, `deleteDayType`) to remove it from both the database and React state.

**2.6** WHEN the CalendarSettings component mounts THEN the system SHALL fetch existing academic years via `getYears()`, day categories via `getDayCategories()`, and day classifications via `getDayTypes()` from the backend and populate React state with the fetched data.

**2.7** WHEN a user marks a year as current THEN the system SHALL ensure only one year per mode (BS/AD) is marked as current by updating the database via the backend API.

**2.8** WHEN the component loads existing data from the backend THEN the system SHALL transform the data structure to match the component's state schema (handling ID mapping, date formats, and nested relationships).

### Unchanged Behavior (Regression Prevention)

**3.1** WHEN a user interacts with the Day Assignments tab (Tab 2) THEN the system SHALL CONTINUE TO use the existing API integration (`getCalendarDays`, `bulkAssignDayTypes`, `assignByWeekday`, etc.) without any changes to that functionality.

**3.2** WHEN a user interacts with the Import/Export tab (Tab 3) THEN the system SHALL CONTINUE TO use the existing CSV export/import functionality without any changes.

**3.3** WHEN backend APIs receive requests for year, day category, or day classification operations THEN the system SHALL CONTINUE TO validate data, enforce constraints, and return appropriate status codes and error messages as currently implemented.

**3.4** WHEN the backend services interact with the database THEN the system SHALL CONTINUE TO use the multi-tenant pool architecture and handle transactions correctly.

**3.5** WHEN existing API endpoints are called with valid data THEN the system SHALL CONTINUE TO return the same response structure and status codes (201 for create, 200 for read/update/delete, 404 for not found, 400 for validation errors).

**3.6** WHEN the mode toggle switches between BS and AD THEN the system SHALL CONTINUE TO filter and display year data appropriately based on the selected mode without affecting the persistence logic.

**3.7** WHEN the popup calendar is used to select academic start/end dates THEN the system SHALL CONTINUE TO function as implemented without changes to the calendar picker UI or date calculation logic.

**3.8** WHEN the seedNepaliYear endpoint is used to create a complete BS year with all 12 months THEN the system SHALL CONTINUE TO generate month records automatically as currently implemented.
