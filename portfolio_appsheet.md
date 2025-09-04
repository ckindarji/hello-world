# Portfolio Management AppSheet Specification

## Data Model

### Tables
- **Projects**: Tracks each initiative.
  - `ProjectCode` (Text, key)
  - `Name` (Text)
  - `Complexity` (Enum: Simple, Medium, Complex, Very Complex)
  - `PM` (Ref to ProjectManagers)
  - `TechLead` (Ref to TechLeads)
  - `StartDate` (Date)
  - `EndDate` (Date)
  - Budget fields: `BudgetInternal`, `BudgetExternal`, `BudgetVendors`, `BudgetHardware`, `BudgetSoftware`, `BudgetLicenses`, `BudgetContingency`
  - Actual fields: `ActualInternal`, `ActualExternal`, `ActualVendors`, `ActualHardware`, `ActualSoftware`, `ActualLicenses`, `ActualContingency`
  - Virtual columns: `PMWorkload`, `TLWorkload`, `BudgetStatus`

- **ProjectManagers**:
  - `Name` (Text, key)
  - `Level` (Number 1-4, L1=junior, L4=expert)
  - `Available` (Yes/No)

- **TechLeads**:
  - `Name` (Text, key)
  - `Level` (Number 1-4)
  - `Available` (Yes/No)

## Assignment Logic

### Required Level per Complexity
```
SWITCH([Complexity],
  "Simple", 1,
  "Medium", 2,
  "Complex", 3,
  "Very Complex", 4
)
```
Use this formula in a virtual column `RequiredLevel`.

### PM Valid_If
Allows only available PMs at or above required level with fewer than 5 projects.
```
SELECT(ProjectManagers[Name],
  AND(
    [Available],
    [Level] >= [_THISROW].[RequiredLevel],
    COUNT(SELECT(Projects[ProjectCode], [PM] = [Name])) < 5
  )
)
```

### Tech Lead Valid_If
Similar to PM but uses TechLeads table.
```
SELECT(TechLeads[Name],
  AND(
    [Available],
    [Level] >= [_THISROW].[RequiredLevel],
    COUNT(SELECT(Projects[ProjectCode], [TechLead] = [Name])) < 5
  )
)
```

### Workload Calculations
```
PMWorkload := SWITCH([Complexity],
  "Simple", 0.20,
  "Medium", 0.40,
  "Complex", 0.60,
  "Very Complex", 0.80
)

TLWorkload := SWITCH([Complexity],
  "Simple", 0.10,
  "Medium", 0.25,
  "Complex", 0.40,
  "Very Complex", 0.55
)
```
Use slices or dashboards to view total workload per person.

### Duplicate Project Codes
`ProjectCode` Valid_If:
```
COUNT(SELECT(Projects[ProjectCode], [ProjectCode] = [_THIS])) = 1
```

### Unassigned Projects
Slice `UnassignedProjects` with Row filter:
```
OR(ISBLANK([PM]), ISBLANK([TechLead]))
```

## Budget Monitoring
For each budget/actual pair, add a virtual column returning **OK**, **Warning**, or **Critical**.
Example for Internal budget:
```
IF([ActualInternal] > [BudgetInternal] * 1.2, "Critical",
  IF([ActualInternal] > [BudgetInternal] * 1.1, "Warning", "OK")
)
```
Create similar columns for the other categories. Another virtual column `BudgetStatus` can aggregate the worst status across all categories:
```
MAX(LIST(
  [InternalStatus], [ExternalStatus], [VendorStatus],
  [HardwareStatus], [SoftwareStatus], [LicenseStatus], [ContingencyStatus]
))
```
This ensures a single overall budget indicator.

## Lifecycle Validation
Ensure `[StartDate] <= [EndDate]` with a Valid_If on `EndDate`:
```
[_THIS] >= [StartDate]
```

## Reporting
- **Duplicate project codes**: view using a slice with filter `COUNT(SELECT(Projects[ProjectCode], [ProjectCode] = [_THISROW].[ProjectCode])) > 1`.
- **Unassigned projects**: use the `UnassignedProjects` slice above.
- **Budget overspend**: slice filtering rows where any status is `"Warning"` or `"Critical"`.

This specification can be implemented in AppSheet by creating the tables, columns, virtual columns, and slices with the provided expressions.
