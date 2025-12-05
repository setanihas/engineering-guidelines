# Transfer Flow Documentation


## Overview

Two parallel transfer flows are documented:
1. **Organization Ownership Transfer Flow**
2. **Project/Assignment Transfer Flow**

Both flows share the same core logic and business rules, with minor variations in module triggers and recipient types.

## Flow Diagrams

The diagrams are created using Mermaid and illustrate the complete transfer process from initiation to completion.

### Key Components

#### Actors
- **Requester**: Initiates the transfer request
- **Current Owner**: Approves or declines the transfer
- **New Owner / Target User**: Receives and accepts the transfer
- **System**: Handles validation, checks, and processing
- **Triggers**: Module-level triggers (Organization Module or Resource Module)
- **Payment & Limits Service**: Manages payment processing and limit verification

## Transfer Flow Process

### 1. Initiation Phase
- Requester initiates a transfer request
- System validates requester's permission
- If unauthorized, returns permission error and ends process

### 2. Approval Phase
- System sends approval email to current owner
- Current owner reviews and approves/declines
- If declined, process ends
- If approved, proceeds to next phase

### 3. Database & Trigger Phase
- System inserts record into `transfer_ownership` bucket
- Module-level trigger fires:
  - **Organization Module Trigger** (for organization transfers)
  - **Resource Module Trigger** (for project/assignment transfers)
- Trigger sends notification email to recipient using appropriate template:
  - `org_transfer_notification` for organizations
  - `resource_transfer_notification` for projects/assignments

### 4. Recipient Review Phase
- Recipient receives email with in-app link
- Recipient clicks link to access in-app page
- System performs first limit check

### 5. Limit Verification & Payment (if needed)
- **If limits are insufficient:**
  - System displays minimum required plan
  - Initiates quick-payment flow
  - Processes payment through Payment & Limits Service
  - If payment fails, process ends
  - If payment succeeds, continues to display phase

- **If limits are sufficient:**
  - Proceeds directly to display phase

### 6. Information Display Phase
- System displays detailed information:
  - Current usage
  - Limits that will increase after transfer
  - New total limits after transfer
- Recipient reviews information

### 7. Acceptance Phase
- Recipient clicks "Accept Transfer"
- System calls `accept-transfer` endpoint

### 8. Final Validation & Execution
- System performs **second limit check** (safety measure)
- If insufficient, returns error and ends process
- If sufficient:
  - Deducts used limits from old owner
  - Applies limits to new owner/target user
  - Updates `transfer_ownership` entry with status: `completed/success`
  - Transfer complete

## Key Differences Between Flows

| Aspect | Organization Transfer | Project/Assignment Transfer |
|--------|----------------------|----------------------------|
| **Module Trigger** | Organization Module | Resource Module |
| **Email Template** | `org_transfer_notification` | `resource_transfer_notification` |
| **Recipient** | Target Organization Owner | Target User |
| **Limit Application** | Applied to Organization | Applied to Individual User |

## Technical Implementation Notes

### Database
- **Bucket**: `transfer_ownership`
- **Record Status**: `completed/success` upon successful transfer

### API Endpoints
- **accept-transfer**: Handles final acceptance and limit transfer logic

### Email Templates
- `org_transfer_notification`: Used for organization transfers
- `resource_transfer_notification`: Used for project/assignment transfers

### Security & Validation
- **Permission Check**: At initiation (requester must have transfer permission)
- **Approval Required**: Current owner must approve
- **Double Limit Check**: 
  1. Before displaying acceptance page
  2. Inside accept-transfer endpoint (safety measure)

### Limit Management
- Limits are deducted from the old owner
- Same limits are applied to the new owner/target user
- Payment flow integrates automatically for insufficient limits

## Error Handling

The flow includes multiple exit points for error scenarios:
- Permission denied at initiation
- Transfer declined by current owner
- Payment failure
- Insufficient limits at final check

## Mermaid Diagrams

### Organization Ownership Transfer Flow

```mermaid
graph TB
    subgraph ORG[" "]
        direction TB
        A1[Requester: Initiate Organization Transfer Request]
        A2{System: Check Requester Permission}
        A3[System: Return Permission Error]
        A4[System: Send Approval Email to Current Owner]
        A5[Current Owner: Receives Approval Email]
        A6{Current Owner: Approve Transfer?}
        A7[System: Insert Record into transfer_ownership Bucket]
        A8[Triggers: Organization Module Trigger Fires]
        A9[Triggers: Send Email to Target Org Owner<br/>Template: org_transfer_notification]
        A10[New Owner: Receives Transfer Email with In-App Link]
        A11[New Owner: Clicks Link to In-App Page]
        A12{System: Check Target Owner Limits}
        A13[System: Display Minimum Required Plan<br/>Start Quick-Payment Flow]
        A14[Payment & Limits: Process Payment]
        A15{Payment Success?}
        A16[System: Display Limit Details<br/>- Current usage<br/>- Limits that will increase<br/>- New total after transfer]
        A17[New Owner: Review & Click Accept Transfer]
        A18[System: Call accept-transfer Endpoint]
        A19{System: Second Limit Check}
        A20[System: Return Insufficient Limits Error]
        A21[System: Deduct Used Limits from Old Owner<br/>Apply Limits to New Owner]
        A22[System: Update transfer_ownership Entry<br/>Status: completed/success]
        A23[System: Transfer Complete]
        A24[Process Ends]

        A1 --> A2
        A2 -->|No Permission| A3
        A3 --> A24
        A2 -->|Has Permission| A4
        A4 --> A5
        A5 --> A6
        A6 -->|Decline| A24
        A6 -->|Approve| A7
        A7 --> A8
        A8 --> A9
        A9 --> A10
        A10 --> A11
        A11 --> A12
        A12 -->|Limits Insufficient| A13
        A13 --> A14
        A14 --> A15
        A15 -->|Failed| A24
        A15 -->|Success| A16
        A12 -->|Limits Sufficient| A16
        A16 --> A17
        A17 --> A18
        A18 --> A19
        A19 -->|Insufficient| A20
        A20 --> A24
        A19 -->|Sufficient| A21
        A21 --> A22
        A22 --> A23
        A23 --> A24
    end
```

### Project/Assignment Transfer Flow

```mermaid
graph TB
    subgraph PROJ[" "]
        direction TB
        B1[Requester: Initiate Project Transfer Request]
        B2{System: Check Requester Permission}
        B3[System: Return Permission Error]
        B4[System: Send Approval Email to Current Owner]
        B5[Current Owner: Receives Approval Email]
        B6{Current Owner: Approve Transfer?}
        B7[System: Insert Record into transfer_ownership Bucket]
        B8[Triggers: Resource Module Trigger Fires]
        B9[Triggers: Send Email to Target User/Org<br/>Template: resource_transfer_notification]
        B10[Target User: Receives Transfer Email with In-App Link]
        B11[Target User: Clicks Link to In-App Page]
        B12{System: Check Target User Limits}
        B13[System: Display Minimum Required Plan<br/>Start Quick-Payment Flow]
        B14[Payment & Limits: Process Payment]
        B15{Payment Success?}
        B16[System: Display Limit Details<br/>- Current usage<br/>- Limits that will increase<br/>- New total after transfer]
        B17[Target User: Review & Click Accept Transfer]
        B18[System: Call accept-transfer Endpoint]
        B19{System: Second Limit Check}
        B20[System: Return Insufficient Limits Error]
        B21[System: Deduct Used Limits from Old Owner<br/>Apply Limits to Target User]
        B22[System: Update transfer_ownership Entry<br/>Status: completed/success]
        B23[System: Transfer Complete]
        B24[Process Ends]

        B1 --> B2
        B2 -->|No Permission| B3
        B3 --> B24
        B2 -->|Has Permission| B4
        B4 --> B5
        B5 --> B6
        B6 -->|Decline| B24
        B6 -->|Approve| B7
        B7 --> B8
        B8 --> B9
        B9 --> B10
        B10 --> B11
        B11 --> B12
        B12 -->|Limits Insufficient| B13
        B13 --> B14
        B14 --> B15
        B15 -->|Failed| B24
        B15 -->|Success| B16
        B12 -->|Limits Sufficient| B16
        B16 --> B17
        B17 --> B18
        B18 --> B19
        B19 -->|Insufficient| B20
        B20 --> B24
        B19 -->|Sufficient| B21
        B21 --> B22
        B22 --> B23
        B23 --> B24
    end
```

## Usage

To view these diagrams:
1. Use any Mermaid-compatible viewer
2. GitHub automatically renders Mermaid diagrams in markdown files
3. Or use these tools:
   - [Mermaid Live Editor](https://mermaid.live/)
   - VS Code with Mermaid extension
   - Markdown preview tools
