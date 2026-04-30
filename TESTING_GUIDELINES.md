# Testing Guidelines & Pytest Standards

## 1. Testing Overview

### Test Scope
- **Unit Tests:** Individual functions, methods, validators
- **Integration Tests:** Service + Repository interactions
- **API Tests:** Full endpoint testing with actual requests
- **Future:** E2E tests for critical workflows

### Test Coverage Target
- **Minimum:** 70% code coverage
- **Target:** 85%+ coverage for critical paths
- **Tools:** Pytest with pytest-cov

## 2. Pytest Project Structure

### File Organization
```
employee-management-api/
├── tests/
│   ├── conftest.py                    # Fixtures and configuration
│   ├── unit/
│   │   ├── test_validators.py
│   │   ├── test_pagination.py
│   │   └── test_sorting.py
│   ├── integration/
│   │   ├── test_employee_service.py
│   │   ├── test_department_service.py
│   │   ├── test_employee_repo.py
│   │   └── test_department_repo.py
│   └── api/
│       ├── test_employees_routes.py
│       ├── test_departments_routes.py
│       └── test_health.py
└── requirements.txt
```

### Test File Naming
- **Pattern:** `test_*.py` or `*_test.py`
- **Corresponding file:** `test_employee_service.py` for `employee_service.py`
- **One test class per main class**

## 3. Pytest Fixtures

### Shared Fixtures (conftest.py)
```python
# filepath: tests/conftest.py
import pytest
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from app.db import Base
from app.models.employee import Employee
from app.models.department import Department

@pytest.fixture(scope="function")
def test_db():
    """Create in-memory SQLite database for testing."""
    engine = create_engine("sqlite:///:memory:")
    Base.metadata.create_all(bind=engine)
    TestingSessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
    db = TestingSessionLocal()
    yield db
    db.close()

@pytest.fixture
def sample_department(test_db):
    """Create a sample department."""
    dept = Department(
        code="ENG",
        name="Engineering",
        description="Engineering Department",
        is_active=True
    )
    test_db.add(dept)
    test_db.commit()
    test_db.refresh(dept)
    return dept

@pytest.fixture
def sample_employee(test_db, sample_department):
    """Create a sample employee."""
    emp = Employee(
        employee_code="EMP-001",
        first_name="John",
        last_name="Doe",
        email="john@email.com",
        gender="Male",
        department_id=sample_department.id,
        salary=50000,
        joining_date="2024-01-15"
    )
    test_db.add(emp)
    test_db.commit()
    test_db.refresh(emp)
    return emp
```

### Fixture Naming & Scope
- **`function` scope (default):** Fresh fixture for each test
- **`module` scope:** Shared across tests in a module
- **`session` scope:** Shared across entire test session
- **Naming:** `sample_{resource}`, `test_{resource}`, `mock_{dependency}`

## 4. Test Patterns & Examples

### Unit Test: Validator
```python
# filepath: tests/unit/test_validators.py
import pytest
from datetime import date, timedelta
from pydantic import ValidationError
from app.schemas.employee import EmployeeCreate

def test_salary_cannot_be_negative():
    """Test that salary validator rejects negative values."""
    with pytest.raises(ValidationError) as exc_info:
        EmployeeCreate(
            first_name="John",
            last_name="Doe",
            email="john@email.com",
            gender="Male",
            salary=-1000,  # Negative salary
            department_id=1,
            joining_date=date.today()
        )
    assert "Salary cannot be negative" in str(exc_info.value)

def test_joining_date_too_far_in_future():
    """Test that joining_date validator rejects dates beyond 30 days."""
    future_date = date.today() + timedelta(days=31)
    with pytest.raises(ValidationError) as exc_info:
        EmployeeCreate(
            first_name="John",
            last_name="Doe",
            email="john@email.com",
            gender="Male",
            salary=50000,
            department_id=1,
            joining_date=future_date
        )
    assert "Joining date cannot be beyond 30 days" in str(exc_info.value)

def test_valid_employee_creation():
    """Test that valid employee data creates successfully."""
    emp = EmployeeCreate(
        first_name="John",
        last_name="Doe",
        email="john@email.com",
        gender="Male",
        salary=50000,
        department_id=1,
        joining_date=date.today()
    )
    assert emp.first_name == "John"
    assert emp.employee_code is not None  # Auto-generated
```

### Integration Test: Service
```python
# filepath: tests/integration/test_employee_service.py
import pytest
from fastapi import HTTPException
from app.services.employee_service import EmployeeService
from datetime import date

@pytest.fixture
def employee_service():
    return EmployeeService()

def test_create_employee_success(employee_service, test_db, sample_department):
    """Test successful employee creation."""
    data = {
        "first_name": "Jane",
        "last_name": "Smith",
        "email": "jane@email.com",
        "gender": "Female",
        "salary": 60000,
        "department_id": sample_department.id,
        "joining_date": date.today()
    }
    emp = employee_service.create_employee(test_db, data)
    assert emp.id is not None
    assert emp.first_name == "Jane"
    assert emp.employee_code is not None

def test_get_employee_not_found(employee_service, test_db):
    """Test that getting non-existent employee raises 404."""
    with pytest.raises(HTTPException) as exc_info:
        employee_service.get_employee(test_db, 999)
    assert exc_info.value.status_code == 404
    assert "Employee not found" in exc_info.value.detail

def test_soft_delete_employee(employee_service, test_db, sample_employee):
    """Test that employee soft delete sets deleted_at."""
    emp_id = sample_employee.id
    employee_service.delete_employee(test_db, emp_id)
    
    # Verify soft delete marker is set
    from app.models.employee import Employee
    deleted_emp = test_db.query(Employee).filter(Employee.id == emp_id).first()
    assert deleted_emp.deleted_at is not None
    
    # Verify get_employee raises 404 (soft-deleted)
    with pytest.raises(HTTPException):
        employee_service.get_employee(test_db, emp_id)
```

### API Test: Endpoint
```python
# filepath: tests/api/test_employees_routes.py
import pytest
from fastapi.testclient import TestClient
from app.main import app
from datetime import date

client = TestClient(app)

def test_create_employee_endpoint(test_db, sample_department, monkeypatch):
    """Test POST /api/v1/employees endpoint."""
    # Mock database dependency
    def override_get_db():
        return test_db
    
    from app.routers.employees import get_db
    app.dependency_overrides[get_db] = override_get_db
    
    payload = {
        "first_name": "Bob",
        "last_name": "Johnson",
        "email": "bob@email.com",
        "gender": "Male",
        "salary": 55000,
        "department_id": sample_department.id,
        "joining_date": str(date.today())
    }
    response = client.post("/api/v1/employees/", json=payload)
    assert response.status_code == 200
    data = response.json()
    assert data["first_name"] == "Bob"
    assert data["id"] is not None

def test_list_employees_with_pagination(test_db, sample_employee):
    """Test GET /api/v1/employees with pagination."""
    def override_get_db():
        return test_db
    
    from app.routers.employees import get_db
    app.dependency_overrides[get_db] = override_get_db
    
    response = client.get("/api/v1/employees/?page=1&page_size=10")
    assert response.status_code == 200
    data = response.json()
    assert "total" in data
    assert "page" in data
    assert "items" in data
    assert data["page"] == 1

def test_get_employee_by_id(test_db, sample_employee):
    """Test GET /api/v1/employees/{id} endpoint."""
    def override_get_db():
        return test_db
    
    from app.routers.employees import get_db
    app.dependency_overrides[get_db] = override_get_db
    
    response = client.get(f"/api/v1/employees/{sample_employee.id}")
    assert response.status_code == 200
    data = response.json()
    assert data["id"] == sample_employee.id
    assert data["first_name"] == sample_employee.first_name

def test_delete_employee_soft_deletes(test_db, sample_employee):
    """Test DELETE /api/v1/employees/{id} performs soft delete."""
    def override_get_db():
        return test_db
    
    from app.routers.employees import get_db
    app.dependency_overrides[get_db] = override_get_db
    
    emp_id = sample_employee.id
    response = client.delete(f"/api/v1/employees/{emp_id}")
    assert response.status_code == 204
    
    # Verify soft delete: should get 404 when fetching
    response = client.get(f"/api/v1/employees/{emp_id}")
    assert response.status_code == 404
```

## 5. Test Naming Standards

### Test Function Names
- **Pattern:** `test_{action}_{scenario}` or `test_{method}_{condition}_{expected}`
- **Examples:**
  - `test_create_employee_success`
  - `test_get_employee_not_found`
  - `test_salary_cannot_be_negative`
  - `test_list_employees_with_pagination`

### Assertion Naming
- Use descriptive assertion messages
- Example:
  ```python
  assert response.status_code == 404, "Expected 404 for non-existent employee"
  assert emp.deleted_at is not None, "Expected soft delete timestamp to be set"
  ```

## 6. Assertion Best Practices

```python
# ✓ GOOD: Clear, specific assertions
assert emp.salary == 50000, "Salary should match input"
assert emp.deleted_at is not None, "Soft delete timestamp should be set"
assert len(response.json()["items"]) <= 20, "Page size should not exceed 20"

# ✗ BAD: Vague assertions
assert emp is not None
assert response.json()  # Just checking response exists
```

## 7. Running Tests

### Command Line
```bash
# Run all tests
pytest

# Run specific test file
pytest tests/unit/test_validators.py

# Run with coverage
pytest --cov=app --cov-report=html

# Run with verbose output
pytest -v

# Run tests matching pattern
pytest -k "test_create"
```

### Configuration (pytest.ini)
```ini
[pytest]
testpaths = tests
python_files = test_*.py
python_classes = Test*
python_functions = test_*
addopts = -v --strict-markers
```

## 8. Test Isolation & Cleanup

### Database Isolation
```python
@pytest.fixture(scope="function")
def test_db():
    """Each test gets a fresh in-memory database."""
    engine = create_engine("sqlite:///:memory:")
    Base.metadata.create_all(bind=engine)
    TestingSessionLocal = sessionmaker(bind=engine)
    db = TestingSessionLocal()
    yield db
    db.close()
```

### Dependency Overrides
```python
def test_endpoint():
    from app.routers.employees import get_db
    
    def override_get_db():
        return test_db
    
    app.dependency_overrides[get_db] = override_get_db
    # Test code here
    app.dependency_overrides.clear()  # Cleanup
```
