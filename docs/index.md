# Recruitment Management System (RMS)

## Executive Summary

Recruitment Management System (RMS): a cloud-hosted web app to streamline job posting, applicant tracking, status updates, and role-based access. Built with React, Django REST Framework, and PostgreSQL (RDS), deployed via CI/CD on AWS, with secure JWT authentication and a scalable backend—foundation for a future AI ATS.

## Purpose and Scope

### Purpose

The purpose of this project is to design and implement a centralized recruitment management system that digitizes and automates essential HR functions like job management, application tracking, and department-wise reviews.

### Scope

The system currently supports:

- Secure login and registration for Administrator ,Recruiter and Technical Evaluator
- Job posting and management
- application submission
- Recruitment status updates and interview sheduling
- Role-based permissions for data access and actions
- CI/CD deployment through GitHub Actions with AWS hosting

## System Architecture Overview

The system follows a modular, layered architecture deployed on AWS Cloud Infrastructure.

### 1. CI/CD Pipeline

- GitHub Actions handle automated build and deployment
- The frontend (React) and backend (Django REST Framework) are hosted separately, ensuring independent scalability
- Backend is deployed to AWS EC2, and frontend assets are served from AWS S3 via CloudFront for global availability

### 2. Backend Layer (Django REST Framework on EC2)

- Handles all business logic and RESTful API endpoints
- Uses JWT-based middleware for authentication and authorization
- Implements modular structure with Views, Serializers, and Models
- Interacts with PostgreSQL RDS for secure data storage and querying

### 3. Frontend Layer (React + Axios)

- Developed using React.js for a responsive, interactive UI
- Axios middleware manages API communication
- State management ensures smooth user experience across modules

### 4. Database Layer (PostgreSQL RDS)

- Stores structured data for users, roles, jobs, applicants, and applications
- Simple role-based access control with a dedicated ROLES table

## Architecture Diagram

![System Architecture](diagram-export-11-11-2025-1_12_23-PM.png)

## Cloud Deployment Architecture

![AWS Cloud Architecture](Blank diagram.png)

### Data Flow Diagram (Level 0)

```mermaid
flowchart TB
    %% External Entities
    Applicant([Job Applicant])
    Admin([Administrator])
    Recruiter([Recruiter])
    Evaluator([Technical Evaluator])
    
    %% Main System
    RMS[Recruitment Management System]
    
    %% Data Store
    DB[(Database)]
    
    %% Applicant Flows
    Applicant -->|Submit Application| RMS
    RMS -->|Job Listings| Applicant
    
    %% Administrator Flows
    Admin -->|Manage Users, Roles & Statuses| RMS
    
    %% Recruiter Flows
    Recruiter -->|Create Jobs & Update Applications| RMS
    
    %% Technical Evaluator Flows
    Evaluator -->|Add Technical Notes| RMS
    
    %% System to Database
    RMS <-->|Store/Retrieve Data| DB
    
    style RMS fill:#4A90E2,stroke:#2E5C8A,stroke-width:3px,color:#fff
    style DB fill:#50C878,stroke:#2D7A4A,stroke-width:2px,color:#fff
    style Applicant fill:#FFB84D,stroke:#CC8A3D,stroke-width:2px
    style Admin fill:#E74C3C,stroke:#C0392B,stroke-width:2px,color:#fff
    style Recruiter fill:#9B59B6,stroke:#7D3C98,stroke-width:2px,color:#fff
    style Evaluator fill:#3498DB,stroke:#2874A6,stroke-width:2px,color:#fff
```

## Database Architecture (ERD Overview)

The ER diagram defines clear relationships between the system's entities:

- **USERS** – System users (Administrator, Recruiter, Technical Evaluator) with UUID primary keys
- **ROLES** – Simple role definitions (Administrator, Recruiter, Technical Evaluator)
- **JOBS** – Job listings with complete details including salary range, requirements, and benefits
  - Uses UUID primary keys
  - Status choices: Open, Closed 
  - Auto-sets closing_date when status changes to Closed
- **APPLICATIONS** – Contains all applicant information and application data
  - Uses UUID primary keys
  - Stores resume as binary data in PostgreSQL
  - Indian phone number validation (10 digits with optional +91)
  - Indian PIN code validation (6 digits)
  - Experience fields are integers (0-100 years)
  - CTC fields are integers (not decimals)
  - References current_status directly 
- **APPLICATION_STATUSES** – Defines recruitment stages with auto-incrementing order_sequence
  - Uses UUID primary keys
  - Unique constraint on order_sequence with deferred checking
- **APPLICATION_ASSIGNED_USER_STATUSES** – Tracks status changes and user assignments
  - Uses UUID primary keys
  - Removed status_date field (uses created_at instead) 

> **Note:** For complete technical specifications, API endpoints, and implementation details, see the [Technical Documentation](technical-documentation.md) page.



### Database Entity Relationship Diagram

```mermaid
erDiagram
    %% ===== CORE RELATIONSHIPS =====
    USERS }o--|| ROLES : "has role"
    USERS ||--o{ APPLICATION_ASSIGNED_USER_STATUSES : "reviews / updates"
    USERS ||--o{ JOBS : "posts"
    
    JOBS ||--o{ APPLICATIONS : "receives"
    
    APPLICATIONS }o--|| APPLICATION_STATUSES : "has current status"
    APPLICATIONS ||--o{ APPLICATION_ASSIGNED_USER_STATUSES : "tracked by"
    APPLICATION_ASSIGNED_USER_STATUSES }o--|| APPLICATION_STATUSES : "records status"
    APPLICATION_ASSIGNED_USER_STATUSES }o--|| USERS : "assigned to"
    
    %% ===== CORE TABLES =====
    
    USERS {
        uuid id PK "Primary Key, UUID"
        string email UK "Unique, NOT NULL, max_length=254, indexed"
        string password "NOT NULL (hashed with PBKDF2)"
        string first_name "NOT NULL, max_length=255"
        string last_name "NOT NULL, max_length=255"
        uuid role_id FK "References ROLES(id)"
        boolean is_active "Default=True"
        boolean is_staff "Default=False"
        datetime last_login "Nullable"
        datetime created_at "NOT NULL, default=NOW()"
        datetime updated_at "NOT NULL, auto-updated"
        datetime deleted_at "Nullable, for soft delete"
    }
    
    ROLES {
        uuid id PK "Primary Key, UUID"
        string name UK "Unique, NOT NULL, max_length=100"
        text description "Optional"
        datetime created_at "NOT NULL, default=NOW()"
    }
    
    %% ===== APPLICATION TABLES =====
    
    JOBS {
        uuid id PK "Primary Key, UUID"
        string title "NOT NULL, max_length=255"
        text description "NOT NULL"
        string employment_type "NOT NULL, choices: Full-time, Part-time, Contract, Internship"
        string location "NOT NULL, max_length=255"
        string department "NOT NULL"
        string experience_level "NOT NULL, choices: Entry, Mid, Senior, Lead"
        int salary_min "Optional, PositiveInteger"
        int salary_max "Optional, PositiveInteger"
        text requirements "NOT NULL"
        text benefits "Optional"
        uuid posted_by_user_id FK "References USERS(id) ON DELETE SET NULL"
        datetime closing_date "Optional, auto-set when status=Closed"
        string status "Default='Open', choices: Open, Closed"
        datetime created_at "NOT NULL, default=NOW()"
        datetime updated_at "NOT NULL, auto-updated"
        datetime deleted_at "Nullable, for soft delete"
    }
    
    APPLICATIONS {
        uuid id PK "Primary Key, UUID"
        uuid job_id FK "References JOBS(id) ON DELETE CASCADE"
        string first_name "NOT NULL, max_length=255"
        string last_name "NOT NULL, max_length=255"
        string email "NOT NULL, indexed"
        string phone_no "NOT NULL, max_length=13, Indian phone validation"
        binary resume "NOT NULL, stored in PostgreSQL"
        int total_experience "Optional, 0-100 years"
        int relevant_experience "Optional, 0-100 years"
        int current_ctc "Optional, PositiveInteger"
        int expected_ctc "Optional, PositiveInteger"
        string notice_period "Optional, max_length=255"
        string current_job_title "Optional, max_length=255"
        url linkedin "Optional, max_length=255"
        url github "Optional, max_length=255"
        string street "NOT NULL, max_length=255"
        string city "NOT NULL, max_length=255"
        string zip_code "NOT NULL, 6-digit Indian PIN code"
        uuid current_status FK "References APPLICATION_STATUSES(id) ON DELETE PROTECT"
        datetime created_at "NOT NULL, default=NOW()"
        datetime updated_at "NOT NULL, auto-updated"
        datetime deleted_at "Nullable, for soft delete"
    }
    
    APPLICATION_STATUSES {
        uuid id PK "Primary Key, UUID"
        string name UK "Unique, NOT NULL, max_length=255"
        text description "NOT NULL"
        int order_sequence "NOT NULL, auto-increment, indexed, unique"
        datetime created_at "NOT NULL, default=NOW()"
        datetime updated_at "NOT NULL, auto-updated"
    }
    
    APPLICATION_ASSIGNED_USER_STATUSES {
        uuid id PK "Primary Key, UUID"
        uuid application_id FK "References APPLICATIONS(id) ON DELETE CASCADE"
        uuid status_id FK "References APPLICATION_STATUSES(id) ON DELETE RESTRICT"
        uuid assigned_user_id FK "References USERS(id) ON DELETE SET NULL"
        text notes "Optional (review feedback or comments)"
        datetime created_at "NOT NULL, default=NOW()"
        datetime updated_at "NOT NULL, auto-updated"
        datetime deleted_at "Nullable, for soft delete"
    }
```

### Database Indexing Strategy

Essential indexes for better performance (Django creates most of these automatically):

#### USERS Table
- `id` - Primary key (UUID, auto-indexed)
- `email` - Unique field with explicit index for login and user search
- `role_id` - Foreign key (auto-indexed by Django)

#### JOBS Table
- `id` - Primary key (UUID, auto-indexed)
- `status` - For filtering open/closed jobs
- `department` - For department filtering
- `posted_by_user_id` - Foreign key (auto-indexed by Django)

#### APPLICATIONS Table
- `id` - Primary key (UUID, auto-indexed)
- `email` - Indexed for searching applicants
- `job_id` - Foreign key (auto-indexed by Django)
- `current_status` - Foreign key (auto-indexed by Django)

#### APPLICATION_STATUSES Table
- `id` - Primary key (UUID, auto-indexed)
- `name` - Unique field (auto-indexed)
- `order_sequence` - Explicitly indexed for ordering, unique constraint

> **Note:** Django automatically creates indexes for Primary Keys (UUIDs), Foreign Keys, and fields with `unique=True`. Additional indexes are added for frequently filtered fields like `email` and `order_sequence`.

## Goals

Build a working MVP for recruitment management. Future plan: add AI agents for ranking candidates, analyzing GitHub/LinkedIn profiles, automated phone calls for repetitive tasks like sheduling interview ,getting notice period other stuff , and smart filtering beyond just resumes.

## Functional Modules

### 1. Authentication and Authorization

- JWT-based login and sign-up system using Django REST Framework SimpleJWT
- Simple role-based access control with three roles: Administrator, Recruiter, and Technical Evaluator

### 2. Job Management

- Create, edit, and delete job postings
- Assign jobs to departments and specify employment details

### 3. Applicant Management

- Candidate registration and resume upload
- Apply for specific job openings

### 4. Application Tracking

- Track status transitions such as Applied, Under Review, Shortlisted, Rejected, or Hired

## Technical Requirements

| Component | Technology Stack |
|-----------|-----------------|
| Frontend | React.js, Axios, Tailwind CSS |
| Backend | Django 4.2+, Django REST Framework 3.14+, Python 3.10+ |
| Database | PostgreSQL 14+ (AWS RDS) with UUID primary keys |
| Hosting | AWS EC2 (Backend), AWS S3 + CloudFront (Frontend) |
| CI/CD | GitHub Actions |
| Authentication | JWT via djangorestframework-simplejwt with token blacklisting |
| Configuration | python-decouple for environment variables |
| File Validation | python-magic for resume file type checking |
| Version Control | GitHub |

## Constraints

- Developed and maintained by a single developer; not much can be done
- Focused on building a solid foundation for data management and workflows
- AI-based candidate filtering and predictive analytics are deferred for later integration

## Use Cases

### Use Case 1: Applicant Applies for Job

**Actor:** Job Applicant

**Steps:**

1. Visit website and browse jobs
2. Click on a job to see details
3. Upload resume (PDF or DOC)
4. Fill out application form
5. Submit application
6. Get confirmation email

**Notes:**

- No login needed to browse jobs
- No login required 

---

### Use Case 2: Admin Adds New User

**Actor:** Admin

**Steps:**

1. Login to admin panel
2. Go to user management
3. Enter new user's email
4. Select their department
5. Assign group (Administrator, Recruiter and Technical Evaluator)
6. System sends invite email

**What happens next:**

- New user gets email with setup link
- Link expires in 48 hours
- User clicks link and creates password
- User can now login

---

### Use Case 3: HR Reviews Applications

**Actor:** HR Team Member

**Steps:**

1. Login to dashboard
2. See list of applications
3. Filter by status or job position
4. Click on an application to open it
5. Review resume and application form
6. Add notes for team
7. Update status (shortlist, reject, or interview)
8. System emails applicant about status change

**Notes:**

- Multiple HR members can review same application
- All changes are logged with timestamps

---

## Future Enhancements

### Phase 1 - Core Features (Current MVP)
- Job posting and management
- Application submission and tracking
- Role-based access control (Administrator, Recruiter, Technical Evaluator)
- Resume storage in PostgreSQL
- Status management and workflow
- JWT authentication with token blacklisting

### Phase 2 - AI-Powered Candidate Analysis
- AI resume parsing and intelligent ranking
- GitHub profile analysis (repositories, contributions, code quality)
- LinkedIn profile analysis (experience, skills, endorsements)
- Portfolio website analysis
- Smart candidate matching beyond just resume keywords
- Automated candidate scoring based on multiple data sources
- Skills gap analysis

### Phase 3 - AI Voice Calling Agent
- Automated phone calls for repetitive tasks
- Get notice period information from candidates
- Collect joining date preferences
- Verify salary expectations
- Schedule interview slots automatically
- Send interview confirmations
- Follow-up calls for pending information

## Conclusion

The Recruitment Management System (RMS) provides a reliable foundation for automating company internal hiring process. By combining Django REST Framework, React, and AWS, it establishes a scalable and secure system architecture.

This base version focuses on streamlining recruitment workflows and cloud-ready infrastructure—serving as a strong platform for future expansion into a full-featured AI-driven Applicant Tracking System (ATS).
