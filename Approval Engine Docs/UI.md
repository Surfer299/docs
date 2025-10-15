## **Razor Pages Integration Strategy**

### **1. Extend Existing Detail Page**

**Enhanced Transaction Detail Page (`TransactionDetail.cshtml`):**

```html
@page "{id:int}"
@model TransactionDetailModel

<div class="row">
    <!-- Existing Transaction Details -->
    <div class="col-md-8">
        <partial name="_TransactionDetails" model="Model.Transaction" />
    </div>
    
    <!-- Approval Workflow Panel -->
    <div class="col-md-4">
        <partial name="_ApprovalWorkflow" model="Model.WorkflowState" />
    </div>
</div>

<!-- Approval Actions Modal -->
<partial name="_ApprovalModal" model="Model.CurrentStep" />

<!-- Workflow History Timeline -->
<partial name="_WorkflowTimeline" model="Model.ApprovalHistory" />
```

### **2. Key Partial Views to Create**

**`_ApprovalWorkflow.cshtml`:**
```html
<div class="card approval-panel">
    <div class="card-header">
        <h5>Approval Status: @Model.Status</h5>
        <span class="badge @(Model.Status == "Pending" ? "bg-warning" : "bg-success")">
            @Model.NextDeadline
        </span>
    </div>
    
    <!-- Current Step Progress -->
    <div class="progress mb-3">
        <div class="progress-bar" style="width: @(Model.ProgressPercentage)%"></div>
    </div>
    
    <!-- Action Buttons (Role-based) -->
    @if (Model.CanApprove)
    {
        <div class="btn-group w-100 mb-3">
            <button class="btn btn-success" onclick="showApprovalModal('approve')">
                <i class="fas fa-check"></i> Approve
            </button>
            <button class="btn btn-danger" onclick="showApprovalModal('reject')">
                <i class="fas fa-times"></i> Reject
            </button>
        </div>
    }
    
    <!-- Step Indicators -->
    <div class="step-indicators">
        @foreach (var step in Model.Steps)
        {
            <div class="step @(step.IsCompleted ? "completed" : step.IsCurrent ? "active" : "")">
                @step.Name
            </div>
        }
    </div>
</div>
```

**`_ApprovalModal.cshtml`:**
```html
<div class="modal fade" id="approvalModal">
    <div class="modal-dialog">
        <div class="modal-content">
            <form asp-page-handler="Approve" method="post">
                <input type="hidden" name="stepId" value="@Model.StepId" />
                <input type="hidden" name="decision" id="decisionType" />
                
                <div class="modal-body">
                    <h5 id="modalTitle">Take Action</h5>
                    <div class="form-group">
                        <label>Comments (Required for Reject)</label>
                        <textarea class="form-control" name="comments" rows="3"></textarea>
                    </div>
                    
                    <!-- Dynamic Fields based on Application Type -->
                    @if (Model.ApplicationType == "Loan")
                    {
                        <partial name="_LoanSpecificFields" />
                    }
                </div>
                
                <div class="modal-footer">
                    <button type="button" class="btn btn-secondary" data-bs-dismiss="modal">Cancel</button>
                    <button type="submit" class="btn btn-primary" id="submitBtn">Submit</button>
                </div>
            </form>
        </div>
    </div>
</div>

<script>
function showApprovalModal(decision) {
    $('#decisionType').val(decision);
    $('#modalTitle').text(decision === 'approve' ? 'Approve Request' : 'Reject Request');
    $('#submitBtn').removeClass('btn-primary btn-danger').addClass(decision === 'approve' ? 'btn-primary' : 'btn-danger');
    $('#approvalModal').modal('show');
}
</script>
```

### **3. Page Model Enhancements (`TransactionDetailModel.cs`)**

```csharp
public class TransactionDetailModel : PageModel
{
    private readonly IWorkflowService _workflowService;
    
    [BindProperty]
    public Transaction Transaction { get; set; }
    
    public WorkflowState WorkflowState { get; set; }
    public List<ApprovalStep> ApprovalHistory { get; set; }
    public CurrentApprovalStep CurrentStep { get; set; }
    
    public async Task OnGetAsync(int id)
    {
        Transaction = await _transactionService.GetByIdAsync(id);
        var workflow = await _workflowService.GetWorkflowForTransaction(id);
        
        WorkflowState = new WorkflowState
        {
            Status = workflow.Status,
            ProgressPercentage = CalculateProgress(workflow.Steps),
            Steps = workflow.Steps,
            CanApprove = await CanUserApproveAsync(workflow.CurrentStep),
            NextDeadline = workflow.NextStepDeadline
        };
        
        ApprovalHistory = workflow.ApprovalHistory;
        CurrentStep = workflow.CurrentStep;
    }
    
    public async Task<IActionResult> OnPostApproveAsync(string stepId, string decision, string comments)
    {
        var result = await _workflowService.ProcessApprovalAsync(
            Transaction.Id, stepId, decision, comments, User.Identity.Name);
            
        if (result.IsSuccess)
        {
            TempData["Success"] = "Approval processed successfully";
            return RedirectToPage("./Detail", new { id = Transaction.Id });
        }
        
        ModelState.AddModelError("", result.Error);
        await OnGetAsync(Transaction.Id); // Reload for display
        return Page();
    }
}
```

### **4. Application-Specific Field Partials**

**`_LoanSpecificFields.cshtml`:**
```html
<div class="alert alert-info">
    <h6>Loan-Specific Review</h6>
    <div class="row">
        <div class="col-6">
            <label>Risk Score: @Model.RiskScore</label>
            <div class="progress">
                <div class="progress-bar @(GetRiskColor(Model.RiskScore))" 
                     style="width: @(Model.RiskScore)%"></div>
            </div>
        </div>
        <div class="col-6">
            <label>Collateral Value: $@Model.CollateralValue</label>
            <input type="checkbox" name="verifiedCollateral" /> Verified
        </div>
    </div>
</div>
```

### **5. Real-time Updates with SignalR**

**Add to `_Host.cshtml` or `Site.css`:**
```html
<script src="~/lib/signalr/signalr.min.js"></script>
<script>
const connection = new signalr.HubConnectionBuilder()
    .withUrl("/workflowHub")
    .build();

connection.on("WorkflowUpdated", function (transactionId) {
    if (transactionId == @Model.Transaction.Id) {
        location.reload(); // Or update specific elements
    }
});

connection.start();
</script>
```

### **6. Role-Based Authorization**

**In Page Model:**
```csharp
[Authorize(Roles = "Approver,Admin")]
public async Task<IActionResult> OnPostApproveAsync(...)
{
    // Ensure user can approve this specific step
    if (!await _workflowService.CanUserApproveStepAsync(stepId, User.Identity.Name))
    {
        return Forbid();
    }
    // ... process approval
}
```

### **7. CSS for Workflow Styling**

**`wwwroot/css/workflow.css`:**
```css
.approval-panel {
    border-left: 4px solid #007bff;
    box-shadow: 0 2px 4px rgba(0,0,0,.1);
}

.step-indicators {
    display: flex;
    justify-content: space-between;
    margin-top: 15px;
}

.step {
    flex: 1;
    text-align: center;
    padding: 8px;
    border-radius: 4px;
    background: #f8f9fa;
}

.step.active { background: #007bff; color: white; }
.step.completed { background: #28a745; color: white; }
```

### **8. Progressive Enhancement Strategy**

1. **Phase 1**: Add approval panel to existing detail page
2. **Phase 2**: Modal-based approval with validation
3. **Phase 3**: Real-time updates via SignalR
4. **Phase 4**: Application-specific field rendering
5. **Phase 5**: Bulk actions and delegation features

### **9. Testing Considerations**

```csharp
// Integration tests for approval flow
[Fact]
public async Task ApproveTransaction_ValidUser_Succeeds()
{
    var page = await _client.GetPageAsync<TransactionDetailModel>(transactionId);
    await page.ApproveAsync("step1", "APPROVE", "Looks good");
    
    var result = Assert.IsType<RedirectToPageResult>(page.Response);
    Assert.Equal("/Detail", result.PageName);
    Assert.True(page.TempData["Success"].ToString().Contains("successfully"));
}
```
