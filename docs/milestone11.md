### **Architect the Automated Reconciliation Service**

*   **Problem:** Our system's ledger, while internally consistent, is not guaranteed to be a perfect match with the Payment Service Provider's records. Issues like missed webhooks, network partitions, or timing differences can lead to discrepancies. Manually comparing these records is not scalable and is a significant operational burden for the finance team.

*   **Solution:** We will create an automated, scheduled `Reconciliation Service`. This service will run on a periodic basis (e.g., daily at 02:00 UTC). Its responsibilities will be:
    1.  Securely connect to an external source (e.g., a PSP's SFTP server) to download the daily settlement report.
    2.  Parse this report (which could be in formats like CSV or XML).
    3.  Query our `LedgerDB` for all transactions within the same settlement period.
    4.  Perform a comparison to identify discrepancies (e.g., amount mismatches, or transactions present in one system but not the other).
    5.  Generate a structured `Discrepancy Report`, store it, and send a notification to an operations/finance team for review.

*   **Trade-offs:**
    *   **Implementation as a Scheduled Job:**
        *   **Pro:** A scheduled, serverless approach using **Amazon EventBridge** to trigger an **AWS Lambda function** is highly cost-effective and operationally simple. We only pay for the brief time the reconciliation job is running, and there is no need to manage servers or containers for this periodic task.
        *   **Con:** AWS Lambda has a maximum execution timeout (15 minutes). If settlement files become extremely large and take longer than this to process, we would need to migrate the logic to a long-running service like AWS Fargate or AWS Batch. The Lambda-first approach is the best starting point.
    *   **Report Storage:**
        *   **Pro:** Storing the generated discrepancy reports in **Amazon S3** is a durable, secure, and cost-effective solution. It provides a historical archive of all reconciliation activities.
        *   **Con:** Requires setting up appropriate lifecycle policies to manage storage costs over time.

---

#### **Logical View (C4 Component Diagram)**

The logical view introduces a new `Reconciliation Service`, which acts as a scheduled job. It interacts with both our internal `LedgerDB` and an external PSP system to perform its function.

```mermaid
graph TD
    subgraph "FinCore Payment System"
        reconciliation_service["Reconciliation Service<br>[Scheduled Job]<br><br>Fetches, compares, and reports on settlement data."]
        ledger_db["LedgerDB<br>[Read Model]"]
    end

    subgraph "External Systems"
        psp_settlement["PSP Settlement System<br>[e.g., SFTP Server]"]
        ops_team["Operations/Finance Team<br>[Person]"]
    end
    
    reconciliation_service -- "1.Runs on a schedule (e.g., daily)" --> reconciliation_service
    reconciliation_service -- "2.Fetches Settlement File" --> psp_settlement
    reconciliation_service -- "3.Reads transaction data" --> ledger_db
    reconciliation_service -- "4.Generates Discrepancy Report and notifies" --> ops_team
```

---

#### **Physical View (AWS Deployment Diagram)**

The physical diagram adds the new serverless components required for reconciliation. An EventBridge rule triggers the Lambda function, which then accesses the necessary resources.

```mermaid
graph TD
    subgraph "AWS Cloud"
        eventbridge["Amazon EventBridge<br>(Scheduled Rule: e.g., 'cron(0 2 * * ? *)')"]

        subgraph "VPC"
            subgraph "Private Subnet (Application)"
                reco_lambda["AWS Lambda Function<br>(Reconciliation Service)"]
            end
            
            subgraph "Private Subnet (Database)"
                rds_ledger["Amazon RDS<br>(LedgerDB)"]
            end
        end

        s3_reports["Amazon S3 Bucket<br>(for Discrepancy Reports)"]
        sns["Amazon SNS Topic<br>(for Notifications)"]
    end

    psp_sftp["External PSP SFTP Server"]

    eventbridge -- "1.Triggers" --> reco_lambda
    reco_lambda -- "2.Fetches file from" --> psp_sftp
    reco_lambda -- "3.Reads from" --> rds_ledger
    reco_lambda -- "4.Writes report to" --> s3_reports
    reco_lambda -- "5.Publishes notification to" --> sns
```

---

#### **Component-to-Resource Mapping Table**

| Logical Component | Physical Resource | Rationale |
| :--- | :--- | :--- |
| **Reconciliation Service** | **AWS Lambda Function** | **Serverless & Event-Driven:** Lambda is the ideal choice for a periodic, scheduled task. It's fully managed, cost-effective, and can be granted specific IAM permissions to securely access other AWS resources. |
| **(Scheduler)** | **Amazon EventBridge Rule** | **Managed & Reliable:** EventBridge provides a simple and highly reliable cron-like scheduler to trigger the Lambda function automatically without any custom code. |
| **(Report Storage)** | **Amazon S3 Bucket** | **Durable & Secure:** S3 is the standard for durable, cost-effective object storage. It is the perfect place to store generated reports for auditing and historical analysis. |
| **(Notification)** | **Amazon SNS Topic** | **Decoupled & Scalable:** SNS provides a publish/subscribe pattern for sending notifications. The finance team can subscribe via email, or we can later add other subscribers (like a Slack integration) without changing the core service. |
| **LedgerDB** | **Amazon RDS for PostgreSQL** | (No change) The reconciliation service will have read-only access to this database. |
| **PSP Settlement System**| **External SFTP Server** | This is a common method for secure, automated file exchange in the financial industry. |
