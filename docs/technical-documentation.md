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
- **PyJWT** 2.8.0+ - JWT token handling
- **python-decouple** - Configuration management
- **gunicorn** - WSGI HTTP Server (Production)
- **python-magic** - File type validation
- **django-filter** - Advanced filtering for querysets
- **django-ratelimit** - Rate limiting for API endpoints
- **coverage** 7.11.1 - Code coverage testing

---



### Application Structure

The backend follows a modular Django app structure:

- **accipere/** - Project configuration (settings, URLs, WSGI)
- **authentication/** - JWT authentication and user login
- **users/** - User management with role-based access
- **roles/** - Role definitions (Administrator, Recruiter, Technical Evaluator)
- **jobs/** - Job posting management
- **applications/** - Application tracking and status management
- **common/** - Shared utilities, permissions, and validators
- **tests/** - Test suite for all modules

---

## 2. Complete API Endpoints

All endpoints use UUID for `{id}` parameters.

### 2.1 Authentication APIs

| Method | Path | Description | Roles |
|--------|------|-------------|-------|
| POST | `/api/auth/token/` | Login and get JWT access & refresh tokens | Public |
| POST | `/api/auth/token/refresh/` | Refresh access token using refresh token | Public |
| POST | `/api/auth/token/logout/` | Logout and blacklist refresh token | Authenticated |
| GET | `/api/auth/me/` | Get current authenticated user profile | Authenticated |

### 2.2 User Management APIs

| Method | Path | Description | Roles |
|--------|------|-------------|-------|
| GET | `/api/users/` | List all users (use ?simple=true for minimal info) | Administrator (or Authenticated if simple=true) |
| POST | `/api/users/` | Create new user and send password setup email | Administrator |
| GET | `/api/users/{id}/` | Get user details by UUID | Administrator or Self |
| PUT | `/api/users/{id}/` | Update user (full update) | Administrator or Self |
| PATCH | `/api/users/{id}/` | Partial update user | Administrator or Self |
| DELETE | `/api/users/{id}/` | Soft delete user (sets deleted_at and is_active=False) | Administrator |
| POST | `/api/users/{id}/reset-password/` | Reset user password and send reset email | Administrator |
| POST | `/api/users/set-password/` | Set password via setup/reset link (token required) | Public |
| GET | `/api/users/{id}/applications/` | Get applications assigned to user | Administrator or Self |

**Notes:**
- Administrators cannot delete, deactivate, or reset their own account
- Password setup/reset links expire in 2 days
- `?simple=true` returns minimal user info (id, email, name, role_name) for active users only

### 2.3 Role Management APIs

| Method | Path | Description | Roles |
|--------|------|-------------|-------|
| GET | `/api/roles/` | List all roles | Authenticated |
| GET | `/api/roles/{id}/` | Get role details by UUID | Authenticated |

**Note:** Roles are read-only. No create, update, or delete endpoints available.

### 2.4 Job Management APIs

| Method | Path | Description | Roles |
|--------|------|-------------|-------|
| GET | `/api/jobs/` | List jobs (unauthenticated: Open only, authenticated: all) | Public |
| POST | `/api/jobs/` | Create new job posting (auto-sets posted_by_user_id) | Administrator, Recruiter |
| GET | `/api/jobs/{id}/` | Get job details by UUID | Public |
| PUT | `/api/jobs/{id}/` | Update job (full update) | Administrator, Recruiter |
| PATCH | `/api/jobs/{id}/` | Partial update job | Administrator, Recruiter |
| DELETE | `/api/jobs/{id}/` | Soft delete job | Administrator |
| GET | `/api/jobs/{id}/applicants/` | Get all applicants for a specific job | Administrator, Recruiter |

**Job Status Choices:** `Open` (accepting applications), `Closed` (auto-sets closing_date)

### 2.5 Application Management APIs

| Method | Path | Description | Roles |
|--------|------|-------------|-------|
| GET | `/api/applications/` | List all applications | Authenticated |
| POST | `/api/applications/` | Submit application (multipart/form-data with resume) | Public |
| GET | `/api/applications/{id}/` | Get application details by UUID | Authenticated |
| PUT | `/api/applications/{id}/` | Update application (full update) | Authenticated |
| PATCH | `/api/applications/{id}/` | Partial update (status, notes, assignment) | Authenticated |
| DELETE | `/api/applications/{id}/` | Soft delete application | Authenticated |

**Application Behavior:**
- On creation: Auto-assigns first status (by order_sequence) and creates status history entry
- On status update: Validates forward-only progression (increasing order_sequence)
- Status history is automatically tracked in ApplicationAssignedUserStatuses
- Supports status_notes and assigned_user_id fields for tracking

**Application Field Validations:**
- `phone_no` - Indian phone number (10 digits with optional +91 prefix)
- `zip_code` - 6-digit Indian PIN code
- `total_experience` - Integer (0-100 years)
- `relevant_experience` - Integer (0-100 years)
- `current_ctc` - Positive integer
- `expected_ctc` - Positive integer
- `resume` - Binary field (PDF, DOC, DOCX, max 5MB)
---

## 3. Authentication & Permissions

### 3.1 JWT Authentication

- **Access token expiry:** Configurable via `JWT_ACCESS_TOKEN_LIFETIME` (default: 60 minutes)
- **Refresh token expiry:** Configurable via `JWT_REFRESH_TOKEN_LIFETIME` (default: 10080 minutes / 7 days)
- **Token rotation:** Enabled - new refresh token issued on refresh
- **Token blacklisting:** Enabled - tokens are blacklisted after rotation and on logout
- **Algorithm:** HS256
- **Header type:** Bearer

### 3.2 Role-Based Access Control

**Simple Role System:**
The system uses a straightforward role-based approach with three predefined roles stored in the ROLES table:

- **Administrator** - Full access to all resources (users, jobs, applications, statuses)
- **Recruiter** - Can manage jobs and applications (Administrator + Recruiter permission)
- **Technical Evaluator** - Can view and update applications (Administrator + Recruiter + Technical Evaluator permission)

**Permission Implementation:**
- Each user has a single role assigned via `role_id` foreign key
- Permissions are checked based on the user's role name
- Custom permission classes in `common/permissions.py`:
  - `IsAdmin` - Only Administrator role
  - `IsRecruiter` - Administrator or Recruiter roles
  - `IsTechnicalEvaluator` - Administrator, Recruiter, or Technical Evaluator roles
  - `IsAdminOrSelf` - Administrator or the user themselves (object-level)

**Permission Hierarchy:**

Administrator (highest access) → Recruiter (can manage jobs and applications) → Technical Evaluator (can view and update applications)

---

## 4. Status Management

### 4.1 Application Status Workflow

The system uses a flexible status management system that allows administrators to:

1. **Create custom statuses** - Add new stages to the recruitment pipeline
2. **Define order** - Set the sequence of statuses
4. **Track transitions** - Monitor status changes over time

### 4.2 Status Management Best Practices

1. **Keep statuses sequential** - Use order_sequence to maintain logical flow
2. **Use descriptive names** - Clear status names help team communication
4. **Document transitions** - Add notes when changing application status

---

## 5. Soft Delete Implementation

### 5.1 Overview

The system implements soft delete functionality across all major tables to preserve data integrity and enable data recovery. Instead of permanently removing records from the database, records are marked as deleted using a `deleted_at` timestamp field.

### 5.2 Tables with Soft Delete

The following tables include the `deleted_at` field:

- **USERS** - User accounts
- **JOBS** - Job postings
- **APPLICATIONS** - Job applications
- **APPLICATION_ASSIGNED_USER_STATUSES** - Status tracking records

**Note:** The following tables do NOT have soft delete:
- **ROLES** - Role definitions (permanent)

### 5.3 Soft Delete Behavior

**Delete Operation:**
- When a DELETE request is made, the deleted_at timestamp is set to current time
- Record remains in database but is excluded from normal queries
- Original data is preserved for audit trails and recovery

**Query Filtering:**
- Default queryset filters exclude records where deleted_at is not null
- Soft-deleted records are hidden from standard API responses

### 5.4 Implementation Guidelines

**Model Level:**
- Abstract SoftDeleteModel class in common/models.py provides soft_delete() and restore() methods
- Models inherit from SoftDeleteModel to enable soft delete functionality
- deleted_at field is nullable DateTimeField with default None

**Usage:**
- Jobs and Applications models inherit from SoftDeleteModel
- ViewSet destroy() method calls instance.soft_delete() instead of hard delete
- Queryset automatically filters out soft-deleted records

---

## 6. File Storage & Resume Handling

### 6.1 Resume Storage in PostgreSQL

- **Storage Method:** BinaryField in APPLICATION model
- **Maximum file size:** 5MB
- **Allowed formats:** PDF
- **Validation:** Magic byte checking (not just extension)

### 6.2 Resume Upload/Download

**Upload Process:**
- Multipart form data with resume file
- File stored as BinaryField in PostgreSQL
- Maximum file size: 5MB (5,242,880 bytes)
- Allowed formats: PDF
- File type validation using python-magic (magic byte checking, not just extension)

---

## 7. Error Handling

### Error Response Format

All API errors follow a consistent format with error message, optional field-specific details, and error code.

### HTTP Status Codes

- **200 OK**: Successful GET, PUT, PATCH requests
- **201 Created**: Successful POST requests
- **204 No Content**: Successful DELETE requests
- **400 Bad Request**: Validation errors, duplicate applications
- **401 Unauthorized**: Missing or invalid JWT token
- **403 Forbidden**: Insufficient permissions
- **404 Not Found**: Resource not found
- **500 Internal Server Error**: Unexpected server errors

### Common Error Types

- **Duplicate Application**: User has already applied to the job
- **Invalid Resume File**: File size exceeds 5MB or invalid format
- **Missing Required Fields**: Required fields not provided in request
- **Unauthorized Access**: Authentication credentials not provided
- **Forbidden Access**: User lacks permission for the action

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

### 8.3 API Security
- JWT token validation
- CORS whitelist configuration
- Rate limiting

---

## 9. Testing Requirements

### 9.1 Test Coverage
- Used django internal toolig for testing

### 9.2 Test Types
- tests for API endpoints
- Permission tests

---

## 10. Future Enhancements

### 10.1 Phase 2 - AI-Powered Candidate Analysis

**AI Resume Analysis:**
- Intelligent resume parsing and data extraction
- Automated candidate ranking based on job requirements
- Skills matching and scoring

**Multi-Source Profile Analysis:**
- GitHub API integration for repository analysis
- LinkedIn API integration for professional profile analysis
- Portfolio website scraping and analysis
- Code quality assessment from GitHub contributions
- Skills verification across multiple platforms

**Smart Matching:**
- Candidate scoring beyond keyword matching
- Skills gap analysis
- Experience relevance scoring
- Cultural fit prediction

**Technical Implementation:**
- Machine learning models for candidate ranking
- Natural language processing for resume analysis
- API integrations (GitHub, LinkedIn)
- Background job processing with Celery + Redis

### 10.2 Phase 3 - AI Voice Calling Agent

**Automated Voice Interactions:**
- AI-powered voice calls to candidates
- Natural language understanding and generation
- Multi-language support

**Information Collection:**
- Notice period verification
- Joining date preferences
- Salary expectation confirmation
- Current employment status
- Availability for interviews

**Interview Scheduling:**
- Automated interview slot booking
- Calendar integration
- Interview confirmation calls
- Reminder calls before interviews
- Rescheduling handling

**Technical Implementation:**
- Voice AI integration (e.g., Twilio, AWS Connect)
- Speech-to-text and text-to-speech services
- Conversational AI framework
- CRM integration for call logging
- Calendar API integration


