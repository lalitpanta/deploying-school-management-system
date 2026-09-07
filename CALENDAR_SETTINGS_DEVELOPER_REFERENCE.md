# Calendar Settings - Quick Reference for Developers

## Module Location

- Frontend: `frontend/src/components/settings/CalendarSettings.jsx` and `AcademicCalendar.jsx`
- Backend: `backend/src/routing/v1/calendar*.routing.js`, `controller/`, `services/`, `validation/`
- API Client: `frontend/src/api/calendarApi.js`

## Key Components

### Frontend

1. **CalendarSettings.jsx** (NEW - Primary UI)
   - Comprehensive calendar management interface
   - Split into tabs: Setup, Day Assignments, Import/Export
   - Uses inline dark-mode CSS (self-contained)

2. **AcademicCalendar.jsx** (Legacy - Still functional)
   - Original calendar interface
   - Used in Settings page as "Academic Calendar" tab
   - Can be used alongside new CalendarSettings

3. **API Layer** (calendarApi.js)
   - Centralized API calls
   - Consistent endpoint patterns

### Backend

- Year management: `/v1/year/*`
- Month management: `/v1/month/*`
- Calendar days: `/v1/calendar-days/*`
- Day types: `/v1/day/*`
- Categories: `/v1/day-category/*`

---

## API Payload Structures

### Assign by Weekday (Fixed in v1.0)

**Endpoint:** `POST /v1/calendar-days/assign-by-weekday`

**Payload:**

```javascript
{
  day_of_week: "Sunday",      // Required: Full weekday name
  day_type_id: "uuid-here",   // Required: Classification UUID
  month_id: "uuid-here",      // Optional: Month UUID (takes priority)
  year_id: "uuid-here"        // Optional: Year UUID (used if month_id is null)
}
```

**Frontend Call:**

```javascript
const payload = {
  day_of_week: "Sunday",
  day_type_id: classificationId,
  month_id: gridMonthId, // or
  year_id: gridYearId, // if month-wide not applicable
};
await assignByWeekday(payload);
```

### Bulk Assign

**Endpoint:** `POST /v1/calendar-days/bulk-assign`

**Payload:**

```javascript
{
  assignments: [
    {
      calendarDayId: "day-uuid",
      dayTypeId: "type-uuid", // Can be null to unassign
    },
    // ... more assignments
  ];
}
```

### Create Year

**Endpoint:** `POST /v1/year/uploadyear`

**Payload:**

```javascript
{
  year_label: "2083/084",
  year_label_AD: "2026/27",
  year_label_BS: "2083/084",
  start_date_AD: "2026-04-14",  // YYYY-MM-DD
  end_date_AD: "2027-04-13",
  start_date_BS: "2083-01-01",  // Optional
  end_date_BS: "2084-01-01",    // Optional
  is_current: true
}
```

### Create Month

**Endpoint:** `POST /v1/month/uploadmonth`

**Payload:**

```javascript
{
  year_id: "year-uuid",
  month_name: "Baisakh",
  bs_month_index: 1,  // 1-12
  start_date: "2026-04-14",  // YYYY-MM-DD
  end_date: "2026-05-14",
  month_start_date_AD: "2026-04-14",  // Alternative field names
  month_end_date_AD: "2026-05-14",
  date_format: "BS"
}
```

**Note:** Creating a month auto-generates all calendar days for that date range

### Create Day Type

**Endpoint:** `POST /v1/day/uploadday`

**Payload:**

```javascript
{
  day_type: "Holiday",
  category_id: "category-uuid"  // Optional
}
```

### Create Day Category

**Endpoint:** `POST /v1/day-category`

**Payload:**

```javascript
{
  category_name: "Attendance";
}
```

---

## State Management in Components

### CalendarSettings.jsx Key State

```javascript
const [years, setYears] = useState([]); // Academic years
const [months, setMonths] = useState([]); // Academic months
const [dayTypes, setDayTypes] = useState([]); // Classifications
const [categories, setCategories] = useState([]); // Categories
const [calDays, setCalDays] = useState([]); // Month calendar days

const [gridYearId, setGridYearId] = useState(""); // Selected year for grid view
const [gridMonthId, setGridMonthId] = useState(""); // Selected month for grid view
const [bulkSelected, setBulkSelected] = useState(new Set()); // Selected days for bulk operation

const [weekdayRule, setWeekdayRule] = useState({
  dayTypeId: "", // Selected classification
  weekday: "", // Selected weekday index (0-6)
  scope: "month", // "month" or "year"
});
```

---

## Common Operations

### 1. Add Year

```javascript
const handleAddYear = async () => {
  const yearData = {
    year_label: "2084/085",
    year_label_AD: "2027/28",
    year_label_BS: "2084/085",
    start_date_AD: "2027-04-14",
    end_date_AD: "2028-04-13",
    is_current: false,
  };

  try {
    await createYear(yearData);
    toast.success("Year added");
    loadAll(); // Refresh all data
  } catch (e) {
    toast.error("Failed to add year");
  }
};
```

### 2. Add Month (Auto-generates Days)

```javascript
const handleAddMonth = async () => {
  const monthData = {
    year_id: selectedYearId,
    month_name: "Jestha",
    bs_month_index: 2,
    start_date: "2027-05-15",
    end_date: "2027-06-13",
    date_format: "BS",
  };

  try {
    await createMonth(monthData);
    toast.success("Month added with calendar days auto-generated");
    loadAll();
  } catch (e) {
    toast.error("Failed to add month");
  }
};
```

### 3. Assign by Weekday (v1.0+)

```javascript
const handleAssignByWeekday = async () => {
  const WEEKDAYS = [
    "Sunday",
    "Monday",
    "Tuesday",
    "Wednesday",
    "Thursday",
    "Friday",
    "Saturday",
  ];

  const weekdayName = WEEKDAYS[parseInt(weekdayRule.weekday)];

  const payload = {
    day_of_week: weekdayName, // Must be full name!
    day_type_id: weekdayRule.dayTypeId,
    month_id: weekdayRule.scope === "month" ? gridMonthId : null,
    year_id: weekdayRule.scope === "year" ? gridYearId : null,
  };

  try {
    await assignByWeekday(payload); // Single object parameter!
    toast.success("Weekday assignment successful");
    fetchGridDays();
  } catch (e) {
    toast.error("Assignment failed");
  }
};
```

### 4. Bulk Assign Days

```javascript
const handleBulkAssign = async () => {
  const assignments = Array.from(bulkSelected).map((dayId) => ({
    calendarDayId: dayId,
    dayTypeId: bulkTypeId,
  }));

  try {
    await bulkAssignDayTypes(assignments);
    toast.success(`Assigned to ${assignments.length} days`);
    setBulkSelected(new Set());
    fetchGridDays();
  } catch (e) {
    toast.error("Bulk assignment failed");
  }
};
```

---

## Common Fixes/Debugging

### Issue: Weekday Assignment Returns Error

**Check:**

1. Is `day_of_week` a full weekday name? (Not a number!)
2. Are field names correct? (`day_of_week`, `day_type_id`, `month_id`, `year_id`)
3. Is exactly ONE of month_id or year_id provided? (Not both null)

### Issue: Calendar Days Not Showing

**Check:**

1. Did month creation succeed? Calendar days auto-generate
2. Are start/end dates in correct format? (YYYY-MM-DD)
3. Is start date before end date?
4. Click "Refresh" button to reload

### Issue: Year Can't Be Deleted

**Fix:**

- Associated months and calendar days auto-delete (cascade)
- Ensure you have proper permissions

### Issue: Day Types Not Appearing in Dropdown

**Check:**

1. Are day types created? (Must exist before assignment)
2. Did creation API return success?
3. Try clicking "Refresh" button

---

## Testing Checklist

- [ ] Create year with AD and BS labels
- [ ] Create month - verify calendar days auto-generated
- [ ] Create day type classifications
- [ ] Assign day to single day
- [ ] Assign multiple days via bulk selection
- [ ] **Assign by weekday** (most complex - test thoroughly)
  - [ ] Test month-scope assignment
  - [ ] Test year-scope assignment
  - [ ] Verify all matching weekdays get assigned
- [ ] Edit year and month
- [ ] Delete classifications
- [ ] Delete month (verify cascade delete works)
- [ ] Delete year (verify full cascade)
- [ ] Switch between BS/AD views
- [ ] Export calendar data
- [ ] Clear/unassign days

---

## Performance Notes

- Calendar days query can be expensive for full years
- Batch inserts used for month day generation
- Consider pagination for large calendars
- Cascade deletes may be slow for large datasets

---

## Migration Compatibility

- Year table: Added AD/BS columns in migration `20260525000000`
- Month table: Uses `month_class_data` table
- Calendar days: Auto-generated via migrations
- Day types: Stored in `day_classification` table
- Categories: Stored in `day_category` table

---

## Future Improvements

- [ ] Calendar import/export functionality (CSV)
- [ ] Duplicate month/year templates
- [ ] Holiday calendars from external sources
- [ ] Multi-school calendar synchronization
- [ ] Mobile responsive grid view
- [ ] Undo/redo for bulk operations
- [ ] Scheduled task reminders
- [ ] Calendar sharing with teachers
