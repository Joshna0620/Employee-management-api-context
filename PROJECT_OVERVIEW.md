# Employee Management API - Project Overview

## 1. Application Purpose
The Employee Management API is a RESTful backend service that manages employee and department information for organizations. It provides endpoints to create, read, update, and soft-delete employee and department records with support for hierarchical relationships (managers, departments).

## 2. Tech Stack
- **Language:** Python 3.x
- **Framework:** FastAPI
- **ORM:** SQLAlchemy
- **Database:** SQLite (development), configurable via `DATABASE_URL`
- **Testing:** Pytest
- **Validation:** Pydantic with EmailStr validator
- **Server:** Uvicorn
- **Environment Management:** python-dotenv

## 3. Project Scope

### In Scope
- Employee CRUD operations with soft delete
- Department CRUD operations
- Pagination and sorting on list endpoints
- Relationship management (employee-department, manager-employee)
- Input validation and error handling
- SQLAlchemy ORM-based data persistence
- Enum support for Employment Status and Gender

### Out of Scope
- Authentication & Authorization
- API rate limiting
- Caching strategies
- Email notifications
- Advanced reporting & analytics
- Multi-tenancy support

## 4. Folder Structure
```
employee-management-api/
├── app/
│   ├── main.py                 # FastAPI application entry point
│   ├── db.py                   # Database configuration
│   ├── models/                 # SQLAlchemy models
│   │   ├── employee.py
│   │   └── department.py
│   ├── schemas/                # Pydantic request/response models
│   │   ├── employee.py
│   │   └── department.py
│   ├── repositories/           # Data access layer
│   │   ├── employee_repo.py
│   │   └── department_repo.py
│   ├── services/               # Business logic layer
│   │   ├── employee_service.py
│   │   └── department_service.py
│   ├── routers/                # API route definitions
│   │   ├── employees.py
│   │   └── departments.py
│   └── utils/                  # Utility functions
│       ├── pagination.py
│       └── sorting.py
├── requirements.txt
├── .env.example
└── PROJECT_OVERVIEW.md         # This file

## 5. Running the Application
```bash
# Install dependencies
pip install -r requirements.txt

# Set up environment
cp .env.example .env

# Run the server
uvicorn app.main:app --reload
```

## 6. API Base URL
- Local: `http://localhost:8000`
- Health Check: `GET /health`
- API Version: v1
- Base Path: `/api/v1`

## 7. Key Features
- Soft delete for employees (maintains data integrity)
- Hierarchical employee relationships (manager-subordinate)
- Department-employee association
- Pagination with configurable page size (1-100)
- Sorting by multiple fields (ascending/descending)
- Input validation with custom validators
- Enum-based Employment Status and Gender
