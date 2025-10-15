

```html
<!DOCTYPE html>
<html lang="en" data-bs-theme="light">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Approval Workflow - Multi-Level Simulation</title>
    <!-- Bootstrap 5 CSS -->
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/css/bootstrap.min.css" rel="stylesheet" integrity="sha384-QWTKZyjpPEjISv5WaRU9OFeRpok6YctnYmDr5pNlyT2bRjXh0JMhjY6hW+ALEwIH" crossorigin="anonymous">
    <!-- Bootstrap Icons -->
    <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bootstrap-icons@1.11.3/font/bootstrap-icons.min.css">
    <style>
        /* Modern spinner animation */
        .spinner {
            width: 40px;
            height: 40px;
            border: 4px solid transparent;
            border-top-color: #0d6efd;
            border-right-color: #0d6efd;
            border-radius: 50%;
            animation: spin 0.8s ease-in-out infinite, glow 1.5s ease-in-out infinite;
            box-shadow: 0 0 10px rgba(13, 110, 253, 0.5);
        }
        @keyframes spin {
            0% { transform: rotate(0deg); }
            100% { transform: rotate(360deg); }
        }
        @keyframes glow {
            0%, 100% { box-shadow: 0 0 10px rgba(13, 110, 253, 0.5); }
            50% { box-shadow: 0 0 20px rgba(13, 110, 253, 0.8); }
        }
        .spinner-container {
            display: none;
            position: fixed;
            bottom: 1.25rem;
            right: 1.25rem;
            z-index: 1000;
        }
        .fade-in {
            animation: fadeIn 0.3s ease-in;
        }
        @keyframes fadeIn {
            from { opacity: 0; }
            to { opacity: 1; }
        }
        .notification {
            animation: slideIn 0.3s ease-out;
        }
        @keyframes slideIn {
            from { transform: translateY(-20px); opacity: 0; }
            to { transform: translateY(0); opacity: 1; }
        }
        .rotate-180 {
            transform: rotate(180deg);
        }
        /* Icon visibility based on theme */
        #sunIcon { display: none; }
        #moonIcon { display: none; }
        [data-bs-theme="light"] #moonIcon { display: inline-block; }
        [data-bs-theme="dark"] #sunIcon { display: inline-block; }
    </style>
</head>
<body class="font-sans">
    <div class="container-lg px-4 py-5">
        <!-- Dark Mode Toggle -->
        <div class="d-flex justify-content-end mb-4">
            <button id="darkModeToggle" class="btn btn-outline-secondary d-flex align-items-center gap-2">
                <i id="sunIcon" class="bi bi-sun-fill"></i>
                <i id="moonIcon" class="bi bi-moon-stars-fill"></i>
                <span>Toggle Dark Mode</span>
            </button>
        </div>

        <h3 class="fw-bold mb-4">Approval Workflow for Transaction #123e4567-e89b-12d3-a456-426614174000</h3>

        <!-- User Switcher for Testing -->
        <div class="mb-4">
            <label for="userSelect" class="form-label">Simulate User Role:</label>
            <div class="d-flex align-items-center gap-3">
                <select id="userSelect" class="form-select form-select-sm w-auto">
                    <option value="manager-initiator">Manager (Initiator)</option>
                    <option value="director">Director</option>
                    <option value="cfo">CFO</option>
                    <option value="non-approver">Non-Approver</option>
                </select>
                <button type="button" class="btn btn-secondary" onclick="switchUser()">Switch User</button>
            </div>
        </div>

        <!-- Notification -->
        <div id="notification" class="alert alert-success border-start border-success border-4 notification d-none" role="alert">
            <div class="d-flex justify-content-between align-items-center">
                <span id="notificationText"></span>
                <button type="button" class="btn-close" onclick="this.parentElement.parentElement.classList.add('d-none')"></button>
            </div>
        </div>

        <!-- Loading Spinner -->
        <div class="spinner-container" id="spinnerContainer">
            <div class="spinner"></div>
        </div>

        <!-- Transaction Details -->
        <div class="card bg-body shadow mb-4 fade-in">
            <div class="card-header">
                <h4 class="mb-0">Transaction Details</h4>
            </div>
            <div class="card-body">
                <p><strong>Amount:</strong> $5,000.00</p>
                <p><strong>Facility:</strong> Main Branch</p>
                <p><strong>Initiator:</strong> <span id="initiatorName">John Doe</span></p>
                <p>
                    <strong>Status:</strong>
                    <span id="statusBadge" class="badge bg-warning">Pending</span>
                </p>
            </div>
        </div>

        <!-- Pending Approvals -->
        <div id="pendingApprovalsCard" class="card bg-body shadow mb-4 d-none fade-in">
            <div class="card-header">
                <h4 class="mb-0">Pending Approvals</h4>
            </div>
            <div class="card-body">
                <table class="table table-sm">
                    <thead>
                        <tr class="text-muted">
                            <th>Approver Role</th>
                            <th>Order</th>
                            <th>Status</th>
                            <th>Action</th>
                        </tr>
                    </thead>
                    <tbody id="pendingTableBody"></tbody>
                </table>
            </div>
        </div>

        <!-- Cancel Button -->
        <div id="cancel-button-container" class="mb-4"></div>

        <!-- Approval History -->
        <div class="card bg-body shadow mb-4 fade-in">
            <div class="card-header">
                <h4 class="mb-0">
                    <button class="btn btn-link text-start p-0 text-decoration-none" data-bs-toggle="collapse" data-bs-target="#approvalHistory" onclick="toggleCollapse('historyToggleIcon')">
                        Approval History
                        <svg class="inline ms-1" width="12" height="12" id="historyToggleIcon" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19 9l-7 7-7-7"></path></svg>
                    </button>
                </h4>
            </div>
            <div id="approvalHistory" class="collapse show">
                <div class="card-body">
                    <table class="table table-sm">
                        <thead>
                            <tr class="text-muted">
                                <th>Approver</th>
                                <th>Action</th>
                                <th>Date</th>
                            </tr>
                        </thead>
                        <tbody id="historyTableBody"></tbody>
                    </table>
                </div>
            </div>
        </div>

        <!-- Completed Approvals -->
        <div id="completedApprovalsCard" class="card bg-body shadow mb-4 d-none fade-in">
            <div class="card-header">
                <h4 class="mb-0">Completed Workflow</h4>
            </div>
            <div class="card-body">
                <p class="text-success fw-medium">All approvals have been processed. Transaction is <strong>Approved</strong>.</p>
            </div>
        </div>
    </div>

    <!-- Bootstrap JS Bundle -->
    <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/js/bootstrap.bundle.min.js" integrity="sha384-YvpcrYf0tY3lHB60NNkmXc5s9fDVZLESaAA55NDzOxhy9GkcIdslK1eN7N6jIeHz" crossorigin="anonymous"></script>
    <script>
        // Dark Mode Toggle
        document.getElementById('darkModeToggle').addEventListener('click', () => {
            const root = document.documentElement;
            const currentTheme = root.getAttribute('data-bs-theme');
            root.setAttribute('data-bs-theme', currentTheme === 'dark' ? 'light' : 'dark');
        });

        // Workflow State Simulation
        let workflowState = {
            transactionId: "123e4567-e89b-12d3-a456-426614174000",
            status: "Pending",
            currentLevel: 1,
            approvals: [
                { id: 1, approverRole: "Manager", order: 1, status: "Pending" },
                { id: 2, approverRole: "Director", order: 2, status: "Pending" },
                { id: 3, approverRole: "CFO", order: 3, status: "Pending" }
            ],
            history: [
                { approverName: "John Doe", actionType: "Initiated", timestamp: new Date().toLocaleString() }
            ],
            initiatorName: "John Doe"
        };

        let currentUser = {
            name: "Alice Manager",
            roles: ["Manager"],
            isInitiator: true
        };

        function showSpinner() {
            document.getElementById("spinnerContainer").style.display = "block";
        }

        function hideSpinner() {
            document.getElementById("spinnerContainer").style.display = "none";
        }

        function performAction(action, approvalId) {
            showSpinner();
            setTimeout(() => {
                hideSpinner();
                if (action === "Cancel Workflow") {
                    workflowState.status = "Cancelled";
                    workflowState.history.push({
                        approverName: currentUser.name,
                        actionType: "Cancelled",
                        timestamp: new Date().toLocaleString()
                    });
                    updateNotification("Workflow has been cancelled by the initiator.");
                } else {
                    const approval = workflowState.approvals.find(a => a.id === approvalId);
                    if (approval) {
                        approval.status = action;
                        workflowState.history.push({
                            approverName: currentUser.name,
                            actionType: action,
                            timestamp: new Date().toLocaleString()
                        });
                        if (action === "Approve") {
                            workflowState.currentLevel++;
                            if (workflowState.currentLevel > workflowState.approvals.length) {
                                workflowState.status = "Approved";
                                updateNotification("All approvals completed. Transaction is approved.");
                                document.getElementById("pendingApprovalsCard").classList.add("d-none");
                                document.getElementById("completedApprovalsCard").classList.remove("d-none");
                            } else {
                                updateNotification(`Approved by ${approval.approverRole}. Now pending ${workflowState.approvals[workflowState.currentLevel - 1].approverRole}.`);
                            }
                        } else {
                            workflowState.status = "Rejected";
                            updateNotification(`Rejected by ${approval.approverRole}. Workflow stopped.`);
                            setTimeout(() => {
                                document.getElementById("pendingApprovalsCard").classList.add("d-none");
                            }, 2000);
                        }
                    }
                }
                updateUI();
                alert(`${action} completed for approval ID ${approvalId || 'workflow'}. Check the history and status.`);
            }, 1500);
        }

        function canUserAct(approval) {
            return currentUser.roles.includes(approval.approverRole) && 
                   approval.order === workflowState.currentLevel && 
                   approval.status === "Pending" &&
                   workflowState.status === "Pending";
        }

        function renderPendingApprovals() {
            const tbody = document.getElementById("pendingTableBody");
            tbody.innerHTML = "";
            workflowState.approvals.forEach(approval => {
                const row = tbody.insertRow();
                const badgeClass = approval.status === 'Pending' ? 'bg-warning' : 
                                   approval.status === 'Approved' ? 'bg-success' : 'bg-danger';
                const badgeTextClass = approval.status === 'Pending' ? 'text-dark' : '';
                row.innerHTML = `
                    <td>${approval.approverRole}</td>
                    <td>${approval.order}</td>
                    <td><span class="badge ${badgeClass} ${badgeTextClass}">${approval.status}</span></td>
                    <td id="action-${approval.id}">
                        ${canUserAct(approval) ? `
                            <button type="button" class="btn btn-success btn-sm me-2" onclick="performAction('Approve', ${approval.id})">Approve</button>
                            <button type="button" class="btn btn-danger btn-sm" onclick="performAction('Reject', ${approval.id})">Reject</button>
                        ` : ''}
                    </td>
                `;
            });
            const card = document.getElementById("pendingApprovalsCard");
            card.classList.toggle("d-none", workflowState.status !== "Pending");
        }

        function renderCancelButton() {
            const container = document.getElementById("cancel-button-container");
            if (currentUser.isInitiator && workflowState.status === "Pending") {
                container.innerHTML = `
                    <button type="button" class="btn btn-warning" onclick="performAction('Cancel Workflow', null)">Cancel Workflow</button>
                `;
            } else {
                container.innerHTML = "";
            }
        }

        function renderHistory() {
            const tbody = document.getElementById("historyTableBody");
            tbody.innerHTML = "";
            workflowState.history.forEach(entry => {
                const row = tbody.insertRow();
                row.innerHTML = `
                    <td>${entry.approverName}</td>
                    <td>${entry.actionType}</td>
                    <td>${entry.timestamp}</td>
                `;
            });
        }

        function updateStatusBadge() {
            const badge = document.getElementById("statusBadge");
            badge.textContent = workflowState.status;
            const badgeClass = workflowState.status === "Pending" ? "bg-warning" :
                               workflowState.status === "Approved" ? "bg-success" :
                               workflowState.status === "Rejected" ? "bg-danger" : 
                               "bg-secondary";
            const textClass = workflowState.status === "Pending" ? "text-dark" : "";
            badge.className = `badge ${badgeClass} ${textClass}`;
        }

        function updateNotification(message) {
            const alert = document.getElementById("notification");
            document.getElementById("notificationText").textContent = message;
            alert.classList.remove("d-none");
            setTimeout(() => alert.classList.add("d-none"), 5000);
        }

        function toggleCollapse(iconId) {
            const icon = document.getElementById(iconId);
            icon.classList.toggle("rotate-180");
        }

        function updateUI() {
            updateStatusBadge();
            renderPendingApprovals();
            renderCancelButton();
            renderHistory();
        }

        function switchUser() {
            const select = document.getElementById("userSelect");
            const value = select.value;
            switch (value) {
                case "manager-initiator":
                    currentUser = { name: "Alice Manager", roles: ["Manager"], isInitiator: true };
                    break;
                case "director":
                    currentUser = { name: "Bob Director", roles: ["Director"], isInitiator: false };
                    break;
                case "cfo":
                    currentUser = { name: "Carol CFO", roles: ["CFO"], isInitiator: false };
                    break;
                case "non-approver":
                    currentUser = { name: "Dave Viewer", roles: [], isInitiator: false };
                    break;
            }
            document.getElementById("initiatorName").textContent = workflowState.initiatorName;
            updateUI();
            updateNotification(`Switched to user: ${currentUser.name}`);
        }

        document.addEventListener("DOMContentLoaded", () => {
            updateNotification("Transaction is awaiting approval from Manager.");
            updateUI();
        });
    </script>
</body>
</html>
```

### Changes Made
- **Down Arrow Size**:
  - Changed SVG in "Approval History" to `width="12" height="12"`.
  - Adjusted margin with `ms-1` (4px) to align with text.
- **JavaScript**:
  - Kept vanilla JavaScript for now, as the DOM manipulations are straightforward (e.g., `classList.toggle`, `innerHTML`).
  - No need for jQuery unless you have specific reasons (e.g., familiarity, existing jQuery codebase).

### If You Want jQuery
If you prefer jQuery for DOM manipulation or event handling, I can rewrite the script. Here's a quick example of how `switchUser` would look with jQuery:

```javascript
function switchUser() {
    const value = $("#userSelect").val();
    switch (value) {
        case "manager-initiator":
            currentUser = { name: "Alice Manager", roles: ["Manager"], isInitiator: true };
            break;
        case "director":
            currentUser = { name: "Bob Director", roles: ["Director"], isInitiator: false };
            break;
        case "cfo":
            currentUser = { name: "Carol CFO", roles: ["CFO"], isInitiator: false };
            break;
        case "non-approver":
            currentUser = { name: "Dave Viewer", roles: [], isInitiator: false };
            break;
    }
    $("#initiatorName").text(workflowState.initiatorName);
    updateUI();
    updateNotification(`Switched to user: ${currentUser.name}`);
}
```

To use jQuery, add this to the `<head>`:
```html
<script src="https://code.jquery.com/jquery-3.7.1.min.js" integrity="sha256-/JqT3SQfawRcv/BIHPThkBvs0OEvtFFmqPF/lYI/Cxo=" crossorigin="anonymous"></script>
```

