#  Academic Data Warehouse — MySQL Star Schema

A corporate-grade data warehouse designed and built from scratch for a university course scheduling system. Covers the full pipeline: source analysis → dimensional modelling → ETL → analytical queries.

**1 365 scheduled course sections · 2 campuses · 5 dimension tables · Star Schema**

---

## Schema Design

```
                    dim_course
                        │
   dim_teacher ──── fact_course_schedule ──── dim_classroom
                        │
                    dim_class
                        │
                   dim_department
```

### Star Schema — why not Snowflake?

A **Star Schema** was chosen over Snowflake because the primary use case is analytical aggregations (load by teacher, utilisation by campus, theory vs practice ratios). The flat dimension structure eliminates JOIN chains and speeds up every GROUP BY query without sacrificing query clarity.

---

## Tables

### `fact_course_schedule` — grain: one scheduled section per teacher/class/term

| Column | Type | Notes |
|--------|------|-------|
| `schedule_key` | INT PK | Surrogate key |
| `course_key` | INT FK | → dim_course |
| `teacher_key` | INT FK | → dim_teacher |
| `class_key` | INT FK | → dim_class |
| `classroom_key` | INT FK **nullable** | NULL when room unassigned |
| `dept_key` | INT FK | → dim_department |
| `academic_year` | VARCHAR | e.g. `2024-2025-1` |
| `campus` | VARCHAR | Degenerate dimension |
| `course_nature` | VARCHAR | Required / elective |
| `class_type` | VARCHAR | Theory / practice |
| `total_hours` | INT | Measure |
| `class_size` | INT | Measure — enrolled headcount |
| `course_credits` | DECIMAL | Measure |

### Dimension tables

| Table | Source | Key fields |
|-------|--------|------------|
| `dim_course` | Course catalogue | name, type, hours, credits |
| `dim_teacher` | Teacher profiles | name, title, department |
| `dim_class` | Student cohorts | major, campus, enrollment year |
| `dim_classroom` | Room inventory | type, capacity, building, floor |
| `dim_department` | Department master | code, name, teaching flag |

---

## ETL Design Decisions

- **Surrogate keys** generated via `AUTO_INCREMENT` — natural keys from source are kept as `UNIQUE` constraints for traceability
- **Nullable FK for classrooms** — rooms are sometimes unassigned at scheduling time; `NULL` is semantically correct and avoids a dummy "unknown room" record
- **LIKE-pattern JOIN** for class codes — source `class_id` values include year suffixes (`class_id + year`); resolved with `LIKE` patterns without losing referential integrity
- **Degenerate dimensions** — `academic_year`, `campus`, `course_nature`, `class_type` stored directly in the fact table (low cardinality, no joins needed)

---

## Analytical Queries Supported

```sql
-- Teaching load per teacher
SELECT t.teacher_name, SUM(f.total_hours) AS load
FROM fact_course_schedule f JOIN dim_teacher t USING (teacher_key)
GROUP BY t.teacher_name ORDER BY load DESC;

-- Campus workload balance
SELECT f.campus, COUNT(*) AS sections, SUM(f.class_size) AS students
FROM fact_course_schedule f GROUP BY f.campus;

-- Theory vs practice ratio by department
SELECT d.department_name, f.class_type, COUNT(*) AS sections
FROM fact_course_schedule f JOIN dim_department d USING (dept_key)
GROUP BY d.department_name, f.class_type;

-- Classroom utilisation
SELECT c.room_name, c.capacity, AVG(f.class_size) AS avg_fill
FROM fact_course_schedule f JOIN dim_classroom c USING (classroom_key)
GROUP BY c.room_name, c.capacity;
```

---

## Getting Started

### 1. Create the database
```sql
CREATE DATABASE academic_dw CHARACTER SET utf8mb4;
USE academic_dw;
```

### 2. Load dimension tables first
```sql
-- Example for dim_department
LOAD DATA INFILE 'data/dim_department.csv'
INTO TABLE dim_department
FIELDS TERMINATED BY ',' ENCLOSED BY '"'
LINES TERMINATED BY '\n'
IGNORE 1 ROWS;
```
Load order: `dim_department` → `dim_course` → `dim_teacher` → `dim_classroom` → `dim_class` → `fact_course_schedule`

### 3. Open the Tableau workbook
Open `docs/dashboard.twb` in Tableau Desktop and point the connection to your MySQL instance.

---

## Repository Structure

```
data-warehouse-mysql/
├── data/
│   ├── fact_course_schedule.csv   ← 1 365 records
│   ├── dim_course.csv             ← 1 411 courses
│   ├── dim_teacher.csv            ← 516 teachers
│   ├── dim_class.csv              ← 386 student cohorts
│   ├── dim_classroom.csv          ← 290 rooms
│   └── dim_department.csv         ← 36 departments
├── docs/
│   ├── Data_Warehouse_Report.docx ← Full construction report
│   └── dashboard.twb              ← Tableau workbook
└── README.md
```

---

## Tech Stack

- **Database:** MySQL 8
- **Modelling:** Star Schema (Kimball methodology)
- **Visualisation:** Tableau Desktop
- **Data:** University ERP export (2024–2025, Semester 1)

---

## Author

**Manaeva Daria** · [mandariiii19](https://github.com/mandariiii19)  

