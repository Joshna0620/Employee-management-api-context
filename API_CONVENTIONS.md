# API Conventions & Standards

## 1. Endpoint Patterns

### Resource-Based URLs
- **Format:** `/api/v1/{resource}` and `/api/v1/{resource}/{id}`
- **HTTP Methods:** GET (retrieve), POST (create), PUT (update), DELETE (delete)

### Employee Endpoints
```
POST   /api/v1/employees              Create employee
GET    /api/v1/employees              List employees (with pagination, filtering, sorting)
GET    /api/v1/employees/{id}         Get single employee
PUT    /api/v1/employees/{id}         Update employee
DELETE /api/v1/employees/{id}         Delete employee (soft delete)
```

### Department Endpoints
```
POST   /api/v1/departments            Create department
GET    /api/v1/departments            List departments (with pagination)
GET    /api/v1/departments/{id}       Get single department
PUT    /api/v1/departments/{id}       Update department
DELETE /api/v1/departments/{id}       Delete department (hard delete with validation)
```

## 2. HTTP Methods & Status Codes

### POST (Create)
- **Request:** Send data in request body
- **Success Response:** `200 OK` with created resource
- **Error Response:** `400 Bad Request` (validation error), `409 Conflict`
- **Example:**
  ```
  POST /api/v1/employees
  Content-Type: application/json
  
  {
    "first_name": "John",
    "last_name": "Doe",
    "email": "john@email.com",
    "gender": "Male",
    "salary": 50000,
    "department_id": 1,
    "joining_date": "2024-01-15"
  }
  
  Response: 200 OK
  {
    "id": 1,
    "employee_code": "EMP-1714243200",
    "first_name": "John",
    ...
  }
  ```

### GET (Retrieve)
- **Request:** No body, parameters in query string
- **Success Response:** `200 OK` with resource(s)
- **Error Response:** `404 Not Found` (for single resource)
- **Example:**
  ```
  GET /api/v1/employees/1
  
  Response: 200 OK
  {
    "id": 1,
    "employee_code": "EMP-1714243200",
    ...
  }
  ```

### PUT (Update)
- **Request:** Send partial or full data in request body
- **Success Response:** `200 OK` with updated resource
- **Error Response:** `404 Not Found`, `400 Bad Request`
- **Example:**
  ```
  PUT /api/v1/employees/1
  Content-Type: application/json
  
  {
    "salary": 55000,
    "job_title": "Senior Developer"
  }
  
  Response: 200 OK
  {
    "id": 1,
    ...
    "salary": 55000,
    "job_title": "Senior Developer"
  }
  ```

### DELETE (Delete/Soft Delete)
- **Request:** No body, resource ID in URL
- **Success Response:** `204 No Content` (empty response)
- **Error Response:** `404 Not Found`, `409 Conflict`
- **Example:**
  ```
  DELETE /api/v1/employees/1
  
  Response: 204 No Content
  ```

## 3. Pagination Standards

### Query Parameters
```
GET /api/v1/employees?page=1&page_size=20
```

- **`page`:** Page number (starts at 1), default: 1
- **`page_size`:** Records per page, default: 20, max: 100

### Pagination Rules
```python
# If both missing: Use defaults (1, 20)
# If only page provided: page_size = page value
# If only page_size provided: page = page_size value
# Validate: page >= 1, 1 <= page_size <= 100
```

### Response Format
```json
{
  "total": 150,
  "page": 1,
  "page_size": 20,
  "items": [
    { "id": 1, "name": "John", ... },
    { "id": 2, "name": "Jane", ... }
  ]
}
```

### Implementation
```python
def paginate(query, page: int = 1, page_size: int = 20):
    page = max(page, 1)
    page_size = min(max(page_size, 1), 100)
    total = query.count()
    items = query.offset((page - 1) * page_size).limit(page_size).all()
    return {
        "total": total,
        "page": page,
        "page_size": page_size,
        "items": items,
    }
```

## 4. Filtering Standards

### Supported Filters

#### Employee List Filters
- **`department_id`:** Filter by department (query parameter)
  ```
  GET /api/v1/employees?department_id=5&page=1&page_size=20
  ```
- **Soft Delete:** Automatically excluded (only active employees shown)

#### Department List Filters
- None currently implemented (all non-deleted departments shown)

### Filter Implementation Pattern
```python
query = service.repo.get_all(db)  # Base query with soft-delete filter

# Apply optional filters
if department_id:
    query = query.filter(Employee.department_id == department_id)

# Apply pagination
return paginate(query, page, page_size)
```

## 5. Sorting Standards

### Sorting Syntax
```
GET /api/v1/employees?sort=salary:desc,first_name:asc
```

- **Format:** `field:direction[,field:direction]`
- **Direction:** `asc` (default) or `desc`
- **Multiple fields:** Comma-separated, evaluated left-to-right

### Supported Sort Fields

#### Employee Sort Fields
- `id`
- `employee_code`
- `first_name`
- `last_name`
- `email`
- `salary`
- `joining_date`
- `employment_status`
- `created_at`
- `updated_at`

#### Department Sort Fields
- `id`
- `code`
- `name`
- `created_at`
- `updated_at`

### Implementation
```python
def apply_sorting(query, model, sort: str):
    if not sort:
        return query

    for field in sort.split(','):
        parts = field.split(':')
        column = getattr(model, parts[0], None)
        if column is not None:
            if len(parts) > 1 and parts[1] == "desc":
                query = query.order_by(column.desc())
            else:
                query = query.order_by(column.asc())
    return query
```

## 6. Soft Delete Rules

### Employee Soft Delete
- **Operation:** `DELETE /api/v1/employees/{id}`
- **Action:** Sets `deleted_at` to current UTC timestamp
- **Impact:** Employee filtered out from all list queries
- **Relationship:** Can still exist in department (no cascade)
- **Recovery:** Possible by clearing `deleted_at` (manual DB operation for now)

### Department Hard Delete
- **Operation:** `DELETE /api/v1/departments/{id}`
- **Validation:** Cannot delete if active employees exist in department
- **Error:** `409 Conflict` with message "Cannot delete department with active employees"
- **Check:** Only count employees where `deleted_at IS NULL`

### Query Standard for Soft Delete
```python
# ✓ GOOD: Exclude soft-deleted records
query = db.query(Employee).filter(Employee.deleted_at.is_(None))

# ✗ BAD: Include soft-deleted records (unless explicitly needed)
query = db.query(Employee)
```

## 7. Request/Response Schemas

### Request Schemas (EmployeeCreate)
- Contains required fields for creation
- Optional fields for non-mandatory data
- Includes validators for business rules

### Response Schemas (EmployeeOut)
- Includes all fields plus `id`, `created_at`, `updated_at`, `deleted_at`
- Uses `from_attributes = True` for SQLAlchemy model conversion
- Excludes sensitive data (none currently, but future-proof)

### Update Schemas (EmployeeUpdate)
- All fields are optional (`Optional[T]`)
- Uses `exclude_none=True` when converting to dict to avoid overwriting with None

## 8. Error Responses

### Standard Error Format
```json
{
  "detail": "Employee not found"
}
```

### Common Error Scenarios

| Scenario | Status | Detail |
|----------|--------|--------|
| Resource not found | 404 | "{Resource} not found" |
| Validation failed | 400 | "[Field validation error message]" |
| Business rule violated | 409 | "Cannot delete department with active employees" |
| Invalid input | 400 | "Email must be from email.com or yahoo.com" |

## 9. Version Strategy

### API Version in URL
- Current version: `v1`
- Format: `/api/v1/{resource}`
- Future versions: `/api/v2/{resource}` (separate routers)

### Versioning Rules
- Breaking changes require new version
- Additive changes (new optional fields) allowed in same version
- Deprecation period before removing old versions
