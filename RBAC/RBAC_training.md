# RBAC in Grafana Cloud — Training Module

**Module:** GC-101
**Duration:** 45 minutes
**Level:** Beginner to Intermediate
**Audience:** All Grafana Cloud users, administrators, and team leads

---

## 📋 Module Overview

```
By the end of this module you will be able to:

  ✅ Explain what RBAC is and why it matters
  ✅ Describe the Grafana Cloud role hierarchy
  ✅ Differentiate between Basic and Fixed roles
  ✅ Create and assign Custom roles
  ✅ Apply RBAC to Teams and Service Accounts
  ✅ Follow least-privilege best practices
  ✅ Audit and review access in Grafana Cloud
```

---

## 📚 Table of Contents

1. [What is RBAC?](#what)
2. [Why RBAC Matters](#why)
3. [Grafana Cloud Role Hierarchy](#hierarchy)
4. [Basic Roles](#basic)
5. [Fixed Roles](#fixed)
6. [Custom Roles](#custom)
7. [Assigning Roles](#assigning)
8. [Teams & RBAC](#teams)
9. [Service Account RBAC](#serviceaccounts)
10. [Least Privilege in Practice](#leastprivilege)
11. [Auditing Access](#auditing)
12. [Knowledge Check](#quiz)
13. [Summary & Next Steps](#summary)

---

<a name="what"></a>
## 1. 🤔 What is RBAC?

### Definition

**Role-Based Access Control (RBAC)** is a security model that controls what actions users can perform in a system based on the **roles** they have been assigned rather than granting permissions directly to individual users.

```
WITHOUT RBAC                        WITH RBAC
────────────────────────────────────────────────────────────

User A → Permission 1               Role: Editor
User A → Permission 2                  └── Permission 1
User A → Permission 3                  └── Permission 2
User B → Permission 1                  └── Permission 3
User B → Permission 2
User C → Permission 1               User A → Editor role
User C → Permission 3               User B → Editor role
                                    User C → Viewer role

Managing 3 users separately →       Managing 2 roles
Messy, error-prone, unscalable      Clean, consistent, scalable
```

### Core RBAC Concepts

```
┌─────────────────────────────────────────────────────────┐
│                   RBAC Building Blocks                   │
│                                                          │
│  PERMISSION                                              │
│  └── The ability to perform a specific action           │
│      Example: dashboards:read, alerts:write             │
│                                                          │
│  ROLE                                                    │
│  └── A named collection of permissions                  │
│      Example: Viewer = read dashboards + read alerts    │
│                                                          │
│  ASSIGNMENT                                              │
│  └── Linking a role to a user, team, or                │
│      service account                                    │
│      Example: Alice → Editor role                       │
└─────────────────────────────────────────────────────────┘
```

### How Grafana Implements RBAC

Grafana Cloud uses a **layered RBAC system**:

```
Layer 1: Basic Roles
  Pre-built, coarse-grained roles
  (Viewer, Editor, Admin)
  └── Good for simple organisations

Layer 2: Fixed Roles
  Pre-built, fine-grained roles
  (Alerting Editor, Dashboard Viewer, etc.)
  └── Good for specialised access needs

Layer 3: Custom Roles
  Organisation-defined roles
  Built from individual permissions
  └── Good for precise access control
```

---

<a name="why"></a>
## 2. 🎯 Why RBAC Matters

### The Problem Without RBAC

```
Scenario: You give everyone Admin access "temporarily"

Week 1:   New engineer needs to view dashboards
          → Given Admin access "just to get started"

Week 4:   Contractor needs to check some metrics
          → Given Admin access "it's easier"

Month 3:  Junior analyst needs to read reports
          → Given Admin access "we'll change it later"

Result:
  ❌ Everyone has Admin access
  ❌ Anyone can delete dashboards
  ❌ Anyone can modify alert rules
  ❌ Anyone can create API keys
  ❌ No audit trail of who changed what
  ❌ Compliance audit fails
```

### The Benefits of Proper RBAC

```
Security Benefits
  ✅ Limits blast radius of compromised accounts
  ✅ Prevents accidental deletion of dashboards
  ✅ Restricts who can modify production alerts
  ✅ Controls access to sensitive datasources

Operational Benefits
  ✅ Clear ownership and accountability
  ✅ Consistent access across the organisation
  ✅ Easier onboarding and offboarding
  ✅ Self-service for common access requests

Compliance Benefits
  ✅ Demonstrates access control for audits
  ✅ Satisfies SOX, PCI-DSS, ISO 27001 requirements
  ✅ Provides audit trail of access changes
  ✅ Supports principle of least privilege
```

### Real-World RBAC Example

```
Financial Organisation — Grafana Cloud Access Model

Role                Who Has It          What They Can Do
────────────────────────────────────────────────────────────
Org Admin           2 platform leads    Everything
Editor              12 senior engineers Create/edit dashboards
Viewer              150 stakeholders    Read dashboards only
Alerting Editor     8 SRE engineers     Manage alert rules
Datasource Reader   25 analysts         Query datasources
Custom: FinanceViewer 30 finance users  Finance dashboards only
```

---

<a name="hierarchy"></a>
## 3. 🏗️ Grafana Cloud Role Hierarchy

### Overview

```
┌─────────────────────────────────────────────────────────────┐
│                  Grafana Cloud RBAC Hierarchy                │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                   Org Admin                          │    │
│  │   Full control over the Grafana organisation        │    │
│  └─────────────────────────────────────────────────────┘    │
│                          ▲                                   │
│                          │ includes all permissions of       │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                    Editor                            │    │
│  │   Create, edit, delete dashboards and datasources   │    │
│  └─────────────────────────────────────────────────────┘    │
│                          ▲                                   │
│                          │ includes all permissions of       │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                    Viewer                            │    │
│  │   Read-only access to dashboards and data           │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
│  These are your BASIC ROLES — the foundation                │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │              Fixed Roles (add-ons)                   │    │
│  │   Grafana-managed fine-grained permission sets      │    │
│  │   e.g. Alerting Editor, Dashboard Admin             │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │              Custom Roles (add-ons)                  │    │
│  │   Organisation-defined permission sets              │    │
│  │   e.g. FinanceViewer, SREOnCallEngineer             │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

### Important: Roles Are Additive

```
Key Concept: Roles ADD permissions, they do not subtract them

Example:
  User has:  Viewer role (read dashboards)
             + Fixed role: Alerting Editor (edit alerts)

  Result:    User can READ dashboards AND EDIT alerts
             They cannot edit dashboards (no Editor basic role)

  You cannot use a role to REMOVE a permission
  granted by another role.
```

---

<a name="basic"></a>
## 4. 👤 Basic Roles

### The Three Basic Roles

Basic roles are the **starting point** for every user. Every user must have exactly one basic role.

---

#### 🔵 Viewer

```
Who should have this role:
  • Business stakeholders
  • Executives viewing reports
  • External contractors (read-only)
  • Junior team members
  • Audit users

What they CAN do:
  ✅ View dashboards
  ✅ View panels and data
  ✅ View alert rules (cannot modify)
  ✅ View datasources (cannot query Explore)
  ✅ View annotations
  ✅ View playlists

What they CANNOT do:
  ❌ Create or edit dashboards
  ❌ Create or edit alert rules
  ❌ Access the Explore interface
  ❌ Manage datasources
  ❌ Manage users or teams
  ❌ Access administration settings
```

---

#### 🟡 Editor

```
Who should have this role:
  • Application developers
  • Data analysts building dashboards
  • SRE engineers
  • Team leads

What they CAN do:
  ✅ Everything Viewer can do
  ✅ Create and edit dashboards
  ✅ Delete their own dashboards
  ✅ Create and edit alert rules
  ✅ Use the Explore interface
  ✅ Create and manage playlists
  ✅ Create annotations
  ✅ Access dashboard folders they have permission for

What they CANNOT do:
  ❌ Manage datasources (add/edit/delete)
  ❌ Manage users and teams
  ❌ Access server administration
  ❌ Manage API keys
  ❌ Delete other users' dashboards (without folder permission)
```

---

#### 🔴 Org Admin

```
Who should have this role:
  • Platform engineering leads (2-3 people maximum)
  • Grafana Cloud administrators
  • Security team leads (read-only sub-role preferred)

What they CAN do:
  ✅ Everything Editor can do
  ✅ Manage datasources
  ✅ Manage users and teams
  ✅ Manage API keys and service accounts
  ✅ Manage organisation settings
  ✅ Install and manage plugins
  ✅ View server statistics
  ✅ Manage RBAC roles and assignments
  ✅ Access all folders and dashboards

What they CANNOT do:
  ❌ Access Grafana Cloud billing
     (that is a Grafana Cloud account level setting)
  ❌ Manage other Grafana Cloud stacks
```

---

### Basic Role Comparison Table

| Capability | Viewer | Editor | Admin |
|------------|--------|--------|-------|
| View dashboards | ✅ | ✅ | ✅ |
| Create dashboards | ❌ | ✅ | ✅ |
| Edit dashboards | ❌ | ✅ | ✅ |
| Delete dashboards | ❌ | ✅ | ✅ |
| Use Explore | ❌ | ✅ | ✅ |
| Create alert rules | ❌ | ✅ | ✅ |
| Manage datasources | ❌ | ❌ | ✅ |
| Manage users | ❌ | ❌ | ✅ |
| Manage teams | ❌ | ❌ | ✅ |
| Manage API keys | ❌ | ❌ | ✅ |
| Manage RBAC | ❌ | ❌ | ✅ |

---

### 🧠 Knowledge Check — Basic Roles

```
Question 1:
  A business analyst needs to view Grafana dashboards
  to monitor KPIs but should never be able to modify
  anything. Which basic role should they receive?

  A) Org Admin
  B) Editor
  C) Viewer ✅
  D) No role

─────────────────────────────────────────────────────────────

Question 2:
  A developer needs to build new dashboards for their
  team and write alert rules. Which basic role is
  the minimum required?

  A) Viewer
  B) Editor ✅
  C) Org Admin
  D) Custom Role

─────────────────────────────────────────────────────────────

Question 3:
  True or False: You should give all engineers
  Org Admin access to make onboarding easier.

  FALSE ✅
  Reason: Org Admin provides full access including
  user management and datasource configuration.
  Use Editor for engineers who need to build dashboards.
```

---

<a name="fixed"></a>
## 5. 🔧 Fixed Roles

### What Are Fixed Roles?

Fixed roles are **pre-built, fine-grained roles** provided by Grafana. They give you more precise control than Basic roles alone without needing to build custom roles from scratch.

```
Think of Fixed Roles as:
  Specialist add-ons to a Basic Role

Example:
  Engineer has: Editor (basic role)
              + Alerting Manager (fixed role)

  This means they can edit dashboards (Editor)
  AND manage notification policies (Alerting Manager)
  without having full Admin access.
```

---

### Key Fixed Roles Reference

#### Alerting Fixed Roles

```
Fixed Role                  What It Allows
────────────────────────────────────────────────────────────
Alerting Reader             View alert rules, instances,
                            silences, and contact points

Alerting Writer             Create and edit alert rules
                            and silences

Alerting Admin              Full alerting management
                            including notification policies
                            and contact points

Alerting Provisioner        Used for automated provisioning
                            of alerting resources
```

#### Dashboard Fixed Roles

```
Fixed Role                  What It Allows
────────────────────────────────────────────────────────────
Dashboards Reader           View all dashboards
                            (overrides folder restrictions)

Dashboards Writer           Create, edit, and delete
                            all dashboards

Dashboards Admin            Full dashboard management
                            including permissions
```

#### Datasource Fixed Roles

```
Fixed Role                  What It Allows
────────────────────────────────────────────────────────────
Data Source Reader          View datasource configuration
                            (not query data)

Data Source Writer          Create and edit datasources

Data Source Admin           Full datasource management
                            including permissions

Data Source Explorer        Use Explore with this datasource
                            (without full Editor access)
```

#### User & Team Fixed Roles

```
Fixed Role                  What It Allows
────────────────────────────────────────────────────────────
Users Reader                View users in the organisation

Users Writer                Invite and manage users

Teams Reader                View teams and membership

Teams Writer                Create and manage teams
```

---

### Combining Basic and Fixed Roles

```
Practical Examples:
────────────────────────────────────────────────────────────

On-Call Engineer
  Basic Role:   Viewer (read dashboards)
  Fixed Role:   Alerting Writer (manage silences)
  Result:       Can view dashboards and silence alerts
                during incidents

Security Auditor
  Basic Role:   Viewer
  Fixed Role:   Users Reader + Data Source Reader
  Result:       Can audit users and datasource configs
                without being able to change anything

Data Platform Engineer
  Basic Role:   Editor (build dashboards)
  Fixed Role:   Data Source Writer (manage datasources)
  Result:       Can build dashboards AND manage the
                datasources they query — without Admin

Junior SRE
  Basic Role:   Viewer
  Fixed Role:   Alerting Reader
  Result:       Can view dashboards and alert rules
                but cannot modify anything
```

---

<a name="custom"></a>
## 6. 🛠️ Custom Roles

### What Are Custom Roles?

Custom roles allow you to define **exactly the permissions** a role should have using individual permission strings. They are ideal when Basic and Fixed roles do not precisely match your requirements.

```
When to use Custom Roles:
  ✅ You need access to specific dashboards only
     (e.g. Finance team sees only finance dashboards)
  ✅ You need a combination not covered by Fixed roles
  ✅ You need to comply with strict least-privilege policies
  ✅ You have regulatory requirements for precise access
```

---

### Permission Format

```
Grafana permissions follow this format:

  <resource>:<action>

Examples:
  dashboards:read         → Read all dashboards
  dashboards:write        → Create/edit dashboards
  dashboards:delete       → Delete dashboards
  alert.rules:read        → Read alert rules
  alert.rules:write       → Write alert rules
  datasources:read        → Read datasource config
  datasources:query       → Query a datasource
  users:read              → Read user list
  teams:read              → Read team list
  annotations:read        → Read annotations
  annotations:write       → Write annotations
  folders:read            → Read folder contents
  folders:write           → Create/edit folders
  org.users:add           → Add users to org
  serviceaccounts:read    → Read service accounts
```

---

### Creating a Custom Role — Step by Step

#### Via the Grafana Cloud UI

```
Navigation:
Administration → Users and access → Roles → Create role

Step 1: Name your role
  Name:        finance-dashboard-viewer
  Description: Read-only access to Finance folder only
  Version:     1

Step 2: Add permissions
  Click "Add permission"
  Select resource: Dashboards
  Select action:   Read

  Click "Add permission" again
  Select resource: Folders
  Select action:   Read

Step 3: Scope permissions (optional)
  Scope to specific folder:
  folders:uid:finance-folder-uid

Step 4: Save the role

Step 5: Assign to users or teams
  Administration → Users and access → Users
  Select user → Edit role → Add role → finance-dashboard-viewer
```

---

#### Via the Grafana API

```bash
# Step 1 — Create the custom role
curl -X POST \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <admin-token>" \
  https://your-stack.grafana.net/api/access-control/roles \
  -d '{
    "name": "finance-dashboard-viewer",
    "description": "Read-only access to Finance dashboards",
    "version": 1,
    "global": false,
    "permissions": [
      {
        "action": "dashboards:read",
        "scope": "folders:uid:finance-folder-uid"
      },
      {
        "action": "folders:read",
        "scope": "folders:uid:finance-folder-uid"
      },
      {
        "action": "annotations:read",
        "scope": "dashboards:*"
      }
    ]
  }'

# Step 2 — Assign to a user
curl -X POST \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <admin-token>" \
  https://your-stack.grafana.net/api/access-control/users/42/roles \
  -d '{
    "roleUID": "finance-dashboard-viewer",
    "global": false
  }'
```

---

### Custom Role Examples for Common Use Cases

```
Role: sre-oncall-engineer
  Purpose: On-call engineers during incidents
  Permissions:
    dashboards:read         (view dashboards)
    alert.rules:read        (view alert rules)
    alert.instances:read    (view firing alerts)
    alert.instances:create  (create silences)
    annotations:read        (view annotations)
    datasources:query       (query data in Explore)

─────────────────────────────────────────────────────────────

Role: team-dashboard-editor
  Purpose: Edit dashboards within team folder only
  Permissions:
    dashboards:read   [scope: folders:uid:<team-folder>]
    dashboards:write  [scope: folders:uid:<team-folder>]
    folders:read      [scope: folders:uid:<team-folder>]
    datasources:read  (view available datasources)

─────────────────────────────────────────────────────────────

Role: external-auditor
  Purpose: Read-only audit access
  Permissions:
    dashboards:read    (view dashboards)
    users:read         (view user list)
    teams:read         (view teams)
    folders:read       (view folder structure)
    datasources:read   (view datasource config)
    audit:read         (view audit logs)
    — No write permissions of any kind —
```

---

<a name="assigning"></a>
## 7. 👥 Assigning Roles

### Assigning Roles to Individual Users

```
Navigation:
Administration → Users and access → Users
Select a user → Edit

Available options:
  Organisation role:   Viewer | Editor | Admin
                       (Basic role — one per user)

  Additional roles:    Search and add Fixed or Custom roles
                       (Can have multiple)
```

---

### Role Assignment Best Practices

```
✅ DO
  Assign the MINIMUM basic role required
  Add fixed or custom roles to extend where needed
  Document why each role was assigned
  Review assignments quarterly
  Use teams for group assignments (not individual)

❌ DO NOT
  Give Admin to users who only need Editor
  Give Editor to users who only need Viewer
  Assign roles "temporarily" without a review date
  Share accounts between multiple people
  Give roles without understanding what they provide
```

---

### Role Assignment Lifecycle

```
JOINER
  Day 1:   Manager submits access request
  Day 1:   Admin assigns minimum required role
  Day 1:   User added to relevant team(s)
  Day 1:   Welcome email with access confirmation

MOVER (role change)
  Request: Manager submits change request
  Action:  Remove old role(s) first
  Action:  Assign new role(s)
  Note:    Never add before removing (brief overlap OK)

LEAVER
  Day 0:   HR notifies platform team
  Day 0:   Account disabled immediately
  Day 30:  Account reviewed for deletion
  Action:  Dashboard ownership transferred to team
```

---

<a name="teams"></a>
## 8. 👨‍👩‍👧‍👦 Teams & RBAC

### Why Use Teams?

```
WITHOUT Teams (managing 20 users individually)
  User 1  → Payments Folder: Editor
  User 2  → Payments Folder: Editor
  User 3  → Payments Folder: Editor
  ...
  User 20 → Payments Folder: Editor

  New user joins → Remember to add them manually
  User leaves → Remember to remove them manually
  = High maintenance, prone to errors

WITH Teams
  Team: Payments Engineers
    → Payments Folder: Editor role

  User 1  → Team: Payments Engineers
  User 2  → Team: Payments Engineers
  ...
  User 20 → Team: Payments Engineers

  New user joins → Add to team → Gets all permissions
  User leaves → Remove from team → Loses all permissions
  = Low maintenance, consistent, scalable
```

---

### Creating and Configuring a Team

```
Navigation:
Administration → Users and access → Teams → New team

Step 1: Create the team
  Name:     payments-engineers
  Email:    payments-team@company.com (optional)

Step 2: Add team members
  Members tab → Add member → Search and select users

Step 3: Assign team roles
  Roles tab → Add role
  Add: Editor (basic role for the team)
  Add: Alerting Writer (fixed role)

Step 4: Set folder permissions
  Navigate to the folder:
  Dashboards → Browse → Payments folder
  → Permissions → Add permission
  → Team: payments-engineers → Editor
```

---

### Team RBAC Model Example

```
Grafana Cloud Organisation
│
├── Team: Platform SRE
│   ├── Members: alice, bob, charlie
│   ├── Org Role: Editor
│   ├── Fixed Role: Alerting Admin
│   ├── Fixed Role: Data Source Writer
│   └── Folder Access: All folders (Admin)
│
├── Team: Payments Engineers
│   ├── Members: diana, eve, frank
│   ├── Org Role: Editor
│   ├── Fixed Role: Alerting Writer
│   └── Folder Access: /payments/ (Editor)
│
├── Team: Finance Viewers
│   ├── Members: grace, henry, iris
│   ├── Org Role: Viewer
│   └── Folder Access: /finance/ (Viewer)
│
└── Team: External Auditors
    ├── Members: auditor-1, auditor-2
    ├── Org Role: Viewer
    └── Folder Access: /compliance/ (Viewer)
```

---

<a name="serviceaccounts"></a>
## 9. 🤖 Service Account RBAC

### What Are Service Accounts?

Service accounts are **non-human identities** used by applications, scripts, and automation tools to interact with Grafana Cloud. They use tokens instead of usernames and passwords.

```
When to use Service Accounts:
  ✅ CI/CD pipelines deploying dashboards
  ✅ Monitoring tools querying Grafana API
  ✅ Alloy/Agent sending telemetry
  ✅ Terraform managing Grafana resources
  ✅ Custom applications embedding dashboards
  ✅ Alerting integrations (PagerDuty, Slack bots)
```

---

### Service Account RBAC Principles

```
Rule 1: One service account per use case
  ✅ sa-cicd-dashboard-deploy
  ✅ sa-alloy-metrics-write
  ✅ sa-terraform-manage
  ❌ sa-shared-everything  ← Never share

Rule 2: Minimum required permissions
  CI/CD deploying dashboards needs:
    dashboards:write + folders:read
    NOT Admin role

  Alloy writing metrics needs:
    metrics:write
    NOT Editor or Admin

Rule 3: Rotate tokens on schedule
  Rotate every 90 days
  Set expiry when creating tokens
  Alert when expiry is within 7 days

Rule 4: Name tokens descriptively
  sa-cicd-deploy-prod-token-2024-q1
  NOT: token1, mytoken, test
```

---

### Creating a Service Account

```
Navigation:
Administration → Users and access → Service accounts
→ Add service account

Step 1: Create service account
  Display name:  sa-cicd-dashboard-deploy
  Role:          Viewer (start low, add permissions)

Step 2: Assign appropriate role
  Add Fixed Role: Dashboards Writer
  Add Fixed Role: Folders Reader
  Result: Can deploy dashboards, cannot do anything else

Step 3: Generate token
  Click "Add service account token"
  Name:    production-token-2024-q1
  Expiry:  90 days

Step 4: Store token securely
  Save to: HashiCorp Vault / AWS Secrets Manager
  Never:   Hardcode in scripts or commit to Git

Step 5: Set expiry alert
  Alert when token expires in < 7 days
```

---

<a name="leastprivilege"></a>
## 10. 🔒 Least Privilege in Practice

### The Principle of Least Privilege

```
Definition:
  Every user, team, and service account should have
  the MINIMUM access required to do their job
  — nothing more, nothing less.

Why it matters:
  ✅ Limits damage from compromised accounts
  ✅ Reduces risk of accidental misconfiguration
  ✅ Satisfies compliance requirements
  ✅ Creates clear operational boundaries
```

---

### Least Privilege Decision Framework

```
Step 1: What does this person/system need to DO?
  → "View the payments dashboard"
  → "Create alert rules for my team"
  → "Deploy dashboards from CI/CD"

Step 2: What is the MINIMUM role that enables this?
  → View dashboard → Viewer (basic)
  → Create alerts → Editor (basic) + Alerting Writer (fixed)
  → Deploy dashboards → Service account + Dashboards Writer

Step 3: Is there a MORE granular option?
  → Can we use a fixed role instead of Admin?
  → Can we scope to a folder instead of all dashboards?
  → Can we use a custom role for even finer control?

Step 4: Assign and document
  → Assign the minimum identified role
  → Document the business reason
  → Set a review date

Step 5: Review regularly
  → Quarterly access reviews
  → Remove access when no longer needed
  → Adjust when job roles change
```

---

### Common Least Privilege Mistakes

```
❌ Mistake 1: "I'll give Admin for now and fix it later"
   Fix: Always start with the minimum role.
        "Later" never comes.

❌ Mistake 2: "Everyone needs Editor to be productive"
   Fix: Most stakeholders only need Viewer.
        Only build-and-deploy users need Editor.

❌ Mistake 3: "The service account needs Admin to work"
   Fix: Service accounts almost never need Admin.
        Identify the specific permissions required.

❌ Mistake 4: "We share one Admin account for the team"
   Fix: Every person and system gets their own account.
        Shared accounts break audit trails.

❌ Mistake 5: "Contractors get the same access as employees"
   Fix: Contractors typically need Viewer access only.
        Time-bound accounts with explicit review dates.
```

---

<a name="auditing"></a>
## 11. 🔍 Auditing Access

### Viewing Current Role Assignments

```
Navigation:
Administration → Users and access → Users

View:
  All users and their current basic roles
  Filter by role to see who has Admin/Editor/Viewer
  Click a user to see their full role assignment

Administration → Users and access → Teams
  See all teams and their members
  Click a team to see its role assignments

Administration → Users and access → Service accounts
  See all service accounts and their tokens
  Check token expiry dates
```

---

### Quarterly Access Review Process

```
Step 1: Export user list
  Administration → Users and access → Users
  Export or use API:
  GET /api/org/users

Step 2: Review each user
  Questions to ask:
  □ Does this person still work here?
  □ Does their role match their current job?
  □ Have they logged in recently? (90+ days = suspicious)
  □ Do they still need access?

Step 3: Review service accounts
  Questions to ask:
  □ Is this service account still in use?
  □ Are all tokens valid and not expired?
  □ Does it have more permissions than required?

Step 4: Make changes
  Remove stale accounts
  Downgrade over-privileged roles
  Rotate old tokens

Step 5: Document findings
  Record the review in your audit system
  Note any exceptions with business justification
```

---

### Grafana Cloud Audit Logs (Enterprise)

```
Grafana Enterprise and Advanced plans include
full audit logging:

Navigation:
Administration → General → Audit logs

What is captured:
  ✅ User login and logout events
  ✅ Dashboard create, edit, delete
  ✅ Alert rule changes
  ✅ Datasource changes
  ✅ User and team modifications
  ✅ Role assignment changes
  ✅ API key creation and deletion

Use audit logs to:
  → Investigate security incidents
  → Demonstrate compliance
  → Review who changed what and when
  → Identify unusual access patterns
```

---

<a name="quiz"></a>
## 12. 🧠 Knowledge Check

### Questions

---

**Question 1**
A new developer joins your team. They need to create dashboards and write alert rules but should not be able to manage users or datasources. Which basic role should they receive?

```
A) Org Admin
B) Editor      ✅
C) Viewer
D) No role — use a custom role only
```

---

**Question 2**
An on-call engineer needs to be able to silence alerts during incidents but only needs to view dashboards otherwise. What is the best role combination?

```
A) Org Admin
B) Editor + Alerting Admin
C) Viewer + Alerting Writer (fixed role)    ✅
D) Viewer only

Explanation:
  Viewer gives dashboard read access.
  Alerting Writer fixed role allows creating silences.
  No need for Editor or Admin for this use case.
```

---

**Question 3**
Your CI/CD pipeline needs to deploy dashboards to Grafana Cloud. What type of identity should you use?

```
A) A shared admin user account
B) Your personal admin account credentials
C) A service account with Dashboards Writer role    ✅
D) An Editor user account shared by the team

Explanation:
  Service accounts are designed for automated systems.
  They have tokens (not passwords), can be given minimum
  permissions, and create clean audit trails.
```

---

**Question 4**
True or False: You can use roles to REMOVE permissions from a user.

```
FALSE    ✅

Explanation:
  In Grafana RBAC, roles are ADDITIVE only.
  You cannot use a role to take away a permission
  granted by another role. To restrict access,
  you should assign a lower basic role from the start.
```

---

**Question 5**
Your organisation has 50 finance team members who all need read-only access to the Finance folder. What is the most efficient way to manage this?

```
A) Assign Viewer role to each user individually
B) Create a Finance Team, assign all users to it,
   give the team Viewer access to the Finance folder    ✅
C) Give all 50 users Editor access
D) Create 50 custom roles, one per user

Explanation:
  Teams are the efficient way to manage group access.
  When someone joins or leaves, you only update
  team membership — not individual permissions.
```

---

**Question 6**
How often should you review user access assignments?

```
A) Never — set it and forget it
B) Only when someone leaves
C) Quarterly at minimum    ✅
D) Daily

Explanation:
  Quarterly reviews ensure stale access is removed,
  over-privileged accounts are downgraded, and
  your access model reflects current business needs.
  More frequent reviews may be required by compliance
  frameworks like SOX or PCI-DSS.
```

---

### Score Your Results

```
6/6 Correct:  🏆 RBAC Expert — you are ready to implement!
4-5 Correct:  ✅ Good understanding — review any missed questions
2-3 Correct:  📖 Review Sections 4-7 before implementing
0-1 Correct:  🔄 Restart the module from Section 1
```

---

<a name="summary"></a>
## 13. 📋 Summary & Next Steps

### What You Have Learned

```
✅ RBAC controls access through roles, not direct permissions
✅ Grafana Cloud has three layers: Basic, Fixed, and Custom roles
✅ Basic roles: Viewer → Editor → Admin (each includes the previous)
✅ Fixed roles extend Basic roles with fine-grained control
✅ Custom roles let you define precise permission sets
✅ Teams make managing group access efficient and scalable
✅ Service accounts handle automated and application access
✅ Least privilege means: minimum access to do the job
✅ Quarterly access reviews keep your RBAC model healthy
```

---

### Quick Reference Card

```
┌─────────────────────────────────────────────────────────────┐
│               Grafana Cloud RBAC Quick Reference             │
│                                                              │
│  BASIC ROLES                                                 │
│  Viewer   → Read dashboards only                            │
│  Editor   → Create and edit dashboards and alerts           │
│  Admin    → Full organisation management                     │
│                                                              │
│  DECISION GUIDE                                              │
│  Stakeholder viewing reports?      → Viewer                 │
│  Engineer building dashboards?     → Editor                 │
│  Managing Grafana platform?        → Admin (2-3 people max) │
│  On-call silencing alerts?         → Viewer + Alerting Writer│
│  CI/CD deploying dashboards?       → Service Account        │
│  Auditor reviewing access?         → Viewer + Users Reader  │
│                                                              │
│  GOLDEN RULES                                                │
│  1. Start with Viewer, escalate only if needed              │
│  2. Use Teams for group access management                   │
│  3. Service accounts for all automation                     │
│  4. Review access every quarter                             │
│  5. Document why every role was assigned                    │
└─────────────────────────────────────────────────────────────┘
```

---

### Next Steps

```
Immediate Actions (This Week)
  □ Review your own current role in Grafana Cloud
  □ Identify any users with Admin who only need Editor
  □ Check for service accounts with excessive permissions
  □ Ensure all engineers have their own individual accounts

Short Term (This Month)
  □ Create teams for each functional group
  □ Assign users to teams instead of individual permissions
  □ Audit service account tokens for expiry dates
  □ Document your organisation's RBAC policy

Long Term (This Quarter)
  □ Implement quarterly access review process
  □ Configure audit logging (Enterprise/Advanced)
  □ Integrate with your SSO/IdP for automatic role mapping
  □ Create custom roles for specialised use cases
```

---

### Additional Resources

| Resource | Location |
|----------|----------|
| Grafana RBAC Documentation | `grafana.com/docs/grafana/latest/administration/roles-and-permissions` |
| Access Control Permissions | `grafana.com/docs/grafana/latest/administration/roles-and-permissions/access-control` |
| Service Accounts | `grafana.com/docs/grafana/latest/administration/service-accounts` |
| Team Management | `grafana.com/docs/grafana/latest/administration/team-management` |
| Grafana Community Forum | `community.grafana.com` |

---