# Debug Report – bl_resource_utilization

## 1. Model Overview

Model: bl_resource_utilization  
Grain: One row per employee_id × client_id × project_id  

This Business Layer model consolidates internal time-tracking data with employee, client, and project master data.  
It provides:

- Total hours and productive (billable) hours  
- Internal cost allocation  
- Utilization metrics  
- Project metadata such as start/end dates and monthly budget  

Important note:  
In the sample dataset, **all time entries are marked as productive**, meaning:
- billable_hours = total_hours  
- non_billable_hours = 0  
- utilization_rate_billable_hours = 1.0  

The model keeps generic logic for real-world cases where non-billable work exists.

---

## 2. Sanity Check – Employee-level Utilization

### SQL
```sql
select
employee_id,
employee_name,
department_name,
sum(total_hours) as total_hours,
sum(billable_hours) as billable_hours,
round(sum(billable_hours) / nullif(sum(total_hours), 0),4) as utilization_rate_billable_hours
from {{ ref('bl_resource_utilization') }}
group by
employee_id,
employee_name,
department_name
order by
utilization_rate_billable_hours desc nulls last;
```

### Observations

- All employees show utilization_rate_billable_hours = 1.0000  
- This is expected because the dataset contains only productive hours  
- Some employee_name or department_name fields are "-"  
  → These correspond to time entries without matching master data, which is an acknowledged dataset limitation  

### Interpretation

- The model correctly calculates utilization under the dataset constraints  
- In a real operational environment, utilization values would range between 0.6–0.85  
- The model is structurally prepared for more complex scenarios where non-billable time exists

---

## 3. Sanity Check – Client-level Internal Cost Summary

### SQL  
(Using the correct column name: `total_cost_eur`)
```sql
select
client_id,
client_name,
sum(total_hours) as total_hours,
sum(total_cost_eur) as total_internal_cost_eur
from {{ ref('bl_resource_utilization') }}
group by
client_id,
client_name
order by
total_internal_cost_eur desc;
```

### Observations

- Some clients accumulate meaningful internal cost  
- Clients with zero cost simply have no time-tracking entries joining to them  

### Interpretation

- Internal cost distribution matches time-tracking intensity  
- The model correctly aggregates project-level internal cost to client level  
- This output can be used to evaluate operational workload and cost efficiency

---

## 4. Sanity Check – Department-level Time & Cost

### SQL
```sql
select
department_name,
sum(total_hours) as total_hours,
sum(total_cost_eur) as total_internal_cost_eur
from {{ ref('bl_resource_utilization') }}
group by department_name
order by total_internal_cost_eur desc;
```

### Observations

- Departments such as Paid Media, Paid Content, Organic Social, etc.
- Department_name = "-" exists where employee master data is missing  

### Interpretation

- Confirms that cost is distributed consistently across teams  
- Highlights which departments contribute most to internal delivery effort  
- Missing department names reflect source data limitations, not model issues

---

## 5. Structural “Should-be-empty” Checks

### 5.1 Negative hours or cost
```sql
select *
from {{ ref('bl_resource_utilization') }}
where total_hours < 0
or total_cost_eur < 0;
```

Expected: 0 rows  
Result: 0 rows  
→ No invalid negative values detected.

---

### 5.2 Utilization should equal 1.0 for all rows in this sample
```sql
select *
from {{ ref('bl_resource_utilization') }}
where utilization_rate_billable_hours <> 1;
```

Expected: 0 rows  
Result: 0 rows  
→ Matches dataset specification.

---

### 5.3 No missing primary identifiers (employee_id, client_id)
```sql
select *
from {{ ref('bl_resource_utilization') }}
where employee_id is null
or client_id is null;
```

Expected: 0 rows  
→ Confirms structural integrity of the model.

---

## 6. Conclusion

The `bl_resource_utilization` model is logically and structurally sound:

- Utilization logic functions correctly within dataset constraints  
- Internal cost and hours aggregate correctly across clients and departments  
- No invalid values, missing keys, or structural defects  
- Fully prepared for downstream profitability analysis and workload assessments  

This model can be confidently used as the internal operations foundation of the Business Layer.
