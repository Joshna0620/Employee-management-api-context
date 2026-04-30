# GitHub Copilot Instructions & Rules

This document provides explicit rules and guidelines for GitHub Copilot to follow when generating code for the Employee Management API.

## 1. Architecture Rules

### Layer Responsibilities (MANDATORY)
- **DO NOT** put business logic in routers (HTTP handlers only)
- **DO NOT** put database queries directly in routers or services (use repositories)
- **ALWAYS** put business logic in services (validation, rules enforcement)
- **ALWAYS** use repositories for database operations
- **ALWAYS** use SQLAlchemy ORM (no raw SQL)

### Correct Layer Pattern
```
Router (FastAPI) → Service (Business Logic) → Repository (Database Access) → Model (ORM)
```

### Incorrect Patterns to Avoid
```
# ✗ WRONG: Database query in router
@router.get("/{id}")
def get_employee(id: int, db: Session = Depends(get_db)):
    return db.query(Employee).filter(Employee.id == id).first()

# ✓ CORRECT: Use service which delegates to repository
@router.get("/{id}")
def get_employee(id: int, db: Session = Depends(get_db)):
    return service.get_employee(db, id)  # Service handles logic & error handling
```

## 2. Naming & Conventions (MANDATORY)

### Class Naming
- Repository classes: `*Repository` (e.g., `EmployeeRepository`)
- Service classes: `*Service` (e.g., `EmployeeService`)
- Response schemas: `*Out` (e.g., `EmployeeOut`)
- Request schemas: `*Create`, `*Update` (e.g., `EmployeeCreate`, `EmployeeUpdate`)

### Function Naming
- Data retrieval methods: `get_*()`, `get_all()`
- Data creation methods: `create_*()`
- Data update methods: `update_*()`
- Data deletion methods: `delete_*()` or `soft_delete_*()`

### Variable Naming
- Use full names: `employee_id` not `emp_id`
- Use full names: `department_id` not `dept_id`
- Boolean variables: `is_active`, `can_delete`, `is_full_time`
- Database models: singular form (`Employee`, `Department`, not `Employees`)

## 3. Error Handling (MANDATORY)

### HTTP Exceptions
- **ALWAYS** use `HTTPException` from FastAPI for error responses
- **ALWAYS** include appropriate HTTP status codes
- **ALWAYS** include descriptive error messages in `detail` field

```python
# ✓ CORRECT
from fastapi import HTTPException

if not employee:
    raise HTTPException(status_code=404, detail="Employee not found")

if salary < 0:
    raise HTTPException(status_code=400, detail="Salary cannot be negative")

if db.query(Employee).filter(...).count() > 0:
    raise HTTPException(status_code=409, detail="Cannot delete department with active employees")
```

### Status Code Usage
- `200 OK` – Successful GET, PUT
- `201 Created` – Successful POST (implicitly set in FastAPI for create endpoints)
- `204 No Content` – Successful DELETE
- `400 Bad Request` – Validation error
- `404 Not Found` – Resource not found
- `409 Conflict` – Business rule violation

### No Silent Failures
- **DO NOT** return None silently when resource not found
- **ALWAYS** raise HTTPException with appropriate status code
- **ALWAYS** check if resource exists before operations

## 4. Soft Delete Standards (MANDATORY)

### Employee Soft Delete Pattern
```python
# Soft delete implementation (in repository)
def soft_delete(self, db: Session, emp):
    from datetime import datetime
    emp.deleted_at = datetime.utcnow()
    db.commit()
```

### Query Filtering for Soft Deletes
```python
# ✓ CORRECT: Always filter out soft-deleted records
def get(self, db: Session, emp_id: int):
    return db.query(Employee).filter(
        Employee.id == emp_id,
        Employee.deleted_at.is_(None)  # Active employees only
    ).first()

def get_all(self, db: Session):
    return db.query(Employee).filter(Employee.deleted_at.is_(None))

# ✗ WRONG: Returning soft-deleted records
def get(self, db: Session, emp_id: int):
    return db.query(Employee).filter(Employee.id == emp_id).first()
```

### Department Delete Validation
```python
# Before deleting department, check for active employees
def delete_department(self, db: Session, dept_id: int):
    dept = self.get_department(db, dept_id)
    
    active_employees = db.query(Employee).filter(
        Employee.department_id == dept_id,
        Employee.deleted_at.is_(None)  # Count only active
    ).count()
    
    if active_employees > 0:
        raise HTTPException(
            status_code=409,
            detail="Cannot delete department with active employees"
        )
    
    repo.delete(db, dept)  # Hard delete is okay for departments
```

## 5. Pagination & Filtering (MANDATORY)

### Pagination Rules
```python
# Pagination parameters: page, page_size
# Defaults: page=1, page_size=20
# Constraints: page >= 1, 1 <= page_size <= 100

# Special rule: If only one parameter provided, use it for both
# - page=5 (no page_size) → Use page_size=5
# - page_size=25 (no page) → Use page=25
# - page=2, page_size=10 → Use as-is
```

### Implementation Pattern
```python
def list_resources(page: Optional[int] = None, page_size: Optional[int] = None):
    # Apply default or cross-assignment logic
    if page is None and page_size is None:
        page, page_size = 1, 20
    elif page is not None and page_size is None:
        page_size = page
    elif page is None and page_size is not None:
        page = page_size
    
    query = service.repo.get_all(db)
    return paginate(query, page, page_size)
```

### Sorting Rules
```python
# Sorting format: "field:direction[,field:direction]"
# Example: "salary:desc,first_name:asc"
# Direction: "asc" (default) or "desc"

# Use apply_sorting() utility
query = apply_sorting(query, Employee, "salary:desc,first_name:asc")
```

## 6. Validation Standards (MANDATORY)

### Field-Level Validators
```python
from pydantic import validator, EmailStr

class EmployeeCreate(BaseModel):
    salary: float
    
    @validator("salary")
    def salary_non_negative(cls, v):
        if v < 0:
            raise ValueError("Salary cannot be negative")
        return v
```

### Root-Level Validators
```python
from pydantic import root_validator
from datetime import datetime

@root_validator(pre=True)
def generate_employee_code(cls, values):
    if not values.get("employee_code"):
        values["employee_code"] = f"EMP-{int(datetime.utcnow().timestamp())}"
    return values
```

### Custom Domain Validators
```python
@validator("email")
def validate_email_domain(cls, v):
    allowed_domains = ["email.com", "yahoo.com"]
    domain = v.split("@")[1]
    if domain not in allowed_domains:
        raise ValueError("Email must be from email.com or yahoo.com")
    return v
```

## 7. Database & ORM Standards (MANDATORY)

### Transaction Management
```python
# ✓ CORRECT: Explicit commit/rollback
def create(self, db: Session, data):
    obj = Employee(**data)
    db.add(obj)
    db.commit()  # Explicit commit
    db.refresh(obj)  # Get assigned id and defaults
    return obj

# ✓ CORRECT: Session cleanup
def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

### Query Construction
```python
# ✓ CORRECT: Use ORM for queries
query = db.query(Employee).filter(Employee.department_id == dept_id)

# ✗ WRONG: Raw SQL
query = db.execute("SELECT * FROM employees WHERE department_id = ?", (dept_id,))
```

### Relationship Usage
```python
# ✓ CORRECT: Navigate relationships
employee.department.name  # Access related department

# Use relationship() in models
class Employee(Base):
    department = relationship("Department", back_populates="employees")
```

## 8. API Endpoint Standards (MANDATORY)

### Endpoint Patterns
```python
# ✓ CORRECT prefix and route structure
@router.post("/", response_model=EmployeeOut)
@router.get("/{id}", response_model=EmployeeOut)
@router.get("/")
@router.put("/{id}", response_model=EmployeeOut)
@router.delete("/{id}", status_code=204)

# Router prefix: /api/v1/employees or /api/v1/departments
```

### Response Models
```python
# ✓ CORRECT: Use response_model for explicit serialization
@router.get("/{id}", response_model=EmployeeOut)
def get_employee(id: int, db: Session = Depends(get_db)):
    return service.get_employee(db, id)

# ✓ CORRECT: Use from_attributes for SQLAlchemy models
class EmployeeOut(BaseModel):
    class Config:
        from_attributes = True  # Allows ORM model to Pydantic conversion
```

## 9. Testing Standards (MANDATORY)

### Unit Test Pattern
```python
# Test file: tests/unit/test_*.py
# Fixture usage: pass fixtures as parameters
def test_validator_rejects_negative_salary():
    with pytest.raises(ValidationError):
        EmployeeCreate(salary=-1000, ...)
    assert "Salary cannot be negative" in str(exc_info.value)
```

### Integration Test Pattern
```python
# Test file: tests/integration/test_*_service.py
# Use fixtures for setup
def test_create_employee_success(employee_service, test_db, sample_department):
    emp = employee_service.create_employee(test_db, data)
    assert emp.id is not None
```

### API Test Pattern
```python
# Test file: tests/api/test_*_routes.py
# Use TestClient and dependency overrides
from fastapi.testclient import TestClient

def test_endpoint(test_db):
    app.dependency_overrides[get_db] = lambda: test_db
    response = client.post("/api/v1/employees", json=payload)
    assert response.status_code == 200
```

## 10. Code Generation Checklist

When generating new code, follow this checklist:

- [ ] Is business logic in Service layer (not Router)?
- [ ] Are database queries in Repository layer (not Service)?
- [ ] Does every resource retrieval raise HTTPException if not found?
- [ ] Are soft-deleted records filtered out from queries?
- [ ] Does code use proper naming conventions (`*Repository`, `*Service`, `*Out`)?
- [ ] Are all validators defined for input schemas?
- [ ] Does code use pagination/sorting utilities where applicable?
- [ ] Are HTTP status codes correct (200, 201, 204, 400, 404, 409)?
- [ ] Is error detail message descriptive?
- [ ] Are type hints included for all function parameters and returns?
- [ ] Are relationships properly configured with `relationship()` in models?
- [ ] Does code use `from_attributes = True` in response schemas?
- [ ] Are session cleanup and transaction management correct?
- [ ] Are unit tests and integration tests written?

## 11. Example: New Feature Implementation

### Requirement: Add "List Departments with Pagination"

**Step 1: Router**
```python
@router.get("/")
def list_departments(
    page: Optional[int] = None,
    page_size: Optional[int] = None,
    db: Session = Depends(get_db)
):
    if page is None and page_size is None:
        page, page_size = 1, 20
    elif page is not None and page_size is None:
        page_size = page
    elif page is None and page_size is not None:
        page = page_size
    
    query = service.repo.get_all(db)
    return paginate(query, page, page_size)
```

**Step 2: Service**
```python
def list_departments(self, db: Session):
    return self.repo.get_all(db)  # Repository handles query
```

**Step 3: Repository**
```python
def get_all(self, db: Session):
    return db.query(Department)  # Returns query, not results
```

**Step 4: Test**
```python
def test_list_departments_with_pagination(test_db, sample_department):
    response = client.get("/api/v1/departments/?page=1&page_size=10")
    assert response.status_code == 200
    data = response.json()
    assert "items" in data
    assert data["page"] == 1
```

## 12. When to Use Copilot

### Ideal Use Cases
- **DO** use Copilot for: Repository CRUD implementations
- **DO** use Copilot for: Validator implementations
- **DO** use Copilot for: Test case generation
- **DO** use Copilot for: Endpoint boilerplate

### Review Required
- **ALWAYS** review Copilot-generated code before committing
- **ALWAYS** verify error handling (404, 409, etc.)
- **ALWAYS** check soft-delete logic
- **ALWAYS** ensure validators are present

### Copilot Limitations
- May not follow soft-delete patterns correctly → Review and fix
- May miss validation → Add custom validators
- May use raw SQL → Replace with SQLAlchemy ORM
- May put logic in wrong layer → Refactor to correct layer
