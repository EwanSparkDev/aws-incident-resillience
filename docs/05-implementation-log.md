# AWS Infrastructure Implementation Log

**Project:** AWS Incident Management Platform

**Environment:** Proof of Concept

**AWS Region:** ap-southeast-2 (Sydney)

**Deployment Method:** AWS Management Console

---

## 1. Amazon DynamoDB

**Resource Name:** Incidents

**Status:** Deployed and Initial Tests Passed

### 1.1 Purpose

DynamoDB provides persistent storage for incident
records submitted through the application.

The database supports two primary retrieval patterns:

1. Employees retrieving their own tickets.
2. IT administrators retrieving all tickets.

### 1.2 Table Configuration

| Setting | Value |
|---|---|
| Table Name | Incidents |
| Partition Key | incidentId |
| Sort Key | None |
| Capacity Mode | On-Demand |
| Table Class | Standard |
| AWS Region | ap-southeast-2 |

### 1.3 Global Secondary Indexes

**RaisedByIndex**

- Partition Key: raisedBy
- Sort Key: createdAt
- Purpose: Retrieve incidents belonging to an employee.

**AdminIndex**

- Partition Key: adminGroup
- Sort Key: createdAt
- Purpose: Retrieve all incidents chronologically.
- PoC Partition Value: ALL

### 1.4 Testing

A synthetic incident was manually created using
the DynamoDB Console.

| Test | Expected Result | Status |
|---|---|---|
| Create Incident | Record successfully stored | PASS |
| Employee Query | RaisedByIndex returns matching ticket | PASS |
| Admin Query | AdminIndex returns ticket | PASS |


### 1.6 Test Evidence

#### Employee Query

The RaisedByIndex successfully returned incidents
matching the employee's unique identifier.

![Employee Query Test](../screenshots/employee-query.png)

#### Administrator Query

The AdminIndex successfully returned incident
records using the shared adminGroup partition key.

![Adminitrator Query](../screenshots/admin-query.png)


### 1.5 Architectural Considerations

- DynamoDB was selected for its managed serverless
  architecture and low operational overhead.
- The data model supports known application access
  patterns without routine table scans.
- The AdminIndex uses a shared partition key,
  which may limit scalability under high throughput.
- Global Secondary Indexes are eventually consistent.
- Production capacity and scalability have not
  yet been validated.

### 1.6 Next Steps

- Create the CreateIncident Lambda function.
- Configure least-privilege DynamoDB IAM permissions.
- Test programmatic incident creation.
- Implement API Gateway integration.

---

## 2. AWS Lambda

**Status:** Not Yet Deployed
