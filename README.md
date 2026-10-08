# Azure Privilege Audit: RBAC Assignments, Orphaned Access & PIM

## Overview

I performed an access audit in a live Azure training tenant to investigate how permissions were assigned, identify excessive access, and compare different ways to audit Azure role assignments.

The goal was to understand who had access to Azure resources, where their permissions applied, and what each auditing method could reveal or miss.

## Environment

- **Platform:** Microsoft Azure
- **Identity:** Microsoft Entra ID
- **Access level:** Reader, with an eligible role activated through Privileged Identity Management (PIM)
- **Tools:** Azure portal IAM, Azure CLI, Azure Resource Graph Explorer (KQL), and PIM
- **Focus:** Azure RBAC, inherited permissions, orphaned assignments, and privileged access

## Investigation

### 1. IAM Blade — Establishing the Baseline

I started with the Azure portal's Access control (IAM) blade and reviewed the role assignment export. I focused on the assigned roles, scopes, inherited permissions, and accounts with repeated privileged access.

One account stood out because it had Owner assignments across multiple resources. The repeated grants raised concerns about whether that level of access was necessary at every scope.

The export provided a useful baseline, but group assignments did not automatically show every member who could inherit access through those groups. Orphaned identities could also be difficult to identify through the portal alone.

**Finding:** Redundant Owner assignments across multiple scopes increased the potential blast radius of a compromised account.

<img width="952" height="857" alt="image" src="https://github.com/user-attachments/assets/7f305d49-3328-407a-9e53-3beb513ab13c" />


### 2. Azure CLI — Identifying an Orphaned Assignment

I used Azure CLI to enumerate role assignments on the production resource group.

**Command:**

```bash
az role assignment list --resource-group rg-madhatlabs-prod-cus
```

I compared the output with the full JSON export. One assignment contained a `principalId`, but its `principalName` was empty. This indicated that the identity could no longer be resolved and led me to identify an orphaned role assignment.

This demonstrated the value of checking the underlying assignment data instead of relying exclusively on the portal interface.

**Finding:** A role assignment remained after its associated identity was deleted.

<img width="707" height="132" alt="image" src="https://github.com/user-attachments/assets/d9c9c124-691b-4ddf-ac70-92111dfaf7de" />


### 3. Azure Resource Graph — Querying with KQL

Next, I used Azure Resource Graph Explorer to query authorization resources. This allowed me to search role assignments across the available tenant data in a single query instead of manually checking each resource group.

**Query:**

```kusto
authorizationresources
| where type =~ 'microsoft.authorization/roleassignments'
| extend principalId = tostring(properties.principalId)
| extend description = properties.description
| where description contains "MadHat"
| project name, principalId, principalType = properties.principalType, scope = properties.scope, description
```

The query filters for role assignment records, extracts the principal ID and description, searches for descriptions containing the specified text, and returns fields useful for the audit.

Resource Graph made it easier to investigate assignments at scale. However, this method did not replace PIM for reviewing eligible access.

**Finding:** Tenant-wide querying improved visibility into active role assignments, but eligible assignments required a separate review.

<img width="762" height="475" alt="image" src="https://github.com/user-attachments/assets/0aff1763-0d8c-4bfb-b712-a1dd962ee57e" />


### 4. Privileged Identity Management — Eligible vs. Active

I used PIM to review and export role assignment information, focusing on the difference between eligible and active access.

- **Eligible:** The principal can activate the role but does not currently hold its permissions.
- **Active:** The principal currently holds the role and can use its permissions.

The export demonstrated why an access audit should consider both states. Reviewing active assignments alone does not provide the full picture of who can obtain privileged access.

PIM also provides activation history that can help establish who activated a role, when it happened, and what justification was recorded.

**Finding:** Privileged access should be reviewed for unnecessary standing assignments and opportunities to use just-in-time activation.


### 5. The Hunt — Combining the Evidence

For the final stage, I combined my understanding of RBAC, role scopes, least privilege, and PIM to investigate an over-provisioned account in a hidden resource group.

The previous methods each provided a different view of access. IAM established the baseline, CLI helped expose an orphaned assignment, Resource Graph enabled broader querying, and PIM showed eligible and active access.

The final investigation required me to apply those concepts together rather than follow a single prescribed query or portal walkthrough.

**Finding:** Access needs to be evaluated across roles and scopes to understand the potential impact of over-provisioned permissions.


## Methodology Comparison

| Method | What it reveals | Blind spot |
|---|---|---|
| IAM blade / export | Role assignments at a scope, including inherited access | Group members are not expanded, and orphaned identities can be difficult to spot |
| Azure CLI | Role assignments and fields such as an empty `principalName` | Typically requires separate commands for different scopes |
| Resource Graph (KQL) | Queries role assignments across tenant data in one query | Does not show eligible assignments |
| PIM export | Eligible vs. active assignments and activation history | Does not replace auditing standing assignments outside PIM |

The main takeaway is that these tools complement each other. Each provides a different perspective, and combining them produces a more complete access review.

## Findings and Recommendations

### Findings

1. **Excessive permissions:** Repeated Owner assignments across multiple scopes increased the potential impact of account compromise.
2. **Orphaned access:** A role assignment remained associated with a deleted identity.
3. **Privileged access exposure:** Eligible and active assignments need to be reviewed separately.
4. **Audit coverage gaps:** No single method provided every detail needed for the investigation.

### Recommendations

- **Remove orphaned assignments:** Verify the principal and scope, then remove assignments that are no longer justified.
- **Reduce excessive Owner access:** Use the narrowest appropriate job-function role at the smallest necessary scope.
- **Use PIM:** Where supported, keep privileged roles eligible by default and require appropriate MFA, justification, approval when necessary, and limited activation durations.
- **Prefer group-based access:** Assign permissions to appropriately managed groups instead of individual users where practical.
- **Review access quarterly:** Repeat the audit and track findings through remediation.
- **Make findings actionable:** Convert audit results into remediation tickets with an owner, priority, and completion status.

## What I Learned

- Azure RBAC auditing requires understanding who has access, what they can do, and where their permissions apply.
- Deleting an identity does not necessarily remove its associated role assignments.
- Azure CLI can expose useful assignment details, while Resource Graph makes broad queries more efficient.
- PIM adds visibility into eligible access and privileged activation history.
- Comparing methods helps identify gaps that a single interface may miss.

If I repeated this audit, I would standardize the exports, redact sensitive information before saving evidence, and track every finding through remediation and verification.

## Conclusion

This project gave me practical experience auditing Azure RBAC through multiple methods rather than relying on a single interface. It reinforced the importance of least privilege, orphaned assignment cleanup, just-in-time privileged access, and recurring access reviews.

An effective access audit is not just a list of permissions. It is a repeatable process for identifying unnecessary access, understanding the limitations of each tool, and turning findings into remediation work.
