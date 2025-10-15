

# 🧩 Approval Workflow Engine — Architecture Documentation

## Overview

The **Approval Engine** is the backbone for managing approval workflows across various business modules. It’s designed to handle the entire lifecycle of an approval process—initiation, processing, cancellation, and finalization—in a consistent, auditable, and configuration-driven way. Think of it as a flexible orchestrator that ensures approvals follow the right path while letting individual modules customize behavior as needed.

---

## Design Goals

We built the engine with a few key principles in mind:

| Goal              | Why It Matters                                                                 |
|-------------------|--------------------------------------------------------------------------------|
| **Consistency**   | Ensures approvals work the same way across all modules, reducing surprises.    |
| **Extensibility** | Lets teams customize specific behaviors without rewriting the core engine.     |
| **Decoupling**    | Keeps configuration, persistence, and notifications separate for cleaner code. |
| **Resilience**    | Catches validation errors early to avoid broken or inconsistent workflows.     |
| **Transparency**  | Provides clear hooks for logging, notifications, and custom workflow logic.    |

---

## Core Components

### 1. `ApprovalEngine<TApprovalRecord, TApprovalContext>`

**What it does**:  
This is the main orchestrator. It drives the approval workflow, managing state transitions (`Requested`, `Approved`, `Rejected`, `Cancelled`) and ensuring everything stays on track.

**Key tasks**:
- Validates incoming requests.
- Fetches workflow configurations using `IApprovalWorkflowConfigManager`.
- Delegates business logic to the abstract base class `ApprovalWorkflowBase`.
- Updates statuses and finalizes workflows.
- Fires off notifications via a protected virtual hook.

---

### 2. `ApprovalWorkflowBase<TApprovalRecord, TApprovalContext>`

**What it does**:  
This abstract base class handles the nitty-gritty details specific to a transaction’s domain, like persistence, validation, and state updates. It’s where module-specific logic lives.

**Key methods**:

| Method                          | What It Does                                                            |
|---------------------------------|-------------------------------------------------------------------------|
| `ValidateInitiation`            | Checks if an approval can start. Must be implemented by derived classes. |
| `ValidateApproval`              | Validates business rules before approving. Must be implemented.          |
| `SaveApprovalAction`            | Saves individual approval records. Must be implemented.                 |
| `GetApprovalRecords`            | Fetches the current approval snapshot for a transaction.                |
| `UpdateParentTransactionStatus` | Updates the parent transaction’s workflow status.                       |
| `AfterParentTransactionUpdate`  | Optional hook for post-update actions (e.g., logging or notifications).  |

---

### 3. `IApprovalWorkflowConfigManager`

**What it does**:  
Supplies workflow configurations based on transaction details (e.g., facility, amount, or type). It’s the single source of truth for how a workflow should behave.

**Example method**:
```csharp
IEnumerable<ApprovalWorkflowConfigEntity> GetWorkflowForTransaction(ApprovalWorkflowSearchCriteria criteria);
```

---

## Workflow Flowchart

Here’s how the approval process flows, visualized as a Mermaid diagram:

```mermaid
flowchart TD
    A[Start Approval Request] --> B[Validate Request]
    B -->|Invalid| X[Throw ValidationException]
    B --> C[Retrieve Workflow Configuration]
    C -->|Missing| X
    C --> D[Initiate Approval Records]
    D --> E[Set Parent Status: Requested]
    E --> F[Wait for Approvers]

    F --> G{Approval Action}
    G -->|Approve| H[Check Parallel/Lien Approvals]
    G -->|Reject| J[EndWorkflow(Status=Rejected)]
    G -->|Cancel| K[EndWorkflow(Status=Cancelled)]

    H -->|All Complete| L[EndWorkflow(Status=Approved)]
    H -->|Pending| F
    J --> M[Update Remaining Approvals: Cancelled]
    K --> M
    M --> N[Update Parent Transaction Status]
    L --> N
    N --> O[AfterParentTransactionUpdate()]
    O --> P[Send Notification / Audit]
    P --> Q[End]
```

---

## Finalization Flow

### `EndWorkflow(ApprovalRequestDto<TContext> request, ApprovalStatus finalStatus)`

**What it does**:
Wraps up the workflow by:
- Closing or cancelling any pending approval records.
- Setting the parent transaction’s final state (`Approved`, `Rejected`, or `Cancelled`).
- Triggering the `AfterParentTransactionUpdate()` hook for any follow-up actions.

This method is **private** to the engine to ensure consistent termination logic—no accidental overrides allowed.

---

## Extension Points

The engine provides clear ways to customize behavior without breaking the core workflow:

| Method                         | Access              | Purpose                                                                 |
|--------------------------------|---------------------|-------------------------------------------------------------------------|
| `AfterParentTransactionUpdate` | `protected virtual` | Add custom logic after a transaction status update (e.g., notifications).|
| `ValidateInitiation`           | `abstract`          | Define pre-initiation validation rules specific to your module.         |
| `ValidateApproval`             | `abstract`          | Add module-specific validation before approving.                        |
| `SaveApprovalAction`           | `abstract`          | Handle how approval records are saved for your module.                  |

---

## Error Handling & Validation

The engine catches issues early and throws **business-level** exceptions for clear, user-friendly error messages:

| Scenario                      | Exception Thrown              |
|-------------------------------|-------------------------------|
| Null approval request         | `BusinessValidationException` |
| Missing workflow config       | `BusinessValidationException` |
| Unauthorized approver         | `BusinessValidationException` |
| Attempted self-approval       | `BusinessValidationException` |
| Exceeds approval limit        | `BusinessValidationException` |

These ensure workflows stay recoverable and don’t leave transactions in a bad state.

---

## Design Choices

### ✅ **Why `EndWorkflow` is Private**  
Locking down `EndWorkflow` ensures the engine controls how workflows wrap up, preventing inconsistent states from custom code.

### ✅ **Why Cancellation Stays Inline**  
Cancellation is just another way to end a workflow. Keeping it in `ProcessApproval()` avoids duplicating logic and keeps things cohesive.

### ✅ **Why Hooks Over Events**  
Hooks let derived classes extend behavior cleanly without the complexity of event-driven systems or tight coupling across domains.

---

## Future Improvements

Here are some areas we could enhance down the road:

| Area              | Idea                                                                |
|-------------------|---------------------------------------------------------------------|
| **Notifications** | Add support for async or queued notifications (e.g., email, Slack). |
| **Audit Trail**   | Centralize tracking for every state change to simplify auditing.    |
| **Caching**       | Cache workflow configs by transaction type to boost performance.    |
| **Parallelism**   | Optimize for high-volume parallel approvals.                       |
| **Metrics**       | Track workflow duration and approver load for better insights.      |

---

