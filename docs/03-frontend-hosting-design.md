
# Frontend and Hosting Design

**Status:** Proposed

## Objective

Provide a secure, browser-based interface for employees
and IT administrators to manage incidents.

## Hosting Architecture

The frontend will be hosted using Amazon S3 and
distributed through Amazon CloudFront.

The S3 bucket will remain private using CloudFront
Origin Access Control (OAC).

## Frontend Technology

The initial application will use:

- HTML
- CSS
- JavaScript

## Application Views

1. Login
2. Submit Incident
3. Incident Dashboard

## Authentication

Cognito will authenticate approved users.

The browser will obtain an access token and include
it when making protected API requests.

API Gateway will validate the JWT.

Lambda will enforce user-level and role-level
permissions.

## API Operations

- POST /incidents
- GET /incidents
- GET /incidents/{id}

## Security

Frontend files contain no confidential incident data.

Ticket information is retrieved from protected APIs.

Employees may only access their own incidents.

IT administrators may access all incidents.

## Cost Considerations

- Use the default CloudFront domain.
- Avoid Route 53 and custom domain costs.
- Use a small private S3 bucket.
- Avoid unnecessary paid security services.
```

