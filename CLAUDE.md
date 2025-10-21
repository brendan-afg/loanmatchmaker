# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a **static HTML loan application system** for Alt Funds Global (AFG), designed for accredited investors seeking funding matches with lenders. The entire application is built with vanilla HTML, CSS, and JavaScript—no build tools, frameworks, or package managers required.

**Deployment**: GitHub Pages (hosted at brendan-afg.github.io/loanmatchmaker)

## Architecture

### Application Flow

1. **welcome.html** → Landing page explaining the service
2. **index.html** → 6-step application form wizard with email verification
3. **upload-docs.html** → Document upload interface (receives link via email)
4. **payment.html** → Stripe payment collection page (CHF 2000)
5. **payment-success.html** → Success confirmation and matching workflow trigger

### Key Technical Patterns

**Form State Management**
- Uses `localStorage` for multi-step form persistence (key: `afg_funding_app_v1`)
- Stores current step, form data, and verification state
- Auto-saves on input with 200ms debounce

**Webhook Integration**
All forms submit to n8n webhooks at `altfundsglobal.app.n8n.cloud/webhook/`:
- `application-form-external-` - Main form submission
- `verify-email-start` - Sends verification code
- `verify-email-check` - Validates verification code
- `upload-and-process-files` - Document upload with multipart/form-data
- `create-payment-session` - Creates Stripe checkout session
- `process-matching-workflow` - Triggers lender matching

**Email Verification Flow**
- Step 1 of index.html requires email verification before proceeding
- On first "Next" click, sends verification code to user's email
- Shows verification input field and disables navigation until verified
- Verification state persists in localStorage
- Uses loading spinner during code transmission

**URL Parameter Passing**
Critical data flows between pages via URL parameters:
- `submission_id` - Unique identifier for application
- `email` - User's email address
- `full_name` - User's full name
These are extracted and pre-filled into forms using `URLSearchParams`

**Document Upload**
- Dynamic table rows for multiple documents
- Custom file input styling with visual feedback
- File descriptions limited to 30 characters
- Required fields: Government ID, Business Plan, Collateral Info, Resume
- Shows "2.3x higher likelihood" callout to encourage complete submissions
- Uses fullscreen loading overlay during upload

**Payment Flow**
- Pre-fills data from URL parameters
- Stores submission_id, email, full_name in both localStorage and sessionStorage before redirect
- Creates Stripe checkout session via webhook
- Retrieves stored data on success page to trigger matching workflow
- Clears storage after successful workflow initiation

## Backend Architecture (n8n Workflow)

The backend is powered by an **n8n workflow** that handles all server-side logic. There is no traditional application server—instead, n8n orchestrates data processing, storage, API calls, and business logic.

**Data Storage**: Google Sheets acts as the database with 7 sheets managing different aspects of the application lifecycle.

### Webhook Endpoints

All endpoints are hosted at `https://altfundsglobal.app.n8n.cloud/webhook/`:

| Endpoint | Method | Purpose | Key Request Fields | Response |
|----------|--------|---------|-------------------|----------|
| `/application-form-external-` | POST | Main form submission | All form fields + agreement data | 303 redirect to upload-docs.html |
| `/verify-email-start` | POST | Sends 6-digit verification code | `full_name`, `email`, `phone`, `company`, `industry` | `{"status": "email_sent"}` or 429 if cooldown active |
| `/verify-email-check` | POST | Validates verification code | `email`, `entered_code` | `{"status": "verified"}` or 400 with error |
| `/upload-and-process-files` | POST | Processes document uploads | `submission_id`, `email`, `full_name`, files as multipart/form-data | 303 redirect to payment.html |
| `/create-payment-session` | POST | Creates Stripe checkout session | `submission_id`, `client_email`, `full_name` | `{"success": true, "checkout_url": "..."}` |
| `/stripe-webhook` | POST | Receives Stripe events | Stripe event payload | `{"received": true}` |
| `/process-matching-workflow` | GET/POST | Runs matching algorithm | `submission_id` (query or body) | 303 redirect to view-matches page |
| `/view-matches` | GET | Displays match results as HTML | `submission_id` (query param) | HTML page with lender cards |

### Data Model (Google Sheets)

**Sheet: Submissions** (Primary application data)
- Key field: `submission_id` (UUID v4)
- Contains all form fields: contact info, funding needs, business details, location, collateral
- Special fields:
  - `agreement_hash` - SHA-256 hash of accepted user agreement
  - `folder_id` - Google Drive folder for uploaded documents
  - `status` - Tracks progression (null → `documents_received` → post-payment)
  - `verification_status` - Email verification state
  - `documents_count` - Number of files uploaded

**Sheet: Email_Verifications**
- Key field: `email`
- Fields: `verification_code` (6 digits), `verification_status`, `verification_timestamp`, `last_attempt`
- Business logic: 10-minute code expiry, 10-minute cooldown between code requests

**Sheet: Client_Uploads**
- Tracks individual uploaded files
- Fields: `submission_id`, `file_name`, `file_description`, `file_id`, `file_url`, `file_size`, `folder_id`

**Sheet: Payments**
- Key field: `payment_id` (UUID v4)
- Linked by: `submission_id`
- Fields: Stripe session/payment intent IDs, amounts, currency, status, timestamps
- Tracks: `payment_status` (pending/paid/failed), `processed_status`, `refund_status`

**Sheet: Lenders_Structured**
- Key field: `lender_id` (auto-generated: `LEND_{timestamp}_{random}`)
- Contact info: `office_name`, `website`, `contact_email`, `phone`
- Geographic data: `headquarters_country`, `regions_served_*` (boolean flags for Europe, North America, Asia, etc.)
- `specializations` - JSON array linking to Lender_Specializations

**Sheet: Lender_Specializations**
- One-to-many relationship with Lenders_Structured
- Key field: `specialization_id`
- Matching criteria: `industry_focus`, `funding_purpose`, `loan_term_preference`, `min_funding_amount`, `max_funding_amount`
- Requirements: `credit_score_min`, `collateral_required`, `geographic_preference`
- `priority_score` - Used to weight match scores (0-100)

**Sheet: Matching_Scores**
- Key field: `match_id`
- Links: `submission_id` (borrower), `lender_id` (lender)
- Scores: `overall_match_score`, `industry_match_score`, `funding_amount_score`, `geographic_score`, `loan_term_score`, `credit_score_compatibility`, `collateral_alignment`
- Metadata: `match_reasons` (human-readable), `contact_priority` (1-5 stars), `contact_status`, `outcome_status`

### Key Business Logic

**Email Verification**
- Generates 6-digit numeric code
- Expiration: 10 minutes from `verification_timestamp`
- Cooldown: Cannot request new code within 10 minutes of `last_attempt`
- Code cleared after successful verification (set to empty string)

**Matching Algorithm**
Scoring formula (applied per lender specialization):
```
overall_score = (
  industry_score * 0.25 +
  funding_amount_score * 0.25 +
  loan_term_score * 0.20 +
  credit_score * 0.15 +
  geographic_score * 0.10 +
  collateral_score * 0.05
) * (specialization_priority_score / 100)
```

Industry compatibility:
- Exact match: 100 points
- Cross-compatible industries (e.g., Tech ↔ Healthcare): 50 points
- No match: 0 points

Funding amount:
- Within min/max range: 100 points
- Within 80-120% of max: 75 points
- Outside range: 0 points

Credit score hierarchy:
- Borrower must meet or exceed lender's minimum requirement
- Levels: Excellent (720+) > Good (680-719) > Fair (620-679) > Below Fair

**Match threshold**: Only saves matches with `overall_score >= 50`
**Result limit**: Top 30 matches per borrower (sorted by score descending)

**Payment Configuration**
- Amount: CHF 2000 (200000 in cents)
- Stripe Price ID: `price_1SIxqeDmP8XfqZ4Kt74Fswem`
- Success URL: `payment-success.html?submission_id={id}&session_id={CHECKOUT_SESSION_ID}`
- Cancel URL: `payment-cancel.html?submission_id={id}`
- Eligibility: Borrower must have `status = "documents_received"` before payment session can be created

**Google Drive Document Storage**
- Parent folder ID: `1P1wq_8EjqkLLXQMQpj4dydXyhzxnHsTJ` (LoanMatch Pro)
- Folder naming: `{sanitized_email}_{sanitized_fullname}_{timestamp}`
- Sanitization: lowercase, replace @ and . with _, remove special chars, max 100 chars
- Each borrower gets a unique folder for all uploaded documents

### Status Progression

Application lifecycle tracked via `status` field in Submissions sheet:

1. **Initial submission** → `status = null`, generates `submission_id`
2. **Documents uploaded** → `status = "documents_received"`, `documents_count` updated
3. **Payment completed** → Stripe webhook updates Payments sheet
4. **Matching triggered** → `/process-matching-workflow` runs algorithm, populates Matching_Scores
5. **Results viewable** → `/view-matches?submission_id=...` generates HTML report

**Automated reminders**: Disabled Schedule trigger checks for submissions with `documents_count = 0` and `status != "documents_received"` more than 1 hour old, sends reminder emails (max 2 reminders via `num_reminders_sent`)

### submission_id Generation

Uses UUID v4 format (e.g., `a1b2c3d4-e5f6-7890-abcd-ef1234567890`):
```javascript
// Implemented in n8n "Generate submission_id" node
function uuidv4() {
  // Crypto-grade random with RFC 4122 version 4 formatting
  // Returns 36-character string with hyphens
}
```

This ID is the **primary key** linking all data across sheets and is passed between pages via URL parameters.

## Development Commands

### Local Testing
```bash
# Serve locally (any static server works)
python -m http.server 8000
# or
npx serve
```

Then navigate to `http://localhost:8000/welcome.html`

### Deployment
This repository deploys automatically via GitHub Pages. Any push to main branch updates the live site.

```bash
# Standard git workflow
git add .
git commit -m "Description of changes"
git push origin main
```

## Important Implementation Details

### CSS Variables
Consistent design system defined in `:root`:
- `--primary: #2EA3F2` (main blue)
- `--secondary: #A7144C` (accent red)
- `--radius: 12px` (card border radius)

### Form Validation
- Client-side validation on step navigation
- Error messages toggle with `.show` class
- Required fields marked with `<span class="req">*</span>`
- Email regex: `/^\S+@\S+\.\S+$/`

### Step Navigation (index.html)
- 6 total steps (indices 0-5)
- Progress bar updates as `(current / (TOTAL_STEPS - 1)) * 100`
- Step dots show: active, done, or disabled states
- Validation runs before allowing forward navigation
- Back button hidden on first step

### Agreement Loading (index.html Step 6)
- Fetches Google Docs HTML export dynamically
- Doc ID: `1P-sfYszzij1blpUVjb5fITVrunWW73la`
- Strips `<script>` tags for security
- Requires document shared with "Anyone with the link - Viewer"

### File Upload Considerations
- Uses `multipart/form-data` encoding
- File inputs use custom visual wrapper
- Shows filename after selection via change event
- Remove buttons only visible on dynamically added rows

### Loading States
- Button text changes to "Processing..." or "Uploading..."
- Buttons disabled during async operations
- Fullscreen overlays used for document upload (upload-docs.html)
- Inline spinners for email verification (index.html)

## Critical Configuration

**Webhook URLs**: All hardcoded in HTML files. When updating n8n workflows, search for:
```javascript
"https://altfundsglobal.app.n8n.cloud/webhook/"
```

**Stripe Configuration**: Payment amount (CHF 2000) is shown on payment.html but actual charge configured in n8n webhook.

**Logo URLs**: Two variations used:
- `brendan-afg.github.io/loanmatchmaker/afg-logo-secondary-reverse-m.png` (most pages)
- `altfundsglobal.com/wp-content/uploads/2024/01/afg-logo-secondary-reverse2.png` (payment.html)

## Working with This Codebase

### Adding Form Fields
1. Add HTML input in appropriate step section
2. Update validation logic in `validateStep(idx)` function
3. Ensure field `name` attribute is set for FormData collection
4. Add to required fields list if mandatory

### Modifying Webhook Endpoints
1. Find and replace webhook URL in relevant HTML file
2. Ensure n8n workflow expects same parameter names
3. Test error handling for failed requests

### Styling Changes
1. Modify CSS variables in `:root` for global changes
2. Follow existing class naming patterns (BEM-like)
3. Test responsiveness with `@media` queries already defined

### Testing Verification Flow
- Email verification requires working n8n webhook
- Mock by skipping verification: set `isVerified = true` in localStorage
- Remember to test with actual email delivery before production

### Testing with n8n Backend
**Understanding the integration:**
- Frontend HTML files are standalone but depend on n8n webhooks for all data operations
- No backend code exists in this repository—all server logic is in the n8n workflow
- Data flows: HTML form → n8n webhook → Google Sheets → n8n processing → response back to HTML

**Testing approaches:**
1. **Full integration testing**: Requires access to the live n8n workflow at `altfundsglobal.app.n8n.cloud`
2. **Mock testing**: Modify webhook URLs to point to local mock server or testing endpoints
3. **Partial testing**: Test UI/UX without backend by commenting out `fetch()` calls

**Key testing scenarios:**
- Email verification cooldown (10 min) - check `Email_Verifications` sheet for `last_attempt`
- File upload limits (max 10 files) - validation happens in n8n
- Payment eligibility - borrower must have `status = "documents_received"`
- Match generation - requires populated `Lenders_Structured` and `Lender_Specializations` sheets

**Debugging tips:**
- Check Google Sheets directly to see data state
- Use browser DevTools Network tab to inspect webhook requests/responses
- n8n workflow execution history shows detailed logs for each node
- `submission_id` is the key to trace data across all sheets

## Browser Compatibility

- Uses modern JavaScript (async/await, arrow functions, URLSearchParams)
- CSS Grid for layouts
- `fetch` API for all HTTP requests
- No polyfills included—targets modern browsers (Chrome, Firefox, Safari, Edge)
