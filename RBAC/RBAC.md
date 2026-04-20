# RBAC Strategy Advice for Large Financial Organisations in Grafana Cloud

## 1. 🏗️ Foundation Principles

### Least Privilege Access
```
✅ Grant minimum permissions required for each role
✅ Default to READ-ONLY for most users
✅ Escalate permissions only when justified
✅ Regularly audit and revoke unused permissions
```

### Separation of Duties
```
Platform Admins    → Grafana infrastructure management
Security Team      → Audit logs, compliance dashboards
Dev Teams          → Their own service dashboards ONLY
Finance/Business   → Business metrics dashboards (read-only)
On-Call Engineers  → Read + Silence alerts only
```

---

## 2. 📊 Grafana Cloud RBAC Role Structure

### Built-in Roles to Leverage

| Role | Recommended For | Key Permissions |
|------|----------------|-----------------|
| **Org Admin** | Platform team leads only | Full org access |
| **Editor** | Senior engineers | Create/edit dashboards |
| **Viewer** | Business users, junior staff | Read-only access |
| **No Basic Role** | Custom role assignments | Fine-grained control |

### Custom Role Example Structure
```
Financial Org
├── Platform Engineering
│   └── Grafana Org Admin
├── SRE / On-Call
│   └── Editor + Alert management
├── Application Teams
│   └── Editor (scoped to team folders)
├── Security & Compliance
│   └── Viewer + Audit log access
├── Business Stakeholders
│   └── Viewer (specific dashboards only)
└── External Auditors
    └── Viewer (time-limited, specific folders)
```

---

## 3. 🔐 Identity & Authentication Strategy

### SSO Integration (Strongly Recommended)
```yaml
# Recommended Auth Flow
Identity Provider (Okta/Azure AD/Ping)
    ↓
SAML 2.0 or OAuth 2.0
    ↓
Grafana Cloud SSO
    ↓
Role Mapping via Group Claims
```

### Group-to-Role Mapping Example
```ini
# Azure AD Group Mapping
[auth.azuread]
groups_attribute_path = groups

# Map AD groups to Grafana roles
AD Group: GrafanaAdmins     → Grafana Role: Admin
AD Group: GrafanaSRE        → Grafana Role: Editor
AD Group: GrafanaViewers    → Grafana Role: Viewer
AD Group: GrafanaCompliance → Grafana Role: Viewer + Custom Audit Role
```

### MFA Requirements
```
🔒 Enforce MFA for ALL users
🔒 Require hardware tokens for Org Admins
🔒 Session timeout: 30 mins for privileged roles
🔒 Session timeout: 8 hours for standard viewers
```

---

## 4. 📁 Folder & Dashboard Permissions Strategy

### Recommended Folder Structure
```
📁 Root
├── 📁 Platform & Infrastructure      [SRE: Edit | Others: View]
│   ├── 📊 Kubernetes Overview
│   ├── 📊 Network Performance
│   └── 📊 Cloud Costs
├── 📁 Application Teams
│   ├── 📁 Payments Team             [Payments: Edit | SRE: View]
│   ├── 📁 Trading Platform          [Trading: Edit | SRE: View]
│   └── 📁 Customer Portal           [Portal Team: Edit | SRE: View]
├── 📁 Security & Compliance         [Security: Edit | Auditors: View]
│   ├── 📊 Audit Logs
│   ├── 📊 Failed Auth Attempts
│   └── 📊 Data Access Patterns
├── 📁 Business Intelligence         [BI Team: Edit | Stakeholders: View]
│   ├── 📊 Transaction Volumes
│   └── 📊 SLA Reports
└── 📁 Regulatory Reporting          [Compliance: Edit | Auditors: View]
    ├── 📊 FCA Reports
    └── 📊 PRA Dashboards
```

### Folder Permission Matrix

| Folder | Platform | SRE | App Teams | Security | Business | Auditors |
|--------|----------|-----|-----------|----------|----------|----------|
| Platform & Infra | Admin | Edit | View | View | ❌ | ❌ |
| App Team Folders | Admin | View | Edit (own) | View | ❌ | ❌ |
| Security & Compliance | Admin | View | ❌ | Edit | ❌ | View |
| Business Intelligence | Admin | ❌ | ❌ | View | Edit | View |
| Regulatory Reporting | Admin | ❌ | ❌ | View | ❌ | View |

---

## 5. 🎯 Data Source Permissions

### Critical for Financial Organisations
```
✅ Restrict data source access by team
✅ Never allow direct data source editing by non-admins
✅ Use separate data sources per environment
✅ Mask sensitive fields in Explore mode
```

### Data Source Permission Strategy
```
Production Metrics    → SRE + Platform (Query Only)
Production Logs       → SRE + Security (Query Only)
Production Traces     → App Teams (Query, own services only)
Staging/Dev Metrics   → All Engineers (Query + Explore)
Audit/Security Logs   → Security Team ONLY
Financial Data        → BI + Compliance ONLY
```

### Disable Explore for Sensitive Sources
```yaml
# Grafana configuration recommendation
[datasources]
# Disable Explore for production financial data sources
# Limit ad-hoc querying of sensitive databases
hide_from_explore = true  # Applied to financial data sources
```

---

## 6. 🚨 Alerting RBAC Considerations

### Alert Permission Tiers
```
Tier 1 - View Alerts Only
└── Business users, junior staff

Tier 2 - Silence Alerts
└── On-call engineers (during incidents)

Tier 3 - Create/Edit Alert Rules
└── Senior engineers, SRE team

Tier 4 - Manage Notification Policies
└── Platform leads, SRE managers

Tier 5 - Full Alert Administration
└── Grafana Org Admins only
```

### Custom Alert Role Example
```json
{
  "name": "OnCall-Engineer",
  "permissions": [
    {"action": "alert.instances:read"},
    {"action": "alert.instances:create"},
    {"action": "alert.instances.silence:create"},
    {"action": "alert.rules:read"},
    {"action": "dashboards:read", "scope": "dashboards:*"}
  ]
}
```

---

## 7. 📋 Compliance & Audit Requirements

### Regulatory Considerations (FCA/PRA/SOX/GDPR)
```
📌 Enable audit logging for ALL permission changes
📌 Log all dashboard access for sensitive data
📌 Retain audit logs for minimum 7 years (SOX)
📌 Document all RBAC changes with justification
📌 Quarterly access reviews mandatory
📌 Immediate revocation process for leavers
```

### Audit Log Monitoring Dashboard
```promql
# Track permission escalations
sum(increase(grafana_rbac_permission_changes_total[$__range]))

# Monitor admin actions
count_over_time(
  {job="grafana", level="info", 
   message=~".*permission.*"}[$__range]
)
```

---

## 8. 🔄 Operational Processes

### Joiner / Mover / Leaver Process
```
JOINER
├── Request access via ITSM ticket
├── Manager approval required
├── Auto-provisioned via AD group membership
└── Welcome email with access details

MOVER (Role Change)
├── Old permissions removed BEFORE new granted
├── Manager + team lead approval
├── 24hr maximum transition window
└── Audit trail maintained

LEAVER
├── Same-day revocation on HR notification
├── Automated via AD group removal
├── Service account review triggered
└── Dashboard ownership transferred
```

### Quarterly Access Review Checklist
```
□ Export all user-role assignments
□ Review with team managers
□ Identify stale accounts (90+ days inactive)
□ Check for over-privileged accounts
□ Verify service accounts still required
□ Document review findings
□ Submit compliance evidence
```

---

## 9. 🤖 Service Account Strategy

### For Automated Systems
```
✅ Dedicated service account per application
✅ Minimum scope - only required permissions
✅ Rotate tokens every 90 days
✅ Never share service accounts between teams
✅ Store tokens in secrets manager (Vault/Azure KV)
✅ Monitor service account usage patterns
```

### Service Account Naming Convention
```
sa-[team]-[purpose]-[environment]

Examples:
sa-payments-dashboard-provisioning-prod
sa-sre-alerting-automation-prod
sa-compliance-reporting-viewer-prod
sa-cicd-dashboard-deploy-all
```

---

## 10. ⚠️ Common Pitfalls to Avoid

```
❌ DON'T give everyone Editor role "temporarily"
❌ DON'T use shared/generic accounts
❌ DON'T skip the access review process
❌ DON'T store API tokens in code repositories
❌ DON'T allow Org Admin for entire teams
❌ DON'T forget to restrict Explore on production data
❌ DON'T provision access manually without audit trail
❌ DON'T ignore inactive accounts
```

---

## 11. 📐 Implementation Roadmap

```
Phase 1 - Foundation (Month 1-2)
├── SSO Integration
├── Basic role structure
└── Folder hierarchy creation

Phase 2 - Refinement (Month 2-3)
├── Custom roles for specialist teams
├── Data source permissions
└── Service account migration

Phase 3 - Compliance (Month 3-4)
├── Audit logging dashboards
├── Access review process
└── Documentation & runbooks

Phase 4 - Optimisation (Ongoing)
├── Quarterly access reviews
├── Automation improvements
└── Policy refinements
```

---

## 📚 Key Grafana Resources

| Resource | URL |
|----------|-----|
| RBAC Overview | `http://grafana.com/docs/grafana/latest/administration/roles-and-permissions` |
| Custom Roles | `http://grafana.com/docs/grafana/latest/administration/roles-and-permissions/access-control` |
| Team Management | `http://grafana.com/docs/grafana/latest/administration/team-management` |
| Service Accounts | `http://grafana.com/docs/grafana/latest/administration/service-accounts` |

> 💡 **Key Takeaway:** For financial organisations, the priority order should be **Security → Compliance → Usability** — never compromise the first two for operational convenience.