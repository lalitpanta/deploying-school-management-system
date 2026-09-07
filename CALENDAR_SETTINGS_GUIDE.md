# Calendar Settings Module - Complete Guide

## What is the Calendar Settings Module?

The Calendar Settings module is a comprehensive system for managing academic calendars in the School Management System. It allows schools to:

1. **Define Academic Years** - Create and manage academic years with both AD (Gregorian) and BS (Nepali) calendar systems
2. **Create Academic Months** - Divide years into months with specific date ranges
3. **Classify Days** - Mark days as holidays, working days, exam days, etc.
4. **Apply Rules** - Assign classifications to individual days, multiple days, or all occurrences of specific weekdays
5. **Track Statistics** - Monitor and refresh yearly category statistics

---

## Module Features

### 1. **Setup & Configuration Tab**

#### Academic Years Section

- **Add New Year**: Create academic years with:
  - Year label (e.g., "2083/084")
  - AD year label (e.g., "2026/27")
  - BS year label (e.g., "2083/084")
  - Start and end dates (AD format)
  - Mark as current year
- **View & Edit**: List all created years with edit and delete options
- **Current Year Indicator**: Shows which year is currently active

#### Academic Months Section

- **Add New Month**: Create months within a year with:
  - Select academic year
  - Choose BS month (Baisakh, Jestha, etc.)
  - Set period start and end dates
  - Auto-generates calendar days for the date range
- **View & Edit**: List months grouped by selected year
- **Date Handling**: Automatically calculates AD/BS date conversions

#### Day Categories Section

- **Create Categories**: Define classification categories (e.g., "Attendance", "Assessment", "Holidays")
- **View & Manage**: List and delete categories
- **Link to Day Types**: Categories can be assigned to day types for better organization

#### Day Classifications Section

- **Create Day Types**: Define day classifications like:
  - Holiday (national, religious, etc.)
  - Working Day
  - Exam Day
  - Weekend
  - School Closure
- **Assign Categories**: Link classifications to categories for better organization
- **View & Delete**: Manage all created day types

### 2. **Day Assignments Tab**

#### Calendar Grid View

- **Select Year & Month**: Choose which month's calendar to view
- **Interactive Grid**: 7-column calendar showing:
  - Day number
  - Assigned day type (with color coding)
  - Quick clear button for individual assignments
- **Bulk Selection**: Click multiple days to select them
- **Bulk Assignment**: Assign the same type to all selected days at once

#### Weekday Rules

- **Assign by Weekday**: Apply classifications to all occurrences of a specific weekday
  - Select day type (Holiday, Working Day, etc.)
  - Choose target weekday (Sunday, Monday, etc.)
  - Apply to: current month or entire year
  - Example: Mark all Fridays as holidays
- **Scope Options**:
  - **Month**: Affects only current selected month
  - **Year**: Affects entire academic year

#### Statistics

- **Refresh Yearly Stats**: Recalculate category distribution across the year
- Shows working days, holidays, and unassigned days count

### 3. **Import / Export Tab**

#### Export Calendar

- Download month data as CSV
- Useful for backup and sharing

#### Import Calendar

- Upload CSV file to bulk update day assignments
- Pre-formatted CSV required

---

## How It Works: Complete Workflow

### Step 1: Create an Academic Year

1. Go to Settings → Calendar Settings
2. Click "Setup & Configuration" tab
3. In "Academic Years" section:
   - Enter Year Label: "2083/084"
   - Enter AD Label: "2026/27"
   - Enter BS Label: "2083/084"
   - Set Start Date: 2026-04-14
   - Set End Date: 2027-04-13
   - Check "Mark as current year"
   - Click "Add year"

### Step 2: Create Day Classifications

1. Go to "Day Classifications" section
2. Create types like:
   - Holiday
   - Working Day
   - Exam Day
   - Weekend
3. Optionally assign to categories

### Step 3: Add Academic Months

1. Go to "Academic Months" section
2. For each month:
   - Select the year created in Step 1
   - Choose BS month (e.g., Baisakh)
   - Set start date: 2026-04-14
   - Set end date: 2026-05-14
   - Click "Add month"
   - **System auto-generates calendar days for this date range**

### Step 4: Assign Day Classifications

1. Go to "Day Assignments" tab
2. Select year and month from dropdowns
3. Calendar grid displays all days
4. **Option A - Single Day**: Click "X" on day to clear, or use bulk below
5. **Option B - Multiple Days**:
   - Click multiple days to select
   - Select day type from dropdown
   - Click "Assign"
6. **Option C - By Weekday** (Most Powerful):
   - Select Day Type
   - Select Weekday (e.g., Friday)
   - Choose Scope (Month or Year)
   - Click "Apply"
   - All Fridays get the classification instantly

### Step 5: View & Manage Calendar

1. The calendar now shows color-coded days by classification
2. Badge on each day shows the type abbreviation
3. Hover for quick clear option
4. System maintains full audit trail of changes

---

## Data Structure

### Years Table

```javascript
{
  id: UUID,
  year_label: "2083/084",
  year_label_AD: "2026/27",
  year_label_BS: "2083/084",
  start_date_AD: Date,
  end_date_AD: Date,
  start_date_BS: String,
  end_date_BS: String,
  is_current: Boolean
}
```

### Months Table

```javascript
{
  id: UUID,
  year_id: UUID,
  month_name: "Baisakh",
  bs_month_index: 1,
  month_start_date_AD: Date,
  month_end_date_AD: Date,
  start_date: Date,
  end_date: Date
}
```

### Calendar Days Table

```javascript
{
  id: UUID,
  year_id: UUID,
  month_id: UUID,
  day_number: 1,
  day_of_week: "Sunday",
  day_type_id: UUID,  // Null if unassigned
  created_at: Timestamp,
  updated_at: Timestamp
}
```

### Day Types (Classifications) Table

```javascript
{
  id: UUID,
  day_type: "Holiday",
  category_id: UUID,  // Optional
  tenant_id: UUID
}
```

---

## API Endpoints Reference

### Year Operations

- `POST /v1/year/uploadyear` - Create year
- `GET /v1/year/getyear` - Get all years
- `PATCH /v1/year/updateyear/:id` - Update year
- `DELETE /v1/year/deleteyear/:id` - Delete year

### Month Operations

- `POST /v1/month/uploadmonth` - Create month (auto-generates calendar days)
- `GET /v1/month/getmonth` - Get all months
- `PATCH /v1/month/updatemonth/:id` - Update month
- `DELETE /v1/month/deletemonth/:id` - Delete month

### Calendar Days Operations

- `GET /v1/calendar-days/month?month_id=xxx&date_format=BS` - Get calendar with assignments
- `PATCH /v1/calendar-days/:id/assign-type` - Assign type to single day
- `POST /v1/calendar-days/bulk-assign` - Bulk assign types
- `POST /v1/calendar-days/assign-by-weekday` - Assign all matching weekdays
- `POST /v1/calendar-days/refresh-stats/:year_id` - Refresh yearly stats

### Day Type Operations

- `POST /v1/day/uploadday` - Create day type
- `GET /v1/day/getday` - Get all day types
- `DELETE /v1/day/deleteday/:id` - Delete day type

### Category Operations

- `POST /v1/day-category` - Create category
- `GET /v1/day-category` - Get all categories
- `DELETE /v1/day-category/:id` - Delete category

---

## Recent Fixes (v1.0)

### Issues Resolved

1. ✅ **Weekday Assignment API** - Fixed field naming and payload structure
   - `day_of_week` (string: "Sunday") instead of `weekday` (number: 0)
   - `day_type_id` instead of `dayType`
   - `month_id` instead of `monthId`

2. ✅ **API Function Signature** - Updated to use consistent payload object
   - Old: `assignByWeekday(month_id, weekday, typeId, yearId)` (positional)
   - New: `assignByWeekday({ day_of_week, day_type_id, month_id, year_id })` (object)

3. ✅ **Weekday Name Conversion** - Added automatic conversion from index to name
   - Frontend converts: 0→"Sunday", 1→"Monday", etc.

### Files Updated

- `frontend/src/api/calendarApi.js`
- `frontend/src/components/settings/CalendarSettings.jsx`
- `frontend/src/components/settings/AcademicCalendar.jsx`

---

## Color Coding System

Days are color-coded based on their classification:

- 🔴 **Red** - Holidays (critical/national holidays)
- 🟢 **Green** - Working Days
- 🟡 **Yellow** - Exam Days
- 🔵 **Blue** - Weekend/Non-working days
- ⚪ **White/Gray** - Unassigned

---

## Best Practices

1. **Always Create Year First** - Set up the academic year before adding months
2. **Define Day Types Early** - Create all classifications before assigning days
3. **Use Weekday Rules** - For recurring assignments (all Fridays), use weekday rules instead of manual assignment
4. **Refresh Stats** - After bulk changes, click "Refresh Stats" to update category counts
5. **Backup Calendar** - Export calendar as CSV before major changes
6. **Test with Sample Data** - Create test year to verify before school year starts

---

## Troubleshooting

**Q: Calendar days not showing?**

- A: Ensure month dates are set correctly in AD format (Gregorian)
- A: Click "Refresh" button to reload calendar

**Q: Weekday assignment not working?**

- A: Make sure you selected a day type first
- A: Verify you selected either month or year
- A: Check that the weekday/day type combination is valid

**Q: Can't delete year?**

- A: Months and calendar days will be deleted automatically (cascade delete)
- A: Ensure you have admin permissions

**Q: BS/AD switching not showing changes?**

- A: The calendar shows the same data; format is just for display
- A: Try clicking "Refresh" to reload

---

## Integration with Other Modules

The Calendar Settings data is used by:

- **Attendance Module** - Marks working vs. holiday days
- **Assessment Module** - Identifies exam periods
- **Report Generation** - Filters by school calendar
- **Timetable** - Respects academic calendar structure

---

## Version History

- **v1.0** (Current)
  - Complete calendar management system
  - Support for AD/BS calendar systems
  - Weekday-based bulk assignment
  - Multi-tenant support
  - Import/Export functionality
