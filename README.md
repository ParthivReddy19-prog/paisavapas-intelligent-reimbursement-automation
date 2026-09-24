# PAISAVAPAS.com
## Intelligent Employee Reimbursement Automation Platform

A production-oriented Django foundation for employee reimbursement automation.

### Included
- Employee registration, login, logout and dashboard
- Reimbursement submission with bill upload
- Bill metadata and optional OCR hook
- **Rule-based fraud detection engine** (`reimbursements/fraud.py`) — scores every claim 0-100 on:
  duplicate bill files, duplicate vendor+invoice combos, statistical amount anomalies vs. the
  employee's own history, "just under the auto-approval limit" threshold gaming, submission
  velocity, missing bills on high-value claims, future-dated expenses, and stale/backdated claims.
- **Automation engine** (`reimbursements/automation.py`) — runs the moment a claim is submitted:
  - Low-risk claims within the auto-approve limit and with a bill attached are **approved instantly**, no human involved.
  - High-risk claims are **auto-routed to Finance/Admin** with the specific reasons attached, skipping the normal manager queue.
  - Everything else gets a manager **auto-assigned** using department-matching + load-balancing (fewest open pending claims).
- **Fraud Alerts dashboard** for Finance/Admin listing every medium/high risk claim, worst first.
- **Scheduled automation job** (`python manage.py run_automation`) — reminds managers about claims
  stuck pending too long, and auto-escalates very old ones to Finance/Admin. Wire it to cron /
  Windows Task Scheduler / Celery beat for a real deployment.
- Manager assignment (manual override still available)
- Manager dashboard with risk filters
- Approve/reject workflow with rejection reason
- Employee reimbursement status tracking
- Notification center
- Admin management
- Reports/dashboard metrics
- Role-aware access control (fraud details are only ever visible to Manager/Finance/Admin, never the submitting employee)
- CSRF protection, validation and upload restrictions
- Responsive UI

### Quick start

```bash
python -m venv venv
# Windows
venv\Scripts\activate
# macOS/Linux
source venv/bin/activate

pip install -r requirements.txt
python manage.py migrate
python manage.py createsuperuser
python manage.py runserver
```

Open `http://127.0.0.1:8000/`.

### Manager setup

Managers can now register themselves — no admin step needed:
1. Go to `/register/` and choose **Manager** as the account type.
2. Sign in afterwards at `/manager-login/` (not the regular employee `/login/` — that page rejects manager accounts and tells them to use this one).
3. New claims are auto-assigned to managers automatically by the automation engine (least-loaded manager in the employee's department). Managers can then approve/reject from their Manager Queue.

Finance/Admin roles are still elevated privileges and are set via Django admin (`is_staff=True`, or set `EmployeeProfile.role` to `FINANCE`/`ADMIN`) since they can assign managers and see every claim company-wide.

### Fraud detection & automation — how to demo it

Tunable thresholds live in `reimbursement_platform/settings.py` (`AUTO_APPROVE_LIMIT`,
`FRAUD_HIGH_RISK_THRESHOLD`, etc.) so you can adjust them live during a viva without touching code.

To see it work:
1. **Auto-approval** — submit a small claim (e.g. ₹500, category Food) with a bill attached, no
   duplicate invoice/vendor. It should come back already `Approved` with "Auto-approved by the
   system" in the notification.
2. **Duplicate bill fraud** — submit the same bill image twice (as the same or a different
   employee). The second claim scores high and shows "Identical bill file..." in its flags.
3. **Threshold gaming** — submit a claim at ₹1,499 (just under the default ₹1,500 auto-approve
   limit). It gets flagged as a medium-risk "just under the limit" pattern instead of silently
   auto-approving.
4. **Velocity abuse** — submit 3+ claims back-to-back as the same employee within a few minutes;
   the 3rd+ claim picks up a velocity flag.
5. **Manager auto-assignment** — submit any normal claim as an employee with a `department` set on
   their profile; check the claim detail page as Finance/Admin — a manager will already be
   assigned, picked automatically by department + current workload.
6. **Fraud Alerts page** — log in as a Finance/Admin user and open "Fraud Alerts" in the nav to see
   every flagged claim ranked by score.
7. **Scheduled job** — run `python manage.py run_automation` to send reminders/escalations for
   claims that have sat pending too long.

### Notes
The project uses SQLite by default for easy local development. Uploaded bills are stored under `media/bills/`.
OCR is intentionally optional: the application can store uploaded bills and bill fields without requiring an external OCR binary. This keeps the project runnable on a fresh Windows installation.
Fraud detection is implemented as transparent, explainable rules rather than a trained ML model,
since the project has no large labeled dataset of confirmed fraud cases to train against — every
flag comes with a human-readable reason, which matters more in an approval workflow than a
black-box score.


## Manager registration
Manager registration is immediate. There is no Admin approval step for creating or activating a Manager account. A user who selects Manager during registration is assigned the MANAGER role and can access the Manager Dashboard immediately after login.

## Reimbursement approval
Pending reimbursement claims are separate from account approval. They are routed to the employee's assigned manager, who can approve or reject the claim.
