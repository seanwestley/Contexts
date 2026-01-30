# Plan: Replace DocuSign Workflow with Zapier Webhook

## Overview

This plan will replace the current DocuSign Maestro workflow integration with a Zapier webhook submission. The frontend form remains unchanged - only the backend submission logic will be modified to POST all form data to a configurable Zapier webhook URL.

---

## Current Architecture

```text
+------------------------+     +---------------------------+     +-------------------+
|  InvestmentFormModal   | --> | initiate-investment       | --> | DocuSign Maestro  |
|  (9-step form)         |     | (Edge Function)           |     | Workflow API      |
+------------------------+     +---------------------------+     +-------------------+
                                        |
                                        v
                               +-------------------+
                               | subscription_logs |
                               | (tracking table)  |
                               +-------------------+
```

## New Architecture

```text
+------------------------+     +---------------------------+     +------------------+
|  InvestmentFormModal   | --> | initiate-investment       | --> | Zapier Webhook   |
|  (9-step form)         |     | (Edge Function - modified)|     | (Your Zap)       |
+------------------------+     +---------------------------+     +------------------+
                                        |
                                        |  Includes:
                                        |  - All form data (70+ fields)
                                        |  - Submitting user info
                                        |  - User's organization
                                        |  - Timestamp
                                        |
                                        v
                               +-------------------+
                               | subscription_logs |
                               | (tracking table)  |
                               +-------------------+
```

---

## Implementation Steps

### Step 1: Add Zapier Webhook Secret

Add a new secret to store the Zapier webhook URL:
- **Secret Name**: `ZAPIER_INVESTMENT_WEBHOOK_URL`
- **Value Format**: `https://hooks.zapier.com/hooks/catch/XXXX/YYYY/`

This allows you to update the webhook URL without code changes.

---

### Step 2: Modify Edge Function

**File**: `supabase/functions/initiate-investment/index.ts`

#### 2a. Remove DocuSign Dependencies

Remove:
- JWT Grant token generation
- DocuSign API calls (Maestro workflow trigger)
- DocuSign configuration checks
- OAuth token management

#### 2b. Add Zapier Webhook Logic

Replace the DocuSign workflow trigger with a simple POST to Zapier:

```typescript
// Fetch webhook URL from secrets
const webhookUrl = Deno.env.get("ZAPIER_INVESTMENT_WEBHOOK_URL");
if (!webhookUrl) {
  return errorResponse("Webhook not configured", 503);
}

// Build payload with all form data + submitter info
const payload = {
  // Hidden submitter info
  submittedBy: {
    userId: user.id,
    email: user.email,
    firstName: submitterProfile.first_name,
    lastName: submitterProfile.last_name,
    organizationId: submitterProfile.organization_id,
    organizationName: organization?.name,
    roles: userRoles,
    submittedAt: new Date().toISOString(),
  },
  
  // Client info
  client: {
    id: clientId,
    name: clientName,
    email: clientEmail,
    investorClass,
  },
  
  // Strategy/Fund info
  strategy: {
    id: strategy.id,
    name: strategy.name,
    type: strategy.type,
  },
  
  // Complete form data (all 70+ fields)
  formData: formData,
  
  // Primary investment details
  primaryAmount: amount,
  totalAmount: formData?.allocations?.reduce((sum, a) => sum + a.amount, 0) || amount,
};

// POST to Zapier webhook
const response = await fetch(webhookUrl, {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify(payload),
});
```

---

### Step 3: Hidden Submitter Information

The webhook payload will automatically include the following hidden fields about the logged-in user submitting the form:

| Field | Source | Description |
|-------|--------|-------------|
| `submittedBy.userId` | `auth.users.id` | Unique user ID |
| `submittedBy.email` | `auth.users.email` | User's email address |
| `submittedBy.firstName` | `profiles.first_name` | User's first name |
| `submittedBy.lastName` | `profiles.last_name` | User's last name |
| `submittedBy.organizationId` | `profiles.organization_id` | User's organization UUID |
| `submittedBy.organizationName` | `organizations.name` | Organization name (joined) |
| `submittedBy.roles` | `user_roles` | Array of roles (admin, advisor, etc.) |
| `submittedBy.investorType` | `profiles.investor_type` | direct or ria_associated |
| `submittedBy.investorClass` | `profiles.investor_class` | QP, AI, etc. |
| `submittedBy.advisorCrd` | `profiles.advisor_crd` | Advisor CRD number |
| `submittedBy.submittedAt` | Generated | ISO timestamp of submission |

---

### Step 4: Update Response Handling

The edge function will:
1. Check webhook response status
2. Log the submission in `subscription_logs` with status `webhook_sent`
3. Return success/failure to the frontend

Frontend changes are minimal - just update success message text (optional).

---

## Complete Webhook Payload Structure

```json
{
  "submittedBy": {
    "userId": "db399136-b5de-48be-a738-b3f252a89e7c",
    "email": "advisor@example.com",
    "firstName": "Tarun",
    "lastName": "Kumar",
    "organizationId": "2aa2e571-974c-4724-8bbb-a198e668fba1",
    "organizationName": "Prism Wealth Advisors",
    "roles": ["advisor", "admin"],
    "investorType": "ria_associated",
    "investorClass": null,
    "advisorCrd": "123456",
    "submittedAt": "2026-01-30T05:00:00.000Z"
  },
  "client": {
    "id": "fd98d7a0-4cc3-43eb-b21a-b22afa0e298b",
    "name": "Gordon Gekko",
    "email": "gordon@example.com",
    "investorClass": "QP"
  },
  "strategy": {
    "id": "12969787-4c63-4622-bcba-49eaf0350ac0",
    "name": "KPC L/S Activist",
    "type": "long/short equity"
  },
  "primaryAmount": 500000,
  "totalAmount": 500000,
  "formData": {
    "transactionType": "New_Investment",
    "investorType": "Individual",
    "individualFirstName": "Gordon",
    "individualEmail": "gordon@example.com",
    "street": "123 Wall Street",
    "city": "New York",
    "state": "NY",
    "allocations": [...],
    "custodian": "Schwab",
    "...all other form fields..."
  }
}
```

---

## Files to Modify

| File | Change |
|------|--------|
| `supabase/functions/initiate-investment/index.ts` | Replace DocuSign logic with Zapier webhook POST |

---

## Files NOT Modified (Frontend Unchanged)

- `src/components/advisor/investment-form/InvestmentFormModal.tsx` - No changes
- `src/components/advisor/investment-form/types.ts` - No changes
- All step components - No changes

---

## Technical Notes

1. **Secret Management**: The webhook URL is stored as a secret, making it easy to update without code changes

2. **Error Handling**: If Zapier returns an error, the function will return an error to the frontend with a clear message

3. **Logging**: All submissions are logged to `subscription_logs` for audit trail

4. **No-CORS Not Needed**: Since the webhook call happens server-side (edge function), there are no CORS issues

5. **Zapier Response**: Zapier webhooks typically return `{ "status": "success" }` on success

---

## Security Considerations

- Webhook URL is stored as a secret (not in code)
- Only authenticated users with `admin` or `advisor` roles can trigger submissions
- All user context is verified server-side from the JWT token
