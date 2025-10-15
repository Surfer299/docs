# 🚀 Approval Workflow Engine — Developer User Guide

Hey there! This guide is your go-to for using the `ApprovalEngine<TApprovalRecord, TApprovalContext>` to power approval workflows in your module. Whether you’re kicking off approvals, handling actions, or adding custom logic, this engine takes care of the heavy lifting so you can focus on your module’s unique needs. Let’s get you up and running!

## What’s the Approval Engine?

The Approval Engine is a flexible, configuration-driven system for managing approval workflows. It handles the flow of approvals—starting, processing, canceling, and wrapping up—while keeping things consistent across different modules. You’ll extend a base class, plug in your module’s logic, and let the engine orchestrate the rest.

Here’s what you’ll work with:
- **`ApprovalEngine<TApprovalRecord, TApprovalContext>`**: The core engine that runs the workflow.
- **`ApprovalWorkflowBase<TApprovalRecord, TApprovalContext>`**: An abstract class you’ll extend to add your module’s specific rules and persistence.
- **`IApprovalWorkflowConfigManager`**: Supplies the workflow rules (e.g., who approves what, in what order).

## How to Use It

To get started, you’ll:
1. Extend `ApprovalWorkflowBase` to define your module’s approval logic.
2. Set up workflow configurations via `IApprovalWorkflowConfigManager`.
3. Call the engine’s methods to manage approvals.

Let’s walk through each step with an example for a fictional “Expense Approval” module.

---

## Step 1: Extend `ApprovalWorkflowBase`

Create a class that inherits from `ApprovalWorkflowBase<TApprovalRecord, TApprovalContext>`. This is where you define how your module validates approvals, saves records, and updates transactions.

Here’s a sample for expense approvals:

```csharp
public class ExpenseApprovalWorkflow : ApprovalWorkflowBase<ExpenseApprovalRecord, ExpenseContext>
{
    private readonly IExpenseRepository _expenseRepository;
    private readonly INotificationService _notificationService;

    public ExpenseApprovalWorkflow(IExpenseRepository repo, INotificationService notifications)
    {
        _expenseRepository = repo;
        _notificationService = notifications;
    }

    public override async Task ValidateInitiation(ApprovalRequestDto<ExpenseContext> request)
    {
        if (request.Context.Amount <= 0)
            throw new BusinessValidationException("Expense amount must be greater than zero.");
        if (string.IsNullOrEmpty(request.Context.Facility))
            throw new BusinessValidationException("Facility is required.");
        // Add more checks (e.g., user permissions)
        await Task.CompletedTask;
    }

    public override async Task ValidateApproval(ApprovalRequestDto<ExpenseContext> request, ApprovalAction action)
    {
        if (action.ApproverId == request.InitiatorId)
            throw new BusinessValidationException("You can’t approve your own expense!");
        // Check approver roles, limits, etc.
        await Task.CompletedTask;
    }

    public override async Task SaveApprovalAction(ExpenseApprovalRecord record)
    {
        // Save the approval record to your database
        await _expenseRepository.SaveApprovalAsync(record);
    }

    public override async Task<IEnumerable<ExpenseApprovalRecord>> GetApprovalRecords(Guid transactionId)
    {
        // Fetch approval records for the transaction
        return await _expenseRepository.GetApprovalRecordsAsync(transactionId);
    }

    public override async Task UpdateParentTransactionStatus(Guid transactionId, ApprovalStatus status)
    {
        // Update the expense’s status
        await _expenseRepository.UpdateTransactionStatusAsync(transactionId, status);
    }

    protected override async Task AfterParentTransactionUpdate(Guid transactionId, ApprovalStatus status)
    {
        // Optional: Send a notification after status changes
        if (status == ApprovalStatus.Approved)
            await _notificationService.SendAsync(transactionId, "Your expense is approved!");
    }
}
```

### What’s Going On?
- **`ValidateInitiation`**: Ensures the request is valid before starting (e.g., no zero-dollar expenses).
- **`ValidateApproval`**: Checks if an approval action is allowed (e.g., no self-approvals).
- **`SaveApprovalAction`**: Persists approval records to your database.
- **`GetApprovalRecords`**: Retrieves the current approval state for a transaction.
- **`UpdateParentTransactionStatus`**: Syncs the parent transaction (e.g., expense report) with the workflow status.
- **`AfterParentTransactionUpdate`**: A hook for custom actions like notifications or logging.

---

## Step 2: Configure Your Workflow

The engine needs to know who approves what and in what order. That’s where `IApprovalWorkflowConfigManager` comes in. Implement this interface to provide workflow rules based on transaction details (e.g., amount or facility).

Example:

```csharp
public class ExpenseWorkflowConfigManager : IApprovalWorkflowConfigManager
{
    public async Task<IEnumerable<ApprovalWorkflowConfigEntity>> GetWorkflowForTransaction(ApprovalWorkflowSearchCriteria criteria)
    {
        var configs = new List<ApprovalWorkflowConfigEntity>();

        // Example: Require Manager for expenses > $1000, Director for > $5000
        if (criteria.Amount > 1000)
            configs.Add(new ApprovalWorkflowConfigEntity { ApproverRole = "Manager", Order = 1 });
        if (criteria.Amount > 5000)
            configs.Add(new ApprovalWorkflowConfigEntity { ApproverRole = "Director", Order = 2 });

        return configs;
    }
}
```

This sets up a workflow where a $6000 expense needs both a Manager and a Director to approve, in that order.

---

## Step 3: Call the Engine

With your workflow class and config manager ready, use the `ApprovalEngine` to manage approvals. Here’s how to initiate, process, or cancel a workflow.

### Starting an Approval

Kick off a workflow by creating a request and passing it to the engine:

```csharp
var engine = new ApprovalEngine<ExpenseApprovalRecord, ExpenseContext>(
    new ExpenseApprovalWorkflow(new ExpenseRepository(), new NotificationService()),
    new ExpenseWorkflowConfigManager()
);

var request = new ApprovalRequestDto<ExpenseContext>
{
    TransactionId = Guid.NewGuid(),
    InitiatorId = "employee123",
    Context = new ExpenseContext { Amount = 2000, Facility = "HQ" }
};

await engine.InitiateApprovalAsync(request);
```

This:
1. Runs `ValidateInitiation` to check the request.
2. Fetches the workflow config (e.g., Manager approval required).
3. Creates approval records and sets the transaction status to `Requested`.

### Processing an Approval

When an approver acts (approves, rejects, or cancels), call `ProcessApprovalAsync`:

```csharp
var action = new ApprovalAction
{
    ApproverId = "manager456",
    ActionType = ApprovalActionType.Approve,
    ApprovalRecordId = Guid.NewGuid() // ID of the approval record
};

await engine.ProcessApprovalAsync(request, action);
```

This:
1. Validates the action with `ValidateApproval`.
2. Updates the approval record.
3. Checks if all approvals are done and updates the transaction status (e.g., to `Approved`).
4. Calls `AfterParentTransactionUpdate` for any follow-up actions.

### Canceling a Workflow

To cancel a workflow:

```csharp
await engine.ProcessApprovalAsync(request, new ApprovalAction
{
    ApproverId = "employee123",
    ActionType = ApprovalActionType.Cancel
});
```

This marks the workflow as `Cancelled` and updates any pending records.

---

## Dealing with Errors

The engine throws `BusinessValidationException` for issues like:
- Invalid or missing requests.
- Missing workflow configurations.
- Unauthorized actions (e.g., self-approval).
- Approvals exceeding limits.

Handle these in your code for clean error messages:

```csharp
try
{
    await engine.InitiateApprovalAsync(request);
}
catch (BusinessValidationException ex)
{
    Console.WriteLine($"Oops! {ex.Message}");
}
```

---

## Adding Custom Logic

Need to tweak behavior? Use the `AfterParentTransactionUpdate` hook to add custom actions, like sending emails or logging to an audit trail:

```csharp
protected override async Task AfterParentTransactionUpdate(Guid transactionId, ApprovalStatus status)
{
    await _auditService.LogAsync($"Transaction {transactionId} moved to {status}");
}
```

---

## Pro Tips

- **Validate Early**: Use `ValidateInitiation` and `ValidateApproval` to catch issues before they cause trouble.
- **Test Configurations**: Make sure your `IApprovalWorkflowConfigManager` returns the right rules for different scenarios.
- **Use Hooks Wisely**: Leverage `AfterParentTransactionUpdate` for notifications or logging without touching the core workflow.
- **Keep Users Informed**: Catch `BusinessValidationException` and show meaningful messages to users.

---

## Example in Action

Imagine a $2000 expense:
1. You initiate the request, and the engine creates a Manager approval record.
2. The Manager approves, updating the record.
3. Since it’s under $5000, no Director is needed, so the engine sets the status to `Approved` and sends a notification.
4. If someone tries to approve their own expense, the engine throws a `BusinessValidationException`.