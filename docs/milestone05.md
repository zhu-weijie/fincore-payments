### **Architect a Secure and Reliable PSP Webhook Handler**

*   **Problem:** Payment processing is frequently asynchronous. After our service requests a charge, the final confirmation (`succeeded` or `failed`) is sent by the PSP at a later time. Our system needs a secure and reliable way to receive these external, asynchronous updates to finalize the transaction's state in our database. Without this, payments would be perpetually stuck in the `PROCESSING` state.

*   **Solution:** We will create a dedicated, public API endpoint (e.g., `POST /v1/webhooks/stripe`) to receive notifications from the PSP. This handler must be robust and perform two critical functions in order:
    1.  **Signature Verification (Security):** The handler's first step must be to cryptographically verify the signature of every incoming webhook request. This is done using a shared secret key (provided by the PSP and stored securely by us) to ensure the request is authentic and originates from the correct PSP, preventing spoofing attacks.
    2.  **State Transition (Logic):** After successful verification, the handler will parse the event payload, look up the corresponding transaction in the `Database` using the `psp_charge_id`, and update its status to the final state (`SUCCEEDED` or `FAILED`).

*   **Trade-offs:**
    *   **Webhook Processing (Synchronous vs. Asynchronous):**
        *   **Pro:** For the MVP, the handler will process the webhook synchronously (verify, update DB, and return `200 OK`). This is simpler to implement and sufficient for initial traffic volumes.
        *   **Con:** If the processing logic becomes slow, it could cause timeouts for the PSP, leading them to retry the webhook and creating unnecessary load. We accept this trade-off for now and will introduce a queue for asynchronous processing in a later phase if needed.
    *   **Security of Webhook Secret:**
        *   **Pro:** Storing the webhook signing secret in **AWS Secrets Manager** is the most secure method. It keeps the secret out of code and environment variables, providing audited, IAM-controlled access.
        *   **Con:** It introduces a dependency on an external service for a critical security check. This is a standard and acceptable trade-off for the security benefits gained.

---

#### **Logical View (C4 Component Diagram)**

This diagram emphasizes the webhook flow. The key interaction is initiated by the external `Payment Service Provider`, which calls back into our `Payment Service` to provide the final transaction status.

```mermaid
graph TD
    subgraph "FinCore Payment System"
        payment_service["Payment Service<br>[Container]<br><br>Receives, verifies, and processes incoming webhooks."]
        db["Database<br>[Relational]<br><br>Stores the final state of the transaction."]
    end

    psp["Payment Service Provider<br>[External System]<br><br>Sends asynchronous status updates after processing."]

    psp -- "1.Sends Webhook (e.g., 'charge.succeeded')<br>Request includes a cryptographic signature." --> payment_service
    payment_service -- "2.Verifies signature and updates transaction status" --> db
```

---

#### **Physical View (AWS Deployment Diagram)**

The physical infrastructure remains the same. This diagram clarifies the data flow for an incoming webhook and highlights that the Webhook Secret is also stored in AWS Secrets Manager.

```mermaid
graph TD
    subgraph "AWS Cloud"
        subgraph "VPC"
            subgraph "Public Subnet"
                alb["Application Load Balancer (ALB)"]
            end

            subgraph "Private Subnet"
                fargate_service["AWS Fargate Service"]
                task["Fargate Task<br>(fincore-payments container)"]
                fargate_service -- "manages" --> task
                
                rds["Amazon RDS<br>PostgreSQL Instance"]
            end
        end

        secrets_manager["AWS Secrets Manager<br>(Stores PSP API Keys & Webhook Secrets)"]
    end

    internet["Internet"]
    psp_api["External PSP System"]

    internet -- "HTTPS Inbound" --> alb
    alb -- "forwards webhook traffic" --> task
    task -- "1.Retrieves Webhook Secret" --> secrets_manager
    task -- "2.Reads/Writes to DB" --> rds
```

---

#### **Component-to-Resource Mapping Table**

| Logical Component | Physical Resource | Rationale |
| :--- | :--- | :--- |
| **Payment Service** | **AWS Fargate Task** | (No change) The container's application logic now includes the webhook handling endpoint and signature verification logic. |
| **Database** | **Amazon RDS for PostgreSQL** | (No change) Stores the final, updated state of the payment. |
| **(Webhook Secret)** | **AWS Secrets Manager** | **Security:** The webhook signing secret is as sensitive as an API key. Storing it in Secrets Manager ensures it is managed securely and accessed only by the Fargate task via its IAM role. |
