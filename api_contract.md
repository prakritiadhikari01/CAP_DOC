# NIC CAP API Contract

Version: v1

Base URL:

/api/v1/

## Authentication

### Login

POST /auth/login

Request:

{ "organization_email":"","password":"" }

Response:

{ "access":"","refresh":"","role":"AMBASSADOR" }

------------------------------------------------------------------------

## Application APIs

### Submit Application

POST /applications

Access: Public

### View Application Status

GET /applications/{id}

Access: Applicant/Admin

### Admin Application Queue

GET /admin/applications

### Approve or Reject Application

PATCH /admin/applications/{id}/decision

------------------------------------------------------------------------

# Ambassador APIs

## Public Directory

GET /ambassadors

GET /ambassadors/{id}

## Current Profile

GET /me/profile

------------------------------------------------------------------------

# Verification APIs

## Submit Profile Change

POST /me/profile/change-requests

Example:

{ "section":"skills", "proposed_data":{ "skills":\["Python","React"\] }
}

## View Own Requests

GET /me/profile/change-requests

## Submit Activity Report

POST /me/reports

## View Own Reports

GET /me/reports

------------------------------------------------------------------------

# Admin Verification APIs

GET /admin/change-requests

PATCH /admin/change-requests/{id}/approve

PATCH /admin/change-requests/{id}/reject

GET /admin/reports

PATCH /admin/reports/{id}/approve

PATCH /admin/reports/{id}/reject

------------------------------------------------------------------------

# Event APIs

GET /events

GET /events/{id}

POST /events/{id}/register

Admin:

POST /admin/events

PUT /admin/events/{id}

DELETE /admin/events/{id}

------------------------------------------------------------------------

# Dashboard APIs

GET /admin/dashboard

GET /admin/analytics

------------------------------------------------------------------------

# Error Format

{ "error":{ "code":"","message":"","fields":{} } }

Status Codes:

400 Validation Error 401 Authentication Error 403 Permission Error 404
Not Found 409 Conflict 422 Business Rule Error
