# Rachana Beauty Parlour — Production Backend Foundation

Production database, authentication, storage, and security architecture for **Rachana Beauty Parlour** (`Venkateswara Colony, Vijayapuri Colony, Uppal, Hyderabad, Telangana 500039` | Phone: `8074968435` | Hours: `11:00 AM – 8:00 PM`).

Built on **Supabase (PostgreSQL 15/16 + Supabase Auth + Supabase Storage + Row Level Security)**. Zero `localStorage`, `sessionStorage`, mock APIs, or hardcoded arrays are used for business data.

---

## 1. Project Structure

```text
rachana-beauty-parlour/
├── .env.example                                  # Template of all required public & server-only environment variables
├── .gitignore                                    # Prevents committing .env files and secrets
├── package.json                                  # Scripts & dependencies
├── supabase/
│   ├── config.toml                               # Supabase Auth (Indian SMS OTP) & Storage configuration
│   ├── migrations/
│   │   ├── 20260925000001_initial_schema.sql     # 10 core tables, enums, FKs, CHECK constraints, indexes & triggers
│   │   ├── 20260925000002_rls_policies.sql       # Row Level Security (RLS), SECURITY DEFINER admin checks & tamper guards
│   │   └── 20260925000003_storage_buckets.sql    # Storage buckets (service-images, product-images, profile-avatars) & RLS
│   └── seed.sql                                  # Reference business info & initial catalog seed data
├── src/
│   ├── config/
│   │   └── env.js                                # Strict env validation & runtime browser/server secret isolation
│   ├── lib/
│   │   ├── supabaseClient.js                     # Public/Browser-safe Supabase client (Anon Key only, RLS-enforced)
│   │   └── supabaseAdmin.js                      # Server-only Service-Role client (with browser execution guard)
│   ├── auth/
│   │   ├── customerPhoneAuth.js                  # 10-digit Indian mobile (+91) + 6-digit OTP verification architecture
│   │   └── adminRoleAuth.js                      # Separate Admin Auth + server-side & database RLS role verification
│   └── services/
│       ├── appointmentService.js                 # RLS-enforced appointment booking & admin management service
│       ├── orderService.js                       # RLS-enforced order & order_items checkout service
│       └── paymentService.js                     # Server-only Razorpay HMAC-SHA256 signature verification & capture
└── tests/
    └── verify-backend-security.mjs               # Live PostgreSQL 16 (PGlite) verification suite (25 security/RLS checks)
```

---

## 2. Environment Variables & Where to Put Them

Create a `.env.local` file in the root of `rachana-beauty-parlour/` (copied from `.env.example`) for local development, and configure the same variables in your production hosting platform's Environment/Secret Manager (e.g., Vercel / Cloud Run / Railway) as well as your **Supabase Dashboard**.

### A. Public Client Variables (Safe in `.env.local` & Frontend Bundle)
| Variable Name | Description | Where to Obtain & Place |
| :--- | :--- | :--- |
| `NEXT_PUBLIC_SUPABASE_URL` | Your Supabase project API URL (`https://<project-ref>.supabase.co`) | **Supabase Dashboard → Project Settings → API**. Place in `.env.local` and Hosting Env Vars. |
| `NEXT_PUBLIC_SUPABASE_ANON_KEY` | Public anonymous JWT key (all requests through this key are governed by PostgreSQL RLS) | **Supabase Dashboard → Project Settings → API (`anon` `public`)**. Place in `.env.local` and Hosting Env Vars. |
| `NEXT_PUBLIC_RAZORPAY_KEY_ID` | Public Razorpay Key ID (`rzp_live_...` or `rzp_test_...`) | **Razorpay Dashboard → Settings → API Keys**. Place in `.env.local` and Hosting Env Vars. |

### B. Server-Only Database & Payment Secrets (NEVER Expose to Browser)
> **CRITICAL**: Never prefix any of the variables below with `NEXT_PUBLIC_` or `VITE_`. `src/config/env.js` actively scans for accidental public prefixes and throws a fatal security violation if detected.

| Variable Name | Description | Where to Obtain & Place |
| :--- | :--- | :--- |
| `SUPABASE_SERVICE_ROLE_KEY` | Privileged key that bypasses RLS. Used **only** in server-side webhooks (`paymentService.js`) and super-admin provisioning. | **Supabase Dashboard → Project Settings → API (`service_role` `secret`)**. Place **only** in `.env.local` (server) and Hosting Server Secrets. |
| `DATABASE_URL` | Direct PostgreSQL connection string for running SQL migrations (`supabase db push`) | **Supabase Dashboard → Project Settings → Database → Connection string (URI)**. Place in `.env.local` / CI secrets. |
| `RAZORPAY_KEY_SECRET` | Secret key used on the server to verify Razorpay HMAC-SHA256 payment signatures | **Razorpay Dashboard → Settings → API Keys**. Place **only** in `.env.local` (server) and Hosting Server Secrets. |
| `RAZORPAY_WEBHOOK_SECRET` | HMAC secret used to verify incoming Razorpay webhook payloads | **Razorpay Dashboard → Settings → Webhooks**. Place **only** in `.env.local` (server) and Hosting Server Secrets. |

### C. Customer Phone OTP / Indian SMS Provider Variables
Choose either **Option 1 (Supabase Native Phone Auth with Twilio Verify)** or **Option 2 (Indian DLT Gateway: MSG91 / 2Factor via Supabase Send SMS Hook)**:

| Variable Name | Description | Where to Configure |
| :--- | :--- | :--- |
| `SMS_PROVIDER` | `'supabase_native'` \| `'twilio'` \| `'msg91'` \| `'twofactor'` | `.env.local` & Hosting Server Secrets |
| `SUPABASE_AUTH_SMS_TWILIO_ACCOUNT_SID` | Twilio Account SID (`AC...`) | **Supabase Dashboard → Authentication → Providers → Phone** (and `.env.local`) |
| `SUPABASE_AUTH_SMS_TWILIO_AUTH_TOKEN` | Twilio Auth Token Secret | **Supabase Dashboard → Authentication → Providers → Phone** (and `.env.local`) |
| `SUPABASE_AUTH_SMS_TWILIO_VERIFY_SERVICE_SID` | Twilio Verify Service SID (`VA...`) | **Supabase Dashboard → Authentication → Providers → Phone** (and `.env.local`) |
| `SMS_PROVIDER_API_KEY` | API Key for Indian DLT SMS provider (MSG91 or 2Factor.in) | `.env.local` & Hosting Server Secrets |
| `SMS_PROVIDER_SENDER_ID` | 6-character DLT Approved Header / Sender ID (e.g., `RCHNBP`) | `.env.local` & Hosting Server Secrets |
| `SMS_PROVIDER_DLT_TEMPLATE_ID` | TRAI DLT Registered OTP Template ID | `.env.local` & Hosting Server Secrets |
| `SUPABASE_AUTH_SMS_HOOK_SECRET` | Webhook signing secret if using Supabase Auth "Send SMS" Hook | **Supabase Dashboard → Authentication → Hooks (Send SMS)** & `.env.local` |

---

## 3. Applying Migrations & Bootstrapping the First Super Admin

1. **Run the SQL migrations** in order in the **Supabase SQL Editor** or via the Supabase CLI (`supabase db push`):
   - `supabase/migrations/20260925000001_initial_schema.sql`
   - `supabase/migrations/20260925000002_rls_policies.sql`
   - `supabase/migrations/20260925000003_storage_buckets.sql`
   - `supabase/seed.sql`

2. **Create the initial Super Admin**:
   First create the admin user in **Supabase Dashboard → Authentication → Users** (e.g., `owner@rachanabeautyparlour.in`), then run the following once in the Supabase SQL Editor (which executes as `postgres`):
   ```sql
   INSERT INTO public.admin_users (user_id, email, full_name, role, is_active)
   VALUES (
     '<AUTH_USER_UUID_FROM_SUPABASE_AUTH>',
     'owner@rachanabeautyparlour.in',
     'Rachana (Super Admin)',
     'super_admin',
     TRUE
   );
   ```
