# Technical Documentation - Accipere ATS Backend

## Overview

The Accipere ATS backend is a Django REST Framework application that provides a comprehensive API for managing recruitment workflows. The system follows a layered architecture with clear separation between models, serializers, views, and business logic. It uses PostgreSQL for data persistence, JWT for authentication, and stores resume files as binary data directly in the database.

### Key Design Principles

- **RESTful API Design**: All endpoints follow REST conventions with proper HTTP methods and status codes
- **Role-Based Access Control**: Three-tier permission system (Administrator, Recruiter, Technical Evaluator)
- **Stateless Authentication**: JWT tokens for scalable, stateless authentication
- **Database-First Storage**: Resume files stored in PostgreSQL for simplified architecture and ACID compliance
- **Django Best Practices**: Leverage Django ORM, built-in authentication, and DRF conventions

---

## 1. Technology Stack

### Core Framework
- **Django** 4.2+ (LTS)
- **Django REST Framework** 3.14+
- **Python** 3.10+

### Database
- **PostgreSQL** 14+ (Development & Production)

### Authentication & Security
- **djangorestframework-simplejwt** - JWT authentication
- **django-cors-headers** - CORS handling

### Additional Libraries
- **drf-spectacular** - OpenAPI/Swagger documentation
- **python-decouple** - Configuration management
- **gunicorn** - WSGI HTTP Server (Production)
- **python-magic** - File type validation
- **django-filter** - Advanced filtering for querysets
- **django-ratelimit** - Rate limiting for API endpoints

---

## 1.1 Architecture

### High-Level Architecture

```
┌─────────────────┐
│ React Frontend  │
└────────┬────────┘
         │ HTTP/HTTPS
         ▼
┌─────────────────────────────┐
│ Django REST Framework API   │
├─────────────────────────────┤
│ Authentication Middleware   │
│ (JWT Token Validation)      │
├─────────────────────────────┤
│ Permission Layer            │
│ (Role-Based Access Control) │
├─────────────────────────────┤
│ ViewSets & Views            │
├─────────────────────────────┤
│ Serializers                 │
├─────────────────────────────┤
│ Django Models               │
└────────┬────────────────────┘
         │
         ▼
┌─────────────────┐
│   PostgreSQL    │
│    Database     │
└─────────────────┘
```

### Application Structure

```
accipere-backend/
├── manage.py
├── requirements.txt
├── .env.example
├── accipere/                    # Project configuration
│   ├── settings.py              # Django settings with environment-based config
│   ├── urls.py                  # Root URL configuration
│   └── wsgi.py                  # WSGI application entry point
├── authentication/              # JWT authentication & user registration
│   ├── models.py
│   ├── serializers.py
│   ├── views.py
│   └── urls.py
├── users/                       # User management
│   ├── models.py                # User model with role_id FK
│   ├── serializers.py
│   ├── views.py
│   └── urls.py
├── roles/                       # Role management
│   ├── models.py                # Simple ROLES table
│   ├── serializers.py
│   ├── views.py
│   └── urls.py
├── jobs/                        # Job posting management
│   ├── models.py
│   ├── serializers.py
│   ├── views.py
│   ├── urls.py
│   └── filters.py
├── applications/                # Application tracking & status management
│   ├── models.py
│   ├── serializers.py
│   ├── views.py
│   └── urls.py
├── common/                      # Shared utilities, permissions, validators
│   ├── permissions.py
│   ├── validators.py
│   └── exceptions.py
└── tests/                       # Test suite
    ├── test_auth.py
    ├── test_users.py
    ├── test_jobs.py
    └── test_applications.py
```

---

## 2. Complete API Endpoints

### 2.1 Authentication APIs

```
POST   /api/auth/login/             - Login (returns JWT tokens)
POST   /api/auth/logout/            - Logout (blacklist token)
POST   /api/auth/refresh/           - Refresh access token
GET    /api/auth/me/                - Get current user profile
```

### 2.2 User Management APIs

```
GET    /api/users/                  - List all users (Administrator only)
POST   /api/users/                  - Create user (Administrator only)
PUT    /api/users/{id}/             - Update user (Administrator only)
DELETE /api/users/{id}/             - Soft delete user (Administrator only)
GET    /api/users/{id}/applications/ - Get applications assigned to user
```

### 2.3 Role Management APIs

```
GET    /api/roles/                  - List all roles (Administrator only)
POST   /api/roles/                  - Create role (Administrator only)
PUT    /api/roles/{id}/             - Update role (Administrator only)
DELETE /api/roles/{id}/             - Soft delete role (Administrator only)
```

### 2.4 Job Management APIs

```
GET    /api/jobs/                   - List all jobs
POST   /api/jobs/                   - Create new job
GET    /api/jobs/{id}/              - Get job details
PUT    /api/jobs/{id}/              - Update job
PATCH  /api/jobs/{id}/              - Partial update job
DELETE /api/jobs/{id}/              - Delete job
GET    /api/jobs/{id}/applicants/   - Get all applicants for a job
```

### 2.5 Application Management APIs

```
GET    /api/applications/           - List all applications
POST   /api/applications/           - Create application (public endpoint)
GET    /api/applications/{id}/      - Get application details
PUT    /api/applications/{id}/      - Update application (full update)
PATCH  /api/applications/{id}/      - Partial update (status, assignment, notes)
DELETE /api/applications/{id}/      - Delete application
GET    /api/applications/{id}/resume/ - Download applicant's resume
```

### 2.6 Application Status APIs

```
GET    /api/application-statuses/   - List all application statuses
POST   /api/application-statuses/   - Create new status (admin only)
PUT    /api/application-statuses/{id}/ - Update status (admin only)
PATCH  /api/application-statuses/{id}/ - Partial update status
DELETE /api/application-statuses/{id}/ - Delete status (admin only)
```

---

## 3. Authentication & Permissions

### 3.1 JWT Authentication

- Access token expiry: 1 hour
- Refresh token expiry: 7 days
- Token blacklisting on logout

### 3.2 Role-Based Access Control

**Simple Role System:**
The system uses a straightforward role-based approach with three predefined roles stored in the ROLES table:

- **Administrator** - Full access to all resources (users, jobs, applications, statuses)
- **Recruiter** - Can manage jobs, view/update applications, assign evaluators
- **Technical Evaluator** - Can view applications and add technical notes

**Permission Implementation:**
- Each user has a single role assigned via `role_id` foreign key
- Permissions are checked based on the user's role name
- Custom permission decorators validate role access at the view level

### 3.3 Permission Matrix

| Endpoint | Administrator | Recruiter | Technical Evaluator | Public |
|----------|---------------|-----------|---------------------|--------|
| POST /api/applications/ | ✓ | ✓ | ✓ | ✓ |
| GET /api/jobs/ | ✓ | ✓ | ✓ | ✓ |
| POST /api/jobs/ | ✓ | ✓ | ✗ | ✗ |
| GET /api/applications/ | ✓ | ✓ | ✓ | ✗ |
| PUT /api/applications/{id}/ | ✓ | ✓ | ✓ (notes only) | ✗ |
| POST /api/users/ | ✓ | ✗ | ✗ | ✗ |
| POST /api/application-statuses/ | ✓ | ✗ | ✗ | ✗ |
| DELETE /api/applications/{id}/ | ✓ | ✓ | ✗ | ✗ |
| GET /api/roles/ | ✓ | ✗ | ✗ | ✗ |

---

---

## 4. Status Management

### 4.1 Application Status Workflow

The system uses a flexible status management system that allows administrators to:

1. **Create custom statuses** - Add new stages to the recruitment pipeline
2. **Define order** - Set the sequence of statuses
3. **Activate/Deactivate** - Toggle statuses without deleting them
4. **Track transitions** - Monitor status changes over time

### 4.2 Status Transition Rules

- Applications start with "HR Screening" status by default
- Status can be updated to any available active status
- Status changes are logged with timestamp and user
- Rejected applications cannot be moved to other statuses (optional business rule)
- Joined applications are considered final

### 4.3 Status Management Best Practices

1. **Keep statuses sequential** - Use order_sequence to maintain logical flow
2. **Use descriptive names** - Clear status names help team communication
3. **Don't delete statuses** - Deactivate instead to preserve historical data
4. **Document transitions** - Add notes when changing application status
5. **Review regularly** - Audit status usage and optimize pipeline

---

---

## 5. Soft Delete Implementation

### 5.1 Overview

The system implements soft delete functionality across all major tables to preserve data integrity and enable data recovery. Instead of permanently removing records from the database, records are marked as deleted using a `deleted_at` timestamp field.

### 5.2 Tables with Soft Delete

The following tables include the `deleted_at` field:

- **AUTH_USERS** - User accounts
- **JOBS** - Job postings
- **APPLICATIONS** - Job applications
- **APPLICATION_STATUSES** - Status definitions
- **APPLICATION_ASSIGNED_USER_STATUSES** - Status tracking records

### 5.3 Soft Delete Behavior

**Field Specification:**
```
deleted_at: DateTimeField, nullable=True, default=None
```

**Delete Operation:**
- When a DELETE request is made, set `deleted_at = current_timestamp`
- Record remains in database but is excluded from normal queries
- Original data is preserved for audit trails and recovery

**Query Filtering:**
- Default queryset filters: `deleted_at__isnull=True`
- Soft-deleted records are hidden from standard API responses
- Admin users can optionally view deleted records with special filters

### 5.4 Implementation Guidelines

**Model Level:**
```python
class SoftDeleteModel(models.Model):
    deleted_at = models.DateTimeField(null=True, blank=True, default=None)
    
    class Meta:
        abstract = True
    
    def soft_delete(self):
        self.deleted_at = timezone.now()
        self.save()
    
    def restore(self):
        self.deleted_at = None
        self.save()
```

**Manager Level:**
```python
class SoftDeleteManager(models.Manager):
    def get_queryset(self):
        return super().get_queryset().filter(deleted_at__isnull=True)
    
    def with_deleted(self):
        return super().get_queryset()
    
    def deleted_only(self):
        return super().get_queryset().filter(deleted_at__isnull=False)
```

**ViewSet Level:**
```python
def destroy(self, request, *args, **kwargs):
    instance = self.get_object()
    instance.soft_delete()
    return Response(status=status.HTTP_204_NO_CONTENT)
```

### 5.5 Recovery Endpoints

**Restore Deleted Record:**
```
POST /api/{resource}/{id}/restore/
```

**List Deleted Records (Admin only):**
```
GET /api/{resource}/?show_deleted=true
```

### 5.6 Benefits

- **Data Recovery**: Accidentally deleted records can be restored
- **Audit Trail**: Maintain complete history of all records
- **Compliance**: Meet regulatory requirements for data retention
- **Referential Integrity**: Avoid cascade deletion issues
- **Analytics**: Include historical data in reports

### 5.7 Considerations

- Unique constraints must account for soft-deleted records
- Database indexes should consider `deleted_at` field
- Periodic cleanup jobs may be needed for truly removing old data
- Backup strategies should account for soft-deleted records

---

## 6. File Storage & Resume Handling

### 6.1 Resume Storage in PostgreSQL

- **Storage Method:** BinaryField in APPLICATION model
- **Maximum file size:** 5MB
- **Allowed formats:** PDF, DOC, DOCX
- **Validation:** Magic byte checking (not just extension)

### 6.2 Resume Upload/Download

**Upload Process:**
- Multipart form data
- File type validation using magic bytes (python-magic)
- Size limit enforcement (5MB maximum)
- Store filename, content type, and file size

**File Validation:**
```
Maximum Size: 5MB (5,242,880 bytes)
Allowed Types: PDF, DOC, DOCX
Validation Method: Magic byte checking (not just extension)
```

**Download Process:**
- Dedicated endpoint: `GET /api/applications/{id}/resume/`
- Proper Content-Type header
- Content-Disposition: attachment with original filename
- Streaming for large files (>1MB)

**Resume Download Response Headers:**
```
Content-Type: application/pdf (or appropriate MIME type)
Content-Disposition: attachment; filename="john_doe_resume.pdf"
Content-Length: 524288
```

---

## 7. Error Handling

### Error Response Format

All API errors follow a consistent format:

```json
{
  "error": "Error message",
  "details": {
    "field_name": ["Specific error for this field"]
  },
  "code": "ERROR_CODE"
}
```

### HTTP Status Codes

- **200 OK**: Successful GET, PUT, PATCH requests
- **201 Created**: Successful POST requests
- **204 No Content**: Successful DELETE requests
- **400 Bad Request**: Validation errors, duplicate applications
- **401 Unauthorized**: Missing or invalid JWT token
- **403 Forbidden**: Insufficient permissions
- **404 Not Found**: Resource not found
- **500 Internal Server Error**: Unexpected server errors

### Common Error Examples

**Duplicate Application:**
```json
{
  "error": "You have already applied to this job",
  "code": "DUPLICATE_APPLICATION"
}
```

**Invalid Resume File:**
```json
{
  "error": "Resume file size must not exceed 5MB",
  "code": "FILE_SIZE_EXCEEDED"
}
```

**Missing Required Fields:**
```json
{
  "error": "Validation failed",
  "details": {
    "email": ["This field is required"],
    "first_name": ["This field is required"]
  },
  "code": "VALIDATION_ERROR"
}
```

**Unauthorized Access:**
```json
{
  "error": "Authentication credentials were not provided",
  "code": "NOT_AUTHENTICATED"
}
```

**Forbidden Access:**
```json
{
  "error": "You do not have permission to perform this action",
  "code": "PERMISSION_DENIED"
}
```

---

## 8. Security Requirements

### 8.1 Data Protection
- Password hashing using Django's PBKDF2
- HTTPS only in production
- CSRF protection
- SQL injection prevention (ORM)

### 8.2 Input Validation
- Sanitize all user inputs
- File upload validation
- Email format validation
- XSS protection

### 8.3 API Security
- JWT token validation
- CORS whitelist configuration
- Rate limiting (recommended)
- Request size limits

---

---

## 9. Testing Requirements

### 9.1 Test Coverage
- Minimum 80% code coverage
- Use `pytest` and `pytest-django`
- Mock external services

### 9.2 Test Types
- Unit tests for models, serializers, views
- Integration tests for API endpoints
- Permission tests
- File upload/download tests
- Status transition tests

---

## 10. Future Enhancements

### 10.1 Async Task Processing
- Celery + Redis for background jobs
- Email notifications
- Resume parsing
- Scheduled reports

### 10.2 Advanced Features
- Calendar integration
- Interview scheduling
- Video interview integration
- AI-powered candidate matching
- Resume parsing and auto-fill
- Analytics dashboard
- Export to PDF/Excel


