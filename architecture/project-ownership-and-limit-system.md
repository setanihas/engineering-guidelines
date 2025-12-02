# RBAC Project Ownership & Transfer Flow Documentation

## Table of Contents
- [Overview](#overview)
- [System Architecture](#system-architecture)
- [Limit System](#limit-system)
- [Project Ownership Transfer Flow](#project-ownership-transfer-flow)
- [Organization Project Creation Flow](#organization-project-creation-flow)
- [Edge Cases & Considerations](#edge-cases--considerations)
- [Transaction Management](#transaction-management)

---

## Overview

This document describes the ownership transfer process and project creation flow within an organization for a Role-Based Access Control (RBAC) system. The system enforces user-based and project-based limits, ensuring that ownership transfers and project creations comply with quota restrictions.

### Key Concepts

- **Project Owner**: A single user entity that owns a project
- **Organization Owner**: A role within an organization with special privileges
- **Limit Usage**: Real-time tracking of resource consumption
- **Limit Entries**: Defined maximum allowed limits for users and projects

---

## System Architecture

### Ownership Structure

```
Organization
  ├── Organization Owner (role)
  └── Projects
        └── Project Owner (single user entity)
```

**Rules:**
- A project has exactly ONE owner (single entity)
- An organization has an owner role (can be assigned to users)
- Organization owner and project owner are separate concepts

### Limit Buckets

The system maintains two distinct buckets for limit management:

1. **Limit Entries**: Defines maximum allowed limits
2. **Limit Usage**: Tracks current resource consumption (real-time)

**Update Behavior**: Usage is incremented/decremented immediately when actions occur (e.g., storage upload increases current usage + new content size)

---

## Limit System

### User-Based Limits

```javascript
export const USER_LIMIT_KEY = {
  PROJECT_WITH_DOMAIN_CONNECTION: "project_with_domain_connection",
  TEAM_MEMBER: "team_member",
  STORAGE_SIZE: "storage_size",
  VISITOR_COUNT: "visitor_count",
  FORM_COUNT: "form_count",
  MAX_PUBLISHED_PROJECTS: "max_published_projects",
};
```

**Scope**: Tied to `user_id`

**Description**:
- `project_with_domain_connection`: Number of projects with custom domains
- `team_member`: Total team members across all owned projects
- `storage_size`: Total storage consumed across all owned projects
- `visitor_count`: Total visitors across all owned projects
- `form_count`: Total forms across all owned projects
- `max_published_projects`: Maximum number of published projects allowed

### Project-Based Limits

```javascript
export const PROJECT_LIMIT_KEY = {
  LOCALE_COUNT: "locale_count",
  AB_TEST: "ab_test",
  AUTO_VERSION_COUNT: "auto_version_count",
  CUSTOM_VERSION_COUNT: "custom_version_count",
};
```

**Scope**: Tied to `project_id`

**Description**:
- `locale_count`: Number of locales in a project
- `ab_test`: Number of A/B tests in a project
- `auto_version_count`: Number of automatic versions
- `custom_version_count`: Number of custom versions

---

## Project Ownership Transfer Flow

### Overview

Only the current project owner can initiate a transfer. The process requires approval from the target user and validates that the new owner has sufficient limits before completing the transfer.

### Process Flow

#### 1. Transfer Request Initiation

**Actor**: Current Project Owner

**Steps**:
1. Verify requester is the current owner
   - If not → Return `Unauthorized Error`
2. Create transfer request with `status: PENDING`
3. Send notification to target user
4. Wait for approval

#### 2. Transfer Approval/Rejection

**Actor**: Target User (New Owner)

**Decision Point**:
- **Reject**: 
  - Update transfer status to `REJECTED`
  - No entries are updated
  - Process ends
  
- **Accept**: 
  - Proceed to limit validation

#### 3. Limit Validation (Pre-Transfer)

Before any changes occur, validate the new owner's limits:

**Step 3.1: Published Project Check**
```
IF project.is_published == true:
  CHECK new_owner.max_published_projects limit
  IF insufficient → Return Error: "Limit Exceeded: max_published_projects"
```

**Step 3.2: Storage Check**
```
IF project.storage_usage > 0:
  CHECK new_owner.storage_size limit
  IF insufficient → Return Error: "Limit Exceeded: storage_size"
```

**Step 3.3: Domain Connection Check**
```
IF project.has_domain_connection == true:
  CHECK new_owner.project_with_domain_connection limit
  IF insufficient → Return Error: "Limit Exceeded: project_with_domain_connection"
```

**Step 3.4: Team Member Check**
```
IF project.team_member_count > 0:
  CHECK new_owner.team_member limit
  IF insufficient → Return Error: "Limit Exceeded: team_member"
```

**Critical**: If ANY limit check fails, the entire transfer is rejected. No partial transfers are allowed.

#### 4. Transfer Execution (Within Transaction)

**Step 4.1: Update Project Ownership**
```sql
UPDATE projects 
SET owner_id = new_owner_id 
WHERE id = project_id
```

**Step 4.2: Decrement Old Owner Usage**
```javascript
// Decrement old owner's usage
old_owner.usage.max_published_projects -= 1  // if published
old_owner.usage.storage_size -= project.storage_usage
old_owner.usage.project_with_domain_connection -= 1  // if has domain
old_owner.usage.team_member -= project.team_member_count
```

**Step 4.3: Increment New Owner Usage**
```javascript
// Increment new owner's usage
new_owner.usage.max_published_projects += 1  // if published
new_owner.usage.storage_size += project.storage_usage
new_owner.usage.project_with_domain_connection += 1  // if has domain
new_owner.usage.team_member += project.team_member_count
```

**Step 4.4: Finalize Transfer**
```javascript
transfer_request.status = "COMPLETED"
transaction.commit()
```

**Step 4.5: Send Notifications**
- Notify old owner: Transfer completed
- Notify new owner: Transfer completed

### Transfer Flow Diagram Summary

```
Start (Owner initiates transfer)
  ↓
[Check: Is requester current owner?]
  ↓ Yes
[Create transfer request: PENDING]
  ↓
[Notify target user]
  ↓
[Wait for user decision]
  ↓
[User accepts/rejects?]
  ↓ Reject → END (no changes)
  ↓ Accept
[Validate ALL new owner limits]
  ↓ Any limit exceeded? → ERROR
  ↓ All limits OK
[BEGIN TRANSACTION]
  ↓
[Update project.owner_id]
  ↓
[Decrement old owner usage]
  ↓
[Increment new owner usage]
  ↓
[Update transfer status: COMPLETED]
  ↓
[COMMIT TRANSACTION]
  ↓
[Send notifications]
  ↓
END (Transfer successful)
```

---

## Organization Project Creation Flow

### Overview

Projects can be associated with an organization in two ways:
1. **Direct Creation**: Created directly within an organization (organization owner becomes project owner)
2. **Assignment**: An existing project is assigned to an organization later
3. **Member Creation**: An organization member creates a project within the organization

When a project is created within an organization, the organization owner automatically becomes the project owner. The system validates the organization owner's user-based limits before creation.

### Process Flow

#### 1. Project Creation Request

**Actor**: Organization Owner

**Steps**:
1. Verify requester is the organization owner
   - If not → Return `Unauthorized Error: Only Org Owner can create projects`
2. Determine project owner: `project.owner_id = organization.owner_id`

#### 2. User Limit Validation

Validate the organization owner's user-based limits:

**Step 2.1: Published Project Limit**
```
CHECK org_owner.max_published_projects limit
IF insufficient → Return Error: "Limit Exceeded: max_published_projects"
```

**Step 2.2: Storage Limit**
```
CHECK org_owner.storage_size limit (for initial storage allocation)
IF insufficient → Return Error: "Limit Exceeded: storage_size"
```

**Step 2.3: Domain Connection Limit (if applicable)**
```
IF domain_connection_requested:
  CHECK org_owner.project_with_domain_connection limit
  IF insufficient → Return Error: "Limit Exceeded: project_with_domain_connection"
```

**Step 2.4: Team Member Limit (if applicable)**
```
IF team_members_to_add > 0:
  CHECK org_owner.team_member limit
  IF insufficient → Return Error: "Limit Exceeded: team_member"
```

#### 3. Project Creation (Within Transaction)

**Step 3.1: Create Project Entity**
```javascript
project = {
  owner_id: organization.owner_id,
  organization_id: organization.id,
  // ... other project fields
}
```

**Step 3.2: Initialize Project Limits**
```javascript
// Create project-based limit entries
project_limits = {
  locale_count: default_value,
  ab_test: default_value,
  auto_version_count: default_value,
  custom_version_count: default_value
}
```

**Step 3.3: Update Owner User Usage**
```javascript
org_owner.usage.max_published_projects += 1
org_owner.usage.storage_size += initial_storage
org_owner.usage.project_with_domain_connection += 1  // if domain connected
org_owner.usage.team_member += added_member_count  // if members added
```

**Step 3.4: Assign Roles**
```
Assign project roles to other organization members (if applicable)
```

**Step 3.5: Commit Transaction**
```javascript
transaction.commit()
```

**Step 3.6: Send Notifications**
- Notify organization owner: Project created
- Notify organization members: New project available

### Project Creation Flow Diagram Summary

```
Start (Org owner creates project)
  ↓
[Check: Is requester org owner?]
  ↓ No → ERROR: Unauthorized
  ↓ Yes
[Set project.owner_id = org.owner_id]
  ↓
[Validate org owner's user limits]
  ↓ Any limit exceeded? → ERROR
  ↓ All limits OK
[BEGIN TRANSACTION]
  ↓
[Create project entity]
  ↓
[Initialize project limits (locale, ab_test, etc.)]
  ↓
[Update org owner usage]
  ↓
[Assign roles to org members]
  ↓
[COMMIT TRANSACTION]
  ↓
[Send notifications]
  ↓
END (Project created, Owner: Org Owner)
```

---

## Assigning Existing Project to Organization

### Overview

An existing project (with its own owner) can be assigned to an organization. When assigned, the organization owner automatically becomes the project owner. This is essentially an ownership transfer combined with organization assignment.

### Process Flow

#### 1. Assignment Request

**Actor**: Project Owner

**Prerequisites**:
- Requester must be the current project owner
- Target organization must exist
- Project must not already be assigned to another organization

**Steps**:
1. Verify requester is the project owner
   - If not → Return `Unauthorized Error`
2. Check if project already has an organization
   - If yes → Return `Error: Project already assigned to an organization`
3. Get organization owner for ownership transfer
4. Proceed to validation

#### 2. Limit Validation (Organization Owner)

Before assignment, validate the organization owner's limits (same as transfer flow):

**Step 2.1: Published Project Check**
```
IF project.is_published == true:
  CHECK org_owner.max_published_projects limit
  IF insufficient → Return Error: "Limit Exceeded: max_published_projects"
```

**Step 2.2: Storage Check**
```
IF project.storage_usage > 0:
  CHECK org_owner.storage_size limit
  IF insufficient → Return Error: "Limit Exceeded: storage_size"
```

**Step 2.3: Domain Connection Check**
```
IF project.has_domain_connection == true:
  CHECK org_owner.project_with_domain_connection limit
  IF insufficient → Return Error: "Limit Exceeded: project_with_domain_connection"
```

**Step 2.4: Team Member Check**
```
IF project.team_member_count > 0:
  CHECK org_owner.team_member limit
  IF insufficient → Return Error: "Limit Exceeded: team_member"
```

#### 3. Assignment Execution (Within Transaction)

**Step 3.1: Update Project Ownership and Organization**
```javascript
project.organization_id = target_organization_id
project.owner_id = organization.owner_id  // Ownership transfer
```

**Step 3.2: Decrement Old Owner Usage**
```javascript
// Decrement old owner's usage
old_owner.usage.max_published_projects -= 1  // if published
old_owner.usage.storage_size -= project.storage_usage
old_owner.usage.project_with_domain_connection -= 1  // if has domain
old_owner.usage.team_member -= project.team_member_count
```

**Step 3.3: Increment Organization Owner Usage**
```javascript
// Increment org owner's usage
org_owner.usage.max_published_projects += 1  // if published
org_owner.usage.storage_size += project.storage_usage
org_owner.usage.project_with_domain_connection += 1  // if has domain
org_owner.usage.team_member += project.team_member_count
```

**Step 3.4: Grant Organization Access**
```
Grant organization members appropriate roles/permissions on the project
(based on their organization roles)
```

**Step 3.5: Finalize Assignment**
```javascript
transaction.commit()
```

**Step 3.6: Send Notifications**
- Notify old project owner: Project assigned to organization, ownership transferred
- Notify organization owner: New project added, you are now the owner
- Notify organization members: New project available

### Important Notes

- **Ownership DOES Change**: Organization owner becomes the new project owner
- **Limit Transfer Required**: Old owner's usage decrements, org owner's usage increments
- **This is a Transfer + Assignment**: Combines ownership transfer with org assignment
- **Organization Access**: Organization members gain access based on their org roles
- **Validation Required**: Must validate org owner's limits before proceeding

### Assignment Flow Diagram Summary

```
Start (Project owner assigns to org)
  ↓
[Check: Is requester project owner?]
  ↓ No → ERROR: Unauthorized
  ↓ Yes
[Check: Project already in org?]
  ↓ Yes → ERROR: Already assigned
  ↓ No
[Get organization.owner_id]
  ↓
[Validate org owner's limits]
  ↓ Any limit exceeded? → ERROR
  ↓ All limits OK
[BEGIN TRANSACTION]
  ↓
[Update project.organization_id]
  ↓
[Update project.owner_id = org.owner_id]
  ↓
[Decrement old owner usage]
  ↓
[Increment org owner usage]
  ↓
[Grant org members access/roles]
  ↓
[COMMIT TRANSACTION]
  ↓
[Send notifications]
  ↓
END (Project assigned, Owner: Org Owner)
```

---

## Organization Member Creating Project

### Overview

An organization member (not the organization owner) can create a project within the organization. The creating member becomes the project owner.

### Process Flow

#### 1. Project Creation Request

**Actor**: Organization Member

**Prerequisites**:
- Requester must be a member of the organization
- Organization must allow member project creation (policy check)

**Steps**:
1. Verify requester is an organization member
   - If not → Return `Unauthorized Error`
2. Check organization policy for member project creation
   - If not allowed → Return `Error: Policy restriction`
3. Determine project owner: `project.owner_id = requester.user_id` (NOT org owner)

#### 2. User Limit Validation

Validate the **creating member's** user-based limits (not org owner's limits):

**Step 2.1: Published Project Limit**
```
CHECK member.max_published_projects limit
IF insufficient → Return Error: "Limit Exceeded: max_published_projects"
```

**Step 2.2: Storage Limit**
```
CHECK member.storage_size limit (for initial storage allocation)
IF insufficient → Return Error: "Limit Exceeded: storage_size"
```

**Step 2.3: Domain Connection Limit (if applicable)**
```
IF domain_connection_requested:
  CHECK member.project_with_domain_connection limit
  IF insufficient → Return Error: "Limit Exceeded: project_with_domain_connection"
```

**Step 2.4: Team Member Limit (if applicable)**
```
IF team_members_to_add > 0:
  CHECK member.team_member limit
  IF insufficient → Return Error: "Limit Exceeded: team_member"
```

#### 3. Project Creation (Within Transaction)

**Step 3.1: Create Project Entity**
```javascript
project = {
  owner_id: creating_member.user_id,  // Member becomes owner
  organization_id: organization.id,
  // ... other project fields
}
```

**Step 3.2: Initialize Project Limits**
```javascript
// Create project-based limit entries
project_limits = {
  locale_count: default_value,
  ab_test: default_value,
  auto_version_count: default_value,
  custom_version_count: default_value
}
```

**Step 3.3: Update Creating Member's Usage**
```javascript
creating_member.usage.max_published_projects += 1
creating_member.usage.storage_size += initial_storage
creating_member.usage.project_with_domain_connection += 1  // if domain connected
creating_member.usage.team_member += added_member_count  // if members added
```

**Step 3.4: Grant Organization Access**
```
Grant other organization members appropriate roles/permissions
(based on organization policy)
```

**Step 3.5: Commit Transaction**
```javascript
transaction.commit()
```

**Step 3.6: Send Notifications**
- Notify creating member: Project created
- Notify organization owner: New project created by member
- Notify organization members: New project available

### Important Notes

- **Member Becomes Owner**: The creating member (not org owner) becomes the project owner
- **Member's Limits Apply**: The creating member's user limits are checked and updated
- **Organization Context**: Project is created within organization context but owned by member
- **Ownership Transfer**: Later, the project can be transferred to org owner or another user

### Member Project Creation Flow Diagram Summary

```
Start (Org member creates project)
  ↓
[Check: Is requester org member?]
  ↓ No → ERROR: Unauthorized
  ↓ Yes
[Check: Org policy allows member creation?]
  ↓ No → ERROR: Policy restriction
  ↓ Yes
[Set project.owner_id = member.user_id]
  ↓
[Validate MEMBER's user limits]
  ↓ Any limit exceeded? → ERROR
  ↓ All limits OK
[BEGIN TRANSACTION]
  ↓
[Create project entity]
  ↓
[Initialize project limits]
  ↓
[Update MEMBER's usage (not org owner)]
  ↓
[Grant org members access]
  ↓
[COMMIT TRANSACTION]
  ↓
[Send notifications]
  ↓
END (Project created, Owner: Creating Member)
```

---

## Edge Cases & Considerations

### 1. Transfer During Active Usage

**Scenario**: Transfer is pending while the project is actively being used (e.g., files are being uploaded).

**Behavior**: 
- Usage updates are real-time and atomic
- Transfer validation checks the current state at the moment of acceptance
- The transaction ensures consistency between owner change and usage updates

**Recommendation**: Implement row-level locking on the project during transfer to prevent race conditions.

### 2. Unpublished Projects During Transfer

**Scenario**: Project is unpublished while transfer is pending.

**Behavior**:
- If the project is unpublished before acceptance, the `max_published_projects` limit check will not apply
- Old owner's usage is already decremented when unpublishing
- Transfer only needs to validate and update remaining limits

### 3. Limit Changes During Pending Transfer

**Scenario**: New owner's limits are reduced while transfer is pending.

**Behavior**:
- Validation occurs at acceptance time, not at request time
- If limits are reduced and now insufficient, the transfer will fail

### 4. Concurrent Transfers

**Scenario**: Multiple transfers are initiated for different projects to the same target user.

**Behavior**:
- Each transfer validates independently at acceptance time
- If multiple transfers are accepted simultaneously, race conditions may occur
- Consider implementing queue-based processing or pessimistic locking

### 5. Organization Deletion

**Scenario**: Organization is deleted while projects exist.

**Behavior**: This scenario should be handled by your business logic:
- Option A: Prevent deletion if projects exist
- Option B: Unassign all projects (convert to personal projects, owners unchanged)
- Option C: Delete all projects (with confirmation)

### 6. Project Assignment Conflicts

**Scenario**: Attempting to assign a project that's already in another organization.

**Behavior**:
- Validation should prevent assigning to multiple organizations
- Project must first be unassigned from current organization
- Unassignment requires transferring ownership back (to whom? - business decision needed)

### 7. Project Unassignment from Organization

**Scenario**: Removing a project from an organization.

**Critical Question**: Who becomes the owner after unassignment?
- Option A: Previous owner (requires maintaining history)
- Option B: Current user requesting unassignment (if they have permission)
- Option C: Not allowed - projects cannot be unassigned, only transferred between orgs

**If unassignment is allowed**:
- Must validate new owner's limits
- Decrement org owner usage
- Increment new owner usage
- This is essentially another ownership transfer

### 8. Organization Member Limits vs Org Owner Limits

**Scenario**: An org member creates a project, but their personal limits are lower than org owner's limits.

**Behavior**:
- Each member's personal limits apply to their owned projects
- Organization membership does not grant access to org owner's limits
- If member hits limits, they cannot create more projects even within the org

### 8. Rollback Scenarios

**Critical Failure Points**:
- Database transaction failure during usage update
- Network failure during notification sending

**Rollback Strategy**:
- All database changes are within a transaction: automatic rollback on failure
- Notifications are sent after transaction commit: if notification fails, the transfer is still complete
- Consider implementing a retry mechanism for failed notifications

---

## Transaction Management

### Atomicity Guarantee

All ownership changes and usage updates MUST occur within a single database transaction:

```javascript
BEGIN TRANSACTION
  1. Update project.owner_id
  2. Decrement old owner usage
  3. Increment new owner usage
  4. Update transfer status (if applicable)
COMMIT TRANSACTION
```

**Failure Handling**:
- If ANY step fails → Entire transaction rolls back
- No partial updates are allowed
- System returns to pre-transfer state

### Recommended Implementation

```javascript
try {
  await db.transaction(async (trx) => {
    // 1. Update ownership
    await trx('projects')
      .where({ id: projectId })
      .update({ owner_id: newOwnerId });
    
    // 2. Decrement old owner
    await trx('user_limit_usage')
      .where({ user_id: oldOwnerId })
      .decrement('max_published_projects', 1);
    // ... other decrements
    
    // 3. Increment new owner
    await trx('user_limit_usage')
      .where({ user_id: newOwnerId })
      .increment('max_published_projects', 1);
    // ... other increments
    
    // 4. Update transfer status
    await trx('transfer_requests')
      .where({ id: transferId })
      .update({ status: 'COMPLETED' });
  });
  
  // 5. Send notifications (outside transaction)
  await sendNotifications();
  
} catch (error) {
  // Transaction automatically rolled back
  logger.error('Transfer failed:', error);
  throw error;
}
```

---

## Best Practices

1. **Always validate limits before starting a transaction**
   - Reduces unnecessary database operations
   - Provides faster feedback to users

2. **Use pessimistic locking for critical operations**
   - Prevents race conditions during transfers
   - Ensures data consistency

3. **Log all ownership changes**
   - Maintain an audit trail
   - Helps debug transfer issues

4. **Implement idempotency**
   - Transfer requests should be idempotent
   - Prevent duplicate processing

5. **Monitor usage metrics**
   - Track transfer success/failure rates
   - Identify common limit bottlenecks

6. **Set reasonable timeout values**
   - Pending transfers should have expiration
   - Automatically reject expired transfers

---

## Future Considerations

### Potential Enhancements

1. **Bulk Transfers**: Support transferring multiple projects at once
2. **Transfer History**: Maintain a complete history of ownership changes
3. **Partial Transfers**: Allow transferring only certain resources (advanced use case)
4. **Scheduled Transfers**: Allow scheduling transfers for a future date
5. **Transfer Rollback**: Allow reversing a transfer within a grace period
6. **Limit Forecasting**: Warn users before initiating transfers about potential limit issues

### Scalability Considerations

As the system grows, consider:
- Implementing a job queue for transfer processing
- Caching limit entries to reduce database queries
- Horizontal scaling of the transfer service
- Event-driven architecture for notifications

---

## Glossary

- **Owner**: A user entity that has full control over a project or organization
- **Limit Entry**: A record defining the maximum allowed value for a resource
- **Limit Usage**: A record tracking current consumption of a resource
- **Transfer Request**: A pending ownership transfer awaiting approval
- **Transaction**: An atomic database operation ensuring data consistency
- **RBAC**: Role-Based Access Control system for managing permissions


---

## Related Documentation

- API Endpoints for Transfer Operations
- Database Schema Documentation
- RBAC Permissions Matrix
- Limit Configuration Guide
