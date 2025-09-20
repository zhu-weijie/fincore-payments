### **Architect Refunding Capabilities**

*   **Problem:** A core requirement for any payment system is the ability to issue refunds. Merchants need a way to return funds to a customer for a previously successful charge, either in full or partially. This process must be as reliable and auditable as the original payment.

*   **Solution:** We will introduce a refund flow that mirrors the asynchronous architecture of our payment flow. A new API endpoint, `POST /v1/refunds`, will be created. This endpoint will accept a `charge_id` and an optional `amount`. A new `refunds` table will be added to the database to track the state of each refund attempt. The refund process will leverage the existing job queue and worker infrastructure. The `PSP Gateway` will be extended to communicate with the PSP's refund API. This entire flow will be idempotent to prevent duplicate refunds.

*   **Trade-offs:**
    *   **Data Model (Separate Table):**
        *   **Pro:** Creating a dedicated `refunds` table is the correct accounting practice. It treats refunds as first-class, immutable events linked to a charge, rather than simply modifying the original charge. This provides a clear, auditable trail of all money movement.
        *   **Con:** Retrieving a charge with all its refunds requires a database `JOIN`, which is a standard and acceptable trade-off for data integrity.
    *   **Infrastructure Reuse:**
        *   **Pro:** Reusing the existing SQS queue and Lambda worker is highly efficient. It extends our reliable processing pattern to refunds without adding new infrastructure, reducing operational costs and complexity.
        *   **Con:** The job messages in the queue will now need an attribute (e.g., `type: 'REFUND'`) to distinguish them from charges. This requires the worker to have simple routing logic, which is a minor and manageable complexity.

---

#### **Logical View (C4 Component Diagram)**

The logical view is updated to show the new refund flow, which runs parallel to the existing charge flow, utilizing the same core components.

```mermaid
graph TD
    subgraph "FinCore Payment System"
        payment_service["Payment Service<br>[Container]<br><br>Handles API requests for refunds and enqueues jobs."]
        job_queue["Job Queue<br>[Queue]"]
        worker["Worker<br>[Asynchronous Processor]"]
        psp_gateway["PSP Gateway<br>[Internal Component]<br><br>Now includes logic for<br>calling PSP Refund API."]
        db["Database<br>[Relational]<br><br>Now includes a 'refunds' table."]
    end

    api_client["API Client"]
    psp["Payment Service Provider"]

    api_client -- "1.POST /refunds" --> payment_service
    payment_service -- "2.Creates 'refund' record in DB" --> db
    payment_service -- "3.Enqueues 'Process Refund' job" --> job_queue
    
    worker -- "4.Dequeues job" --> job_queue
    worker -- "5.Calls refund method on" --> psp_gateway
    psp_gateway -- "6.Sends Refund API call to" --> psp
    
    psp -- "7.Sends 'refund.succeeded' Webhook" --> payment_service
    payment_service -- "8.Updates refund status in DB" --> db
```

---

#### **Physical View (AWS Deployment Diagram)**

No new physical resources are required. This diagram illustrates how the existing infrastructure handles the new refund workflow. The primary changes are in the application logic within the Fargate task and the Lambda worker, and the addition of a new table in the existing RDS database.

```mermaid
graph TD
    subgraph "AWS Cloud"
        subgraph "VPC"
            subgraph "Private Subnet (Application)"
                fargate_service["AWS Fargate Service"]
                task["Fargate Task<br>(Payment Service)<br>+ /refunds endpoint"]
                fargate_service -- "manages" --> task

                sqs["Amazon SQS Queue<br>Handles charge & refund jobs"]
                
                lambda_worker["AWS Lambda Function<br>(Worker)<br>+ refund processing logic"]
                lambda_worker -- "triggered by" --> sqs
            end
            
            subgraph "Private Subnet (Database)"
                rds["Amazon RDS<br>+ refunds table"]
            end
        end
    end

    task -- "enqueues to" --> sqs
    lambda_worker -- "reads/writes to" --> rds
```

---

#### **Component-to-Resource Mapping Table**

| Logical Component | Physical Resource | Rationale |
| :--- | :--- | :--- |
| **Payment Service** | **AWS Fargate Task** | **Role Expanded:** The logic is updated to include the `POST /v1/refunds` endpoint, request validation, and the creation and enqueuing of refund jobs. |
| **Worker** | **AWS Lambda Function** | **Role Expanded:** The worker logic is updated to handle messages of type `REFUND`, directing them to the `PSP Gateway`'s refund functionality. The existing retry and DLQ mechanisms are reused. |
| **Database** | **Amazon RDS for PostgreSQL** | **Schema Expanded:** A new `refunds` table is added, with a foreign key linking back to the `payments` table. This maintains relational integrity. |
| **Job Queue** | **Amazon SQS** | (No change in resource) **Payload Updated:** The message format is updated to include a `job_type` field to differentiate between charge and refund jobs. |
| **PSP Gateway** | **AWS Fargate Task** (module within the container) | **Logic Expanded:** The internal gateway module is updated with a new method to handle the specifics of the PSP's refund API. |
