# ShiShi — micro-lending platform (v2)

Flask application for a Nigerian micro-lender: public site with a live repayment
calculator, customer app (KYC, applications, repayments, wallet, statements,
support) and a back office (underwriting, disbursement, collections, reports,
audit). Rebuilt from the original ShiShi codebase; see *What changed* below.

## Quick start (development)

```bash
python -m venv venv && source venv/bin/activate      
pip install -r requirements.txt
cp .env.example .env                                 
flask db upgrade                                     
flask seed-products                                  
flask seed-demo                                      
flask run --port 5700
```

Demo logins (password `Passw0rd!`, development only):

| Email | Role | What you'll see |
|---|---|---|
| admin@demo.shishi.ng | Administrator | Everything, incl. products, staff, audit log |
| officer@demo.shishi.ng | Loan officer | Queue, underwriting, collections, support |
| chioma@demo.shishi.ng | Customer | One loan repaid, one running on time |
| musa@demo.shishi.ng | Customer | Running loan ~40 days late with late fees |
| bisi@demo.shishi.ng | Customer | New account, KYC not started |

Create a real administrator with `flask create-admin`.

### Using MySQL (your original database)

```bash
# create a dedicated user; don't run the app as passwordless root
mysql -u root -p -e "CREATE DATABASE shishidb_v2 CHARACTER SET utf8mb4; CREATE USER 'shishi_app'@'localhost' IDENTIFIED BY 'STRONG_PASSWORD'; GRANT ALL ON shishidb_v2.* TO 'shishi_app'@'localhost';"
# in .env
DATABASE_URL=mysql+mysqlconnector://shishi_app:STRONG_PASSWORD@localhost/shishidb_v2
flask db upgrade
```

Use a **new** database. The schema changed substantially (money types, KYC tables,
loan snapshot columns), and the old migration history didn't match the old models.
The migration was generated and verified on SQLite; run `flask db upgrade` against a
throwaway MySQL database first. If you have real data in the old `shishidb`, it
needs a one-off data migration script rather than an in-place upgrade.

Delete or rename the old `instance/config.py`: it hard-codes the old database URL and
would override `.env`.

### Scheduled job

Run daily (cron, Windows Task Scheduler, or your host's scheduler):

```bash
flask loans process-overdues
```

It marks installments overdue after the grace period, charges the one-time late fee,
and moves loans to *defaulted* after `DEFAULT_AFTER_DAYS`. Staff can also trigger it
from the back office.

### Tests

```bash
pytest -q
```

12 end-to-end tests cover the amortisation maths, the full lifecycle (register → KYC →
apply → verify → approve → four-eyes disbursement → partial, wallet and full
repayment → closure), overdue penalties and defaults, access control, upload
validation, login lockout, password reset and support tickets.

## Modules

| Area | Customer | Back office |
|---|---|---|
| Identity (KYC) | BVN/NIN, address, income, documents, payout bank account; submit for review | Verify or reject with a note; name-mismatch warning on bank account |
| Applications | Live quote beside the form; key-facts review (APR, total cost, full schedule) before submitting; cancel | Queue, scorecard with reasons, approve/decline with reason |
| Disbursement | Notified with first due date | To wallet or record bank transfer; approver can't disburse (four-eyes) |
| Repayment | Pay from wallet (next installment, any amount, or pay off) | Record transfers/cash with bank reference; oldest installment first |
| Collections | Overdue notifications, late fee shown per installment | Buckets (this week, 1–30, 31–90, 90+) with phone numbers and guarantor |
| Wallet | Top up (sandbox gateway), balance, history | Transaction ledger |
| Statements | Filter by date/type, CSV export | Loan CSV export |
| Support | Tickets threaded by loan | Reply, close; website contact messages |
| Reports | — | Principal outstanding, PAR30/PAR90, collection rate, monthly disbursed vs collected, by-product approval and arrears |
| Admin | — | Products (terms apply to new loans only), staff roles, audit log, account suspension |

### Loan maths

`pkg/services/loan_engine.py` is pure Python with no database access. Rates are
quoted **per month**. *Reducing balance* uses standard amortisation; *flat* charges
interest on the original principal every period. APR is the annualised internal rate
of return on the **net** amount received (principal minus fee), so it includes the
processing fee. All money is `Decimal`, rounded half-up to kobo; the last installment
absorbs rounding so schedules sum exactly.

The public calculator, the application form and the contract all call the same
engine through `/api/quote`, so the figure a customer sees is the figure they sign.

## Before going live — not done in this codebase

These need decisions, contracts or licences, not just code:

1. **Licensing and disclosures.** Confirm your licence and registration position with
   the CBN and FCCPC before lending, and have a lawyer review the key-facts sheet,
   `terms.html` and `privacy.html` (both marked as drafts) against current consumer
   lending and data-protection (NDPA 2023) rules. Only display regulator logos you are
   entitled to use; the original home page claimed CBN licensing and NDIC insurance.
2. **Identity verification.** BVN/NIN are format-checked only. Integrate an approved
   verification provider and record the result on `KycProfile`.
3. **Credit bureau.** `services/credit.py` is a transparent scorecard, not a bureau
   check. Add CRC / FirstCentral / CreditRegistry results as extra factors.
4. **Payments.** `PAYMENT_PROVIDER=sandbox` simulates checkout. Replace
   `customer.checkout` with a Paystack/Flutterwave/Monnify redirect and call
   `ledger.confirm_funding()` **only** from a signature-verified webhook. Bank
   disbursement is recorded manually; automate with a transfer API when ready.
5. **Email/SMS.** Password-reset links are logged (and shown on screen in development).
   Plug a provider into `services/notify.py` and `auth.forgot_password`.
6. **Pricing.** Seeded product rates, fees and limits are illustrative. Set them from
   your cost of funds, expected losses and compliance review via *Loan products*.
7. **Operations.** Serve with `gunicorn "starter:app"` behind HTTPS, set
   `FLASK_ENV_NAME=production` (enforces a real `SECRET_KEY` and secure cookies), add
   rate limiting at the proxy, back up the database, and move uploads to private
   object storage.

## What changed from the original, and why

Verified by running the original code, not just reading it:

| Original behaviour | Cause | Now |
|---|---|---|
| Home page, /product/ and admin index returned HTTP 500 | Trailing space in `{% include "user/header.html "%}` | Single base layout with inheritance |
| Registration crashed after saving the user | `url_for('user_login')` missing blueprint prefix (same bug on all admin pages) | Blueprint-qualified endpoints throughout; every page covered by tests |
| ₦60,000 over 6 months created **one** ₦10,000 installment | `db.session.add(repayment)` outside the loop | Full schedule generated at disbursement |
| No interest charged | `product_interest_rate` never used; installment = amount ÷ n | Amortisation engine with APR |
| Loans went straight to `active` | No approval step | pending → in review → approved → disbursed → repaid/defaulted |
| Applying crashed | Seeded products had no duration (`range(1, None + 1)`), which also blocked the product-seeding route | Complete product seed + validation |
| Dashboard crashed after first loan | Template read `loan.created_at`, which the model didn't have | Model and templates aligned |
| Secret key and DB root URL in source | `instance/config.py` committed | Environment variables, `.env.example`, `.gitignore` |
| Admin pages open to anyone; `Admin` passwords unhashed | No access control | Role-based staff users, hashed passwords, audit log |
| KYC files saved under public `/static/uploads` | Upload path | Private folder, random names, content-type check, owner/staff-only download |
| Money as `Float` | Column type | `Numeric(14,2)` + `Decimal` |
| 226 MB of images; 8.9 MB background on every page | Unoptimised photos, duplicates | 1.2 MB, resized; unused images removed |
| `requirements.txt` was a full Anaconda dump; Windows venv committed | Environment export | Minimal pinned requirements |
# ShiShi-Loan-Application
