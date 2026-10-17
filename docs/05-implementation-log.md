# AWS Infrastructure Implementation Log

**Project:** AWS Incident Management Platform — Project Resilience  
**Environment:** Proof of Concept (PoC)  
**AWS Region:** ap-southeast-2 (Sydney)  
**Deployment Method:** AWS Management Console

---

## 1. Amazon DynamoDB

**Resource Name:** `Incidents`  
**Status:** Deployed — Initial Tests Passed

### 1.1 Purpose

Amazon DynamoDB provides persistent storage for incidents submitted through the application.

The database supports two primary retrieval patterns:

1. Employees retrieving their own tickets.
2. IT administrators retrieving all tickets.

The data model was designed around these access patterns to avoid relying on inefficient full-table scans.

### 1.2 Table Configuration

| Setting | Value |
|---|---|
| Table Name | `Incidents` |
| Partition Key | `incidentId` (String) |
| Sort Key | None |
| Capacity Mode | On-Demand |
| Table Class | Standard |
| AWS Region | ap-southeast-2 |

Each incident is uniquely identified by its `incidentId` partition key.

### 1.3 Global Secondary Indexes

**RaisedByIndex**

- Partition Key: `raisedBy`
- Sort Key: `createdAt`
- Purpose: Retrieve incidents belonging to a specific employee.
- Query Pattern: Retrieve incidents where `raisedBy` matches the authenticated employee's Cognito `sub`.

**AdminIndex**

- Partition Key: `adminGroup`
- Sort Key: `createdAt`
- Purpose: Retrieve incidents across the organisation in timestamp order.
- PoC Partition Value: `ALL`
- Query Pattern: Retrieve incidents where `adminGroup = ALL`.

Both indexes enable efficient queries based on the application's requirements.

### 1.4 Testing

A synthetic incident was manually created using the DynamoDB Console.

| Test | Expected Result | Status |
|---|---|---|
| Manual Incident Creation | Record successfully stored | PASS |
| Employee Query | RaisedByIndex returns matching incident | PASS |
| Administrator Query | AdminIndex returns incident | PASS |

Both GSI query patterns were successfully validated using the DynamoDB Console.

### 1.5 Test Evidence

#### Employee Query

The `RaisedByIndex` successfully returned incident records matching the employee's unique identifier.

![Employee Query Test](../screenshots/employee-query.png)

#### Administrator Query

The `AdminIndex` successfully returned incident records using the shared `adminGroup` partition key.

![Administrator Query Test](../screenshots/admin-query.png)

### 1.6 Architectural Considerations

- DynamoDB was selected for its managed, serverless architecture and low operational overhead.
- On-demand capacity avoids the need to provision read and write capacity for the PoC.
- The data model supports known application access patterns without routine full-table scans.
- The `AdminIndex` uses a shared partition key (`ALL`), which may create scalability limitations at higher throughput.
- Global Secondary Indexes are eventually consistent.
- Individual reads from the primary table can use strongly consistent reads where required.
- Production capacity, performance, and scalability have not yet been validated.

---

## 2. AWS Lambda — CreateIncident

**Function Name:** `CreateIncident`  
**Runtime:** Python 3.14  
**Region:** ap-southeast-2 (Sydney)  
**Status:** Deployed — Initial Tests Passed

### 2.1 Purpose

The `CreateIncident` Lambda function processes incident submissions and persists validated records to DynamoDB.

The function is responsible for:

1. Receiving incident data.
2. Validating the submitted fields.
3. Extracting employee identity and group information.
4. Generating a unique incident reference.
5. Recording the creation timestamp.
6. Writing the incident to DynamoDB.
7. Returning a successful response only after the database acknowledges the write.

### 2.2 Implementation

The function was implemented using Python 3.14 and the AWS SDK for Python (Boto3).

**Input Validation**

The function validates the following fields:

- `title` — Required string, maximum 150 characters.
- `description` — Required string, maximum 2,000 characters.
- `severity` — Must be `LOW`, `MEDIUM`, or `HIGH`.

Invalid submissions receive an HTTP `400 Bad Request` response.

**Incident Reference Generation**

Each incident receives a unique identifier generated using Python's `uuid` library.

Example format:

`INC-550e8400-e29b-41d4-a716-446655440000`

UUIDs were selected over sequential references to avoid additional database coordination and complexity.

**Identity and Authorisation**

The function extracts the Cognito user identifier (`sub`) and group membership from the API Gateway JWT authorizer claims structure.

Permitted groups:

- `Employees`
- `ITAdmins`

The `raisedBy` attribute is derived from the identity claims rather than a user-supplied request field.

**DynamoDB Integration**

The function uses Boto3's `PutItem` operation to write incident records.

Each record includes:

| Attribute | Purpose |
|---|---|
| `incidentId` | Unique incident reference |
| `raisedBy` | Employee identifier |
| `adminGroup` | Shared administrator query value |
| `createdAt` | UTC creation timestamp |
| `title` | Incident title |
| `description` | Incident description |
| `severity` | Incident priority |
| `status` | Initial incident status |

New incidents receive the status `SUBMITTED`.

A conditional expression prevents a write from overwriting an existing record with the same `incidentId`.

### 2.3 IAM Security

The Lambda function uses an IAM execution role.

Permissions include:

- CloudWatch Logs permissions for Lambda execution logging.
- `dynamodb:PutItem` permission for the `Incidents` table.

The function does not require DynamoDB read, scan, or delete permissions.

This follows the principle of least privilege.

### 2.4 Testing

Testing was performed using Lambda Console test events.

| Test | Expected Result | Status |
|---|---|---|
| Valid Incident Submission | HTTP 201 Created | PASS |
| DynamoDB Record Persistence | Incident successfully stored | PASS |
| Invalid Severity (`CRITICAL`) | HTTP 400 Bad Request | PASS |
| Missing User Identity | HTTP 401 Unauthorized | PASS |
| Unauthorised Group | HTTP 403 Forbidden | PASS |

The HTTP `401` and `403` tests used simulated JWT claims in the Lambda Console.

These tests validate the Lambda's application logic but do not verify real Cognito authentication or API Gateway JWT validation.

### 2.5 Architectural Limitations

- Real Cognito authentication and JWT validation have not yet been integrated.
- UUIDs provide unique incident references but are not chronological ticket numbers.
- Conditional writes prevent incident ID collisions but do not provide request-level idempotency.
- Retried submissions could create multiple incidents if the initial outcome is uncertain.
- Additional exception handling is required for network timeouts and other infrastructure failures.
- Production-grade monitoring, alarms, and recovery procedures remain outstanding.

The function has successfully demonstrated the core synchronous write path, but it is not yet production-ready.

---

## 3. AWS Lambda — ReadIncident

**Planned Function Name:** `ReadIncident`  
**Runtime:** Python 3.14  
**Region:** ap-southeast-2 (Sydney)  
**Status:** Planned — Not Yet Deployed

### 3.1 Purpose

The `ReadIncident` Lambda function will retrieve incident records from DynamoDB and enforce application-level access controls based on the authenticated user's Cognito identity and group membership.

The function will support both employee and IT administrator access patterns.

### 3.2 Functional Requirements

**Employee Access**

- Employees can retrieve incidents they personally submitted.
- Employee queries use the `RaisedByIndex` GSI.
- Employees cannot retrieve incidents belonging to other users.

**Administrator Access**

- IT administrators can retrieve all submitted incidents.
- Administrator queries use the `AdminIndex` GSI.
- Administrators can retrieve individual incident records.

### 3.3 API Endpoints (Planned)

| Method | Endpoint | Purpose |
|---|---|---|
| GET | `/incidents` | Retrieve incidents accessible to the authenticated user |
| GET | `/incidents/{id}` | Retrieve an individual incident |

API Gateway will route these requests to the `ReadIncident` Lambda function.

### 3.4 DynamoDB Access Patterns

**Employee Incident Listing**

- Operation: `Query`
- Index: `RaisedByIndex`
- Partition Key: `raisedBy`
- Value: Authenticated user's Cognito `sub`
- Purpose: Retrieve only the employee's own incidents.

**Administrator Incident Listing**

- Operation: `Query`
- Index: `AdminIndex`
- Partition Key: `adminGroup`
- Value: `ALL`
- Purpose: Retrieve incidents across the organisation.

**Individual Incident Retrieval**

- Operation: `GetItem`
- Table: `Incidents`
- Partition Key: `incidentId`

Before returning an incident, Lambda must verify that either:

1. The incident's `raisedBy` matches the authenticated user's `sub`, or
2. The authenticated user belongs to the `ITAdmins` group.

### 3.5 Security Design

- API Gateway will validate Cognito JWT tokens before invoking Lambda.
- Lambda will extract the user identity from validated claims.
- Employee access will be restricted to incidents matching the authenticated user's identity.
- Administrator access will require membership of the `ITAdmins` group.
- IAM permissions will allow only the required DynamoDB read operations.
- No routine full-table scans will be used.

**Important:** DynamoDB indexes are query mechanisms, not authorisation boundaries.

The Lambda function must enforce record-level access restrictions before returning incident data.

### 3.6 Implementation Plan

- [ ] Create `ReadIncident` Lambda function.
- [ ] Configure least-privilege DynamoDB read permissions.
- [ ] Implement employee incident queries.
- [ ] Implement administrator incident queries.
- [ ] Implement individual incident retrieval.
- [ ] Implement ownership and group authorisation checks.
- [ ] Test successful and unauthorised retrieval.
- [ ] Integrate with API Gateway and Cognito.

### 3.7 Architectural Considerations

- DynamoDB Global Secondary Indexes are eventually consistent.
- Individual `GetItem` requests can use strongly consistent reads.
- Query results may require pagination.
- Employee and administrator queries should return results in the required timestamp order.
- The shared `AdminIndex` partition key is acceptable for the current PoC but may need revisiting at larger scale.
- GSI access patterns have been validated manually, but Lambda read functionality and real JWT authorisation remain untested.

### 3.8 Test Results

**Status:** Pending implementation and testing.

Test evidence and results will be added following deployment.

---

## 4. Outstanding Work / Next Steps

### 4.1 Amazon DynamoDB

- [x] Create `Incidents` table.
- [x] Configure `RaisedByIndex` GSI.
- [x] Configure `AdminIndex` GSI.
- [x] Validate employee queries.
- [x] Validate administrator queries.
- [x] Document configuration and test evidence.

### 4.2 AWS Lambda — CreateIncident

- [x] Deploy Python 3.14 Lambda function.
- [x] Configure least-privilege DynamoDB write permissions.
- [x] Verify successful ticket creation (HTTP 201).
- [x] Verify DynamoDB record persistence.
- [x] Reject invalid severity (HTTP 400).
- [x] Test missing identity (HTTP 401, simulated claims).
- [x] Test unauthorised group (HTTP 403, simulated claims).
- [ ] Implement robust idempotency and retry handling.
- [ ] Improve infrastructure error handling.

### 4.3 AWS Lambda — ReadIncident

- [ ] Create `ReadIncident` Lambda function.
- [ ] Configure least-privilege DynamoDB read permissions.
- [ ] Implement employee incident retrieval.
- [ ] Implement administrator incident retrieval.
- [ ] Implement individual incident retrieval.
- [ ] Verify record-level authorisation.
- [ ] Document test results.

### 4.4 API Gateway and Authentication

- [ ] Configure Amazon Cognito user pool.
- [ ] Disable public self-registration.
- [ ] Create approved employee and administrator test users.
- [ ] Configure `Employees` and `ITAdmins` groups.
- [ ] Deploy API Gateway HTTP API.
- [ ] Configure JWT authorizer and protected routes.
- [ ] Integrate Lambda functions.
- [ ] Configure CORS for frontend access.
- [ ] Test real authentication and authorisation.

### 4.5 Frontend Hosting

- [ ] Create private S3 bucket for static frontend.
- [ ] Configure CloudFront with Origin Access Control.
- [ ] Implement login and logout.
- [ ] Implement incident submission interface.
- [ ] Implement employee incident dashboard.
- [ ] Implement administrator incident dashboard.
- [ ] Test end-to-end application functionality.

### 4.6 Notifications

- [ ] Enable DynamoDB Streams.
- [ ] Create notification Lambda function.
- [ ] Configure Amazon SES.
- [ ] Implement personalised incident confirmation emails.
- [ ] Configure stream-processing retry and failure handling.
- [ ] Test that notification failures do not prevent successful incident creation.

### 4.7 Monitoring, Reliability and Cost

- [ ] Configure CloudWatch monitoring and alarms.
- [ ] Test error handling and failure scenarios.
- [ ] Review request-level idempotency.
- [ ] Verify AWS budget alerts.
- [ ] Review resource security and IAM permissions.
- [ ] Review AWS costs and remove unnecessary test resources.

---

## 5. Implementation Summary

| Component | Status |
|---|---|
| DynamoDB Incidents Table | Deployed and Tested |
| RaisedByIndex | Deployed and Tested |
| AdminIndex | Deployed and Tested |
| CreateIncident Lambda | Deployed and Initial Tests Passed |
| ReadIncident Lambda | Planned |
| Amazon Cognito | Not Yet Deployed |
| API Gateway | Not Yet Deployed |
| S3 and CloudFront | Not Yet Deployed |
| DynamoDB Streams | Not Yet Configured |
| Notification Lambda | Not Yet Deployed |
| Amazon SES | Not Yet Configured |

### Current Project Milestone

The project has successfully implemented and validated the core incident creation functionality using AWS Lambda and Amazon DynamoDB.

The next implementation milestone is deploying `ReadIncident` and verifying role-based incident retrieval.

Authentication, frontend hosting, asynchronous notifications, and production-grade reliability features remain future implementation stages.
