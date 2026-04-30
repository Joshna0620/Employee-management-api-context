# Coding Standards & Conventions

## 1. Naming Conventions

### Files & Directories
- **Lowercase with underscores:** `employee_repo.py`, `department_service.py`
- **No abbreviations:** Use `employee` not `emp`, `department` not `dept`
- **Descriptive names:** `employee_service.py` not `service.py`

### Classes
- **PascalCase:** `EmployeeRepository`, `DepartmentService`, `EmployeeOut`
- **Descriptive names:** `EmployeeRepository` (not `EmpRepo`)
- **Suffix patterns:**
  - `*Repository` for data access classes
  - `*Service` for business logic classes
  - `*Out` for response schemas
  - `*Create`/`*Update` for request schemas

### Functions & Methods
- **snake_case:** `create_employee()`, `get_department()`, `soft_delete()`
- **Action verbs:** `create_`, `get_`, `update_`, `delete_`, `list_`
- **Query methods:** `get_all()` returns SQLAlchemy query, `get()` returns single object

### Variables
- **snake_case:** `employee_id`, `department_code`, `page_size`
- **Meaningful names:** `emp_id` not acceptable, use `employee_id`
- **Boolean prefix:** `is_active`, `is_full_time`, `can_delete`
- **Database models:** Use singular form in model class names: `Employee`, `Department`

### Constants
- **UPPER_SNAKE_CASE:** `DATABASE_URL`, `DEFAULT_PAGE_SIZE`
- **Location:** Define in config files or module top-level
- **Example:**
  ```python
  # In config or settings
  DEFAULT_PAGE_SIZE = 20
  MAX_PAGE_SIZE = 100
  ```

## 2. Code Structure & Organization

### Imports
```python
# Order: Standard library → Third-party → Local imports
from datetime import datetime, date
from typing import Optional

from sqlalchemy.orm import Session
from fastapi import HTTPException

from app.models.employee import Employee
from app.repositories.employee_repo import EmployeeRepository
```

### Class Structure
```python
class EmployeeService:
    """Docstring describing the service."""
    
    def __init__(self):
        self.repo = EmployeeRepository()
    
    def create_employee(self, db: Session, data):
        """Docstring for method."""
        pass
    
    def get_employee(self, db: Session, emp_id: int):
        """Docstring for method."""
        pass
```

### Method Ordering in Classes
1. `__init__()` (if present)
2. Public methods (in logical order: create, read, update, delete)
3. Private methods (prefixed with `_`)

## 3. Error Handling

### HTTP Exceptions
```python
from fastapi import HTTPException

# Use appropriate HTTP status codes
if not employee:
    raise HTTPException(status_code=404, detail="Employee not found")

if db.query(Employee).filter(Employee.department_id == dept_id).count() > 0:
    raise HTTPException(status_code=409, detail="Cannot delete department with active employees")

if invalid_data:
    raise HTTPException(status_code=400, detail="Invalid input data")
```

### Status Codes
- `200` – OK (successful GET, PUT)
- `201` – Created (successful POST) - not explicitly set in FastAPI for create endpoints
- `204` – No Content (successful DELETE)
- `400` – Bad Request (validation errors)
- `404` – Not Found (resource doesn't exist)
- `409` – Conflict (business rule violation)
- `500` – Internal Server Error (unexpected failures)

### No Silent Failures
```python
# ✗ BAD: Silently returns None
def get_employee(self, db: Session, emp_id: int):
    return db.query(Employee).filter(Employee.id == emp_id).first()

# ✓ GOOD: Raises exception if not found
def get_employee(self, db: Session, emp_id: int):
    emp = repo.get(db, emp_id)
    if not emp:
        raise HTTPException(status_code=404, detail="Employee not found")
    return emp
```

## 4. Logging Standards (Future Implementation)

When logging is added, follow these conventions:

```python
import logging

logger = logging.getLogger(__name__)

# Log levels:
# DEBUG: Detailed diagnostic info (variable values, flow)
# INFO: General info (operation start/end)
# WARNING: Warning messages (recoverable issues)
# ERROR: Error messages (operation failed)

logger.info(f"Creating employee: {employee_code}")
logger.error(f"Failed to create employee: {str(e)}")
```

## 5. Database Usage Standards

### Query Filtering
```python
# ✓ GOOD: Filter in repository, return clean queries
def get_all(self, db: Session):
    return db.query(Employee).filter(Employee.deleted_at.is_(None))

# Usage in router:
query = service.repo.get_all(db)  # Already filtered
```

### Soft Delete Pattern
```python
from datetime import datetime

# Soft delete: Set deleted_at timestamp
def soft_delete(self, db: Session, emp):
    emp.deleted_at = datetime.utcnow()
    db.commit()

# Query active records only
query = db.query(Employee).filter(Employee.deleted_at.is_(None))
```

### Transaction Management
```python
# ✓ GOOD: Explicit commit/rollback
def create(self, db: Session, data):
    obj = Employee(**data)
    db.add(obj)
    db.commit()  # Explicit commit
    db.refresh(obj)
    return obj

# ✓ GOOD: Session cleanup in dependency injection
def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()  # Cleanup
```

## 6. Validation Standards

### Field-Level Validators
```python
from pydantic import validator, EmailStr

class EmployeeCreate(BaseModel):
    salary: float
    email: EmailStr
    
    @validator("salary")
    def salary_non_negative(cls, v):
        if v < 0:
            raise ValueError("Salary cannot be negative")
        return v
```

### Root-Level Validators
```python
from pydantic import root_validator

@root_validator(pre=True)
def generate_employee_code(cls, values):
    if not values.get("employee_code"):
        values["employee_code"] = f"EMP-{int(datetime.utcnow().timestamp())}"
    return values
```

### Validator Placement
- Field-level validators on individual fields
- Root-level validators for cross-field validation or auto-generation
- Use `pre=True` for value transformation before validation

## 7. Type Hints
```python
# ✓ GOOD: Always use type hints
def create_employee(self, db: Session, data: dict) -> Employee:
    pass

def list_employees(self, page: int = 1, page_size: int = 20) -> dict:
    pass

# ✓ GOOD: Optional types
from typing import Optional
manager_id: Optional[int] = None
```

## 8. Docstrings (Future Standard)
```python
def create_employee(self, db: Session, data: dict) -> Employee:
    """
    Create a new employee record.
    
    Args:
        db: Database session
        data: Employee data dictionary
        
    Returns:
        Employee: Created employee object
        
    Raises:
        ValueError: If validation fails
    """
    pass
```

## 9. Code Style
- **Line Length:** Max 100 characters
- **Indentation:** 4 spaces (no tabs)
- **Spacing:** 2 blank lines between class definitions, 1 between methods
- **Comments:** Use `#` for inline comments, docstrings for functions/classes
