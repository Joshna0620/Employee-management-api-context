# Architecture Documentation

## 1. Architectural Pattern
The application follows a **Layered Architecture** (also known as N-tier architecture):

```
┌─────────────────────────────────┐
│   FastAPI Routers (HTTP Layer)  │
├─────────────────────────────────┤
│   Services (Business Logic)     │
├─────────────────────────────────┤
│   Repositories (Data Access)    │
├─────────────────────────────────┤
│   SQLAlchemy Models (ORM)       │
├─────────────────────────────────┤
│   Database (SQLite)             │
└─────────────────────────────────┘
```

## 2. Layer Responsibilities

### A. Routers Layer (`app/routers/`)
- **Responsibility:** Handle HTTP requests/responses, route requests to services
- **Files:** `employees.py`, `departments.py`
- **Prefix:** `/api/v1/{resource}`
- **Key Functions:**
  - Request validation via Pydantic schemas
  - Dependency injection for database sessions
  - Response serialization using Pydantic models

### B. Services Layer (`app/services/`)
- **Responsibility:** Implement business logic, orchestrate operations
- **Files:** `employee_service.py`, `department_service.py`
- **Key Functions:**
  - Validation logic (e.g., preventing deletion of department with active employees)
  - Business rule enforcement
  - Error handling with HTTPException
  - Calls to repository layer

### C. Repositories Layer (`app/repositories/`)
- **Responsibility:** Abstract database operations, provide CRUD methods
- **Files:** `employee_repo.py`, `department_repo.py`
- **Key Functions:**
  - Create, Read, Update, Delete operations
  - Query filtering (e.g., exclude soft-deleted employees)
  - Direct SQLAlchemy ORM interactions

### D. Models Layer (`app/models/`)
- **Responsibility:** Define database schema using SQLAlchemy ORM
- **Files:** `employee.py`, `department.py`
- **Key Properties:**
  - Table definitions with columns and constraints
  - Relationships and foreign keys
  - Enum types for predefined values

### E. Schemas Layer (`app/schemas/`)
- **Responsibility:** Define request/response data structures using Pydantic
- **Files:** `employee.py`, `department.py`
- **Key Properties:**
  - Request validation (EmployeeCreate, EmployeeUpdate)
  - Response serialization (EmployeeOut)
  - Field validators and root validators
  - Prevents over-posting, ensures data consistency

### F. Utils Layer (`app/utils/`)
- **Responsibility:** Provide reusable utility functions
- **Files:** `pagination.py`, `sorting.py`
- **Key Functions:**
  - `paginate()` – Implements pagination logic
  - `apply_sorting()` – Dynamic sorting on query results

## 3. Data Flow

### Create Employee Flow
```
POST /api/v1/employees
    ↓
Router (employees.py) validates payload via EmployeeCreate schema
    ↓
Service (employee_service.create_employee) enforces business rules
    ↓
Repository (employee_repo.create) inserts into database
    ↓
Response serialized via EmployeeOut schema
    ↓
200 OK with employee data
```

### List Employees Flow
```
GET /api/v1/employees?page=1&page_size=20&sort=salary:desc&department_id=5
    ↓
Router extracts query parameters
    ↓
Service calls repository.get_all()
    ↓
Repository returns SQLAlchemy query (filters soft-deleted)
    ↓
Router applies department_id filter if provided
    ↓
Router applies sorting via apply_sorting()
    ↓
Router applies pagination via paginate()
    ↓
Response with items array and metadata
```

### Soft Delete Flow
```
DELETE /api/v1/employees/{id}
    ↓
Service validates employee exists
    ↓
Repository sets deleted_at = datetime.utcnow()
    ↓
Database updates record (does NOT delete)
    ↓
204 No Content response
```

## 4. Relationships

### Employee ↔ Department (Many-to-One)
- **Foreign Key:** `Employee.department_id` → `Department.id`
- **Relationship:** SQLAlchemy `relationship("Department", back_populates="employees")`
- **Constraint:** Cannot delete department with active (non-soft-deleted) employees

### Employee ↔ Employee (Self-Referencing for Manager)
- **Foreign Key:** `Employee.manager_id` → `Employee.id`
- **Relationship:** SQLAlchemy `relationship("Employee", remote_side=[id])`
- **Use Case:** Hierarchical employee structure (manager-subordinate)

### Department ↔ Manager (One-to-One)
- **Foreign Key:** `Department.manager_id` → `Employee.id`
- **Relationship:** Optional manager assignment
- **Use Case:** Identify department head

## 5. Database Schema

### Employees Table
```sql
CREATE TABLE employees (
    id INTEGER PRIMARY KEY,
    employee_code VARCHAR UNIQUE NOT NULL,
    first_name VARCHAR NOT NULL,
    last_name VARCHAR NOT NULL,
    email VARCHAR UNIQUE NOT NULL,
    phone VARCHAR,
    gender ENUM (Male, Female, Other),
    department_id INTEGER FOREIGN KEY NOT NULL,
    manager_id INTEGER FOREIGN KEY,
    job_title VARCHAR,
    salary FLOAT NOT NULL,
    salary_currency VARCHAR DEFAULT 'INR',
    joining_date DATE NOT NULL,
    employment_status ENUM (Active, On_Leave, Terminated) DEFAULT 'Active',
    is_full_time BOOLEAN DEFAULT TRUE,
    work_location VARCHAR,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    deleted_at DATETIME NULL,
    version INTEGER DEFAULT 1
);
```

### Departments Table
```sql
CREATE TABLE departments (
    id INTEGER PRIMARY KEY,
    code VARCHAR UNIQUE NOT NULL,
    name VARCHAR UNIQUE NOT NULL,
    description VARCHAR,
    manager_id INTEGER FOREIGN KEY,
    location VARCHAR,
    is_active BOOLEAN DEFAULT TRUE,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

## 6. Key Design Patterns

### Repository Pattern
- Abstracts data access logic
- Allows easy switching of database implementations
- Improves testability via mock repositories

### Service Layer Pattern
- Centralizes business logic
- Decouples routes from business rules
- Enables code reuse across multiple routes

### Soft Delete Pattern
- Maintains referential integrity
- Preserves historical data
- Filters by `deleted_at IS NULL` in queries

### Dependency Injection
- FastAPI's `Depends()` for database session injection
- Ensures proper resource cleanup (`finally: db.close()`)
