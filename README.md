# Backend Custom App

`backend_custom_app` is a Frappe application that provides backend APIs and
Desk customizations for an education management portal. It is designed to run
inside a Frappe Bench alongside ERPNext and the Frappe Education app. It is not
a standalone web server or a separate frontend.

The app's Frappe metadata is defined in `backend_custom_app/hooks.py`. Its
Python package and APIs live under `backend_custom_app/`.

## Features

- **Education dashboards:** student, instructor, and staff dashboard data,
  including attendance, enrollment, teacher, and fee summaries.
- **Student and admissions workflows:** staff-facing applicant and admission
  management, student creation, and enrollment of approved applicants.
- **Attendance and schedules:** attendance data, group rosters, attendance
  marking, course schedules, and instructor-scoped access.
- **Student portal data:** student profile/context, groups, courses, grades,
  assessments, fee invoices, notifications, and leave applications.
- **Fees and payments:** invoice PDF downloads and ERPNext Payment Requests
  for a student's own outstanding Sales Invoices.
- **Notifications:** in-app student notifications for attendance, leave,
  assessment, payment, and upcoming-class events.
- **Holiday calendar:** uses the active Company's configured Frappe Holiday
  List; a limited Central Government of India reference dataset is available
  for 2026 when no configured list supplies holidays.
- **Education access setup:** migration patches configure the Education User
  role, Student academic master fields, and instructor desk permissions.
- **Server-side access controls:** student and instructor access is limited
  to their own records or assigned student groups; staff roles can manage
  school-wide data.

## Requirements

### Required applications

Install compatible versions of these applications in the same Bench before
installing this app:

| Application                       | Why it is needed                                                                    |
| --------------------------------- | ----------------------------------------------------------------------------------- |
| Frappe Framework (v16)            | Application framework, site, API, and Desk                                          |
| ERPNext (v16)                     | Company, Sales Invoice, Payment Request, and payment records                        |
| Frappe Education (v16-compatible) | Student, Instructor, attendance, enrollment, group, course, and assessment DocTypes |

This repository does not currently declare `required_apps` in its hooks, so
Bench may not automatically install these companion apps. Ensure they are
installed on the site first.

### Runtime and developer tools

- Python **3.14 or newer**, as declared by `pyproject.toml`.
- A Frappe Bench installation compatible with the versions above.
- Node.js and the package manager required by the chosen Frappe/Bench version
  if you need to build Desk assets.
- `pre-commit` for the configured formatting and lint checks.

This app does not have a `package.json`; running `npm install` in this
repository is not required to install its Python/Frappe app dependencies.

## Installation

The following commands assume that a compatible Bench and site already exist.
Install the companion apps on the Bench and site before installing this app:

```bash
cd /path/to/your-bench
bench get-app https://github.com/frappe/erpnext --branch version-16
bench get-app https://github.com/frappe/education --branch version-16
bench get-app https://github.com/erp-quantumberg/erpnext-education-backend_custom_app.git --branch version-16
bench --site your-site.example install-app erpnext
bench --site your-site.example install-app education
bench --site your-site.example install-app backend_custom_app
```

This project uses the `version-16` branches of ERPNext and Frappe Education.
Replace `your-site.example` with the site's actual name. If the repository URL
or branch changes, use the URL and branch configured for your deployment.

## Setup

1. **Install prerequisites.** Add compatible Frappe, ERPNext, and Education
   apps to the Bench and install them on the target site.
2. **Install this app** using the commands above.
3. **Run migrations** after installation or after updating the app:

   ```bash
   bench --site your-site.example migrate
   ```

   The app's post-model-sync patches create/configure the Education User role,
   add Branch/Standard/Division links to Student, grant the intended
   read-only access, seed initial academic master records, and configure
   instructor access. These patches are designed to be safe to run again.

4. **Configure the site.** Set up the Company's default Holiday List and,
   if online student payments are needed, configure an enabled ERPNext
   Payment Gateway, Payment Gateway Account, payment account, and Company.
5. **Build assets when required.** If Desk JavaScript or other bundled assets
   have changed, run:

   ```bash
   bench build --app backend_custom_app
   ```

6. **Enable scheduled jobs.** Upcoming-class reminders use a daily scheduler
   event. Ensure the Bench scheduler is enabled for the deployment.
7. **Create and link users.** Assign the appropriate education roles and link
   portal accounts to Student or Instructor records. A user without a valid
   Student/Instructor link cannot access scoped portal data.

### API overview

Frappe whitelisted methods are available through the standard
`/api/method/<dotted.python.path>` route. For example:

```text
/api/method/backend_custom_app.api.education.get_dashboard
```

Key methods include:

| Area                    | Methods                                                                                                                                                                                                                                                      |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Session and account     | `backend_custom_app.api.auth.get_csrf_token`, `backend_custom_app.api.auth.create_user`, `backend_custom_app.api.context.get_context`                                                                                                                        |
| Dashboard               | `backend_custom_app.api.education.get_dashboard`, `backend_custom_app.api.dashboard.get_admin_dashboard`                                                                                                                                                     |
| Education data          | `backend_custom_app.api.education.get_students`, `get_fees`, `get_notifications`, `get_attendance`, `get_courses`, `get_course_schedules`, `get_assessments`, `get_grades`, `get_instructors`, `get_student_groups`                                          |
| Attendance              | `backend_custom_app.api.education.get_group_attendance_roster`, `mark_attendance`                                                                                                                                                                            |
| Admissions and students | `backend_custom_app.api.admissions.get_applicants`, `get_applicant`, `get_admissions`, `get_admission`, `save_admission`, `delete_admission`, `save_applicant`, `set_applicant_status`, `enroll_applicant`; `backend_custom_app.api.students.create_student` |
| Leave and holidays      | `backend_custom_app.api.leave.get_my_groups`, `apply_leave`, `get_leave_applications`, `approve_leave`, `reject_leave`; `backend_custom_app.api.holidays.get_holidays`                                                                                       |
| Fees and notifications  | `backend_custom_app.api.payments.get_payment_status`, `create_payment_request`; `backend_custom_app.api.student_notifications.get_student_notifications`, `mark_student_notification_read`, `mark_all_student_notifications_read`                            |
| Invoices                | `backend_custom_app.api.invoices.download_invoice`                                                                                                                                                                                                           |

The API applies the caller's Frappe session and server-side role/record checks.
Use Frappe's normal authenticated request flow, including a valid CSRF token
for state-changing requests. Do not send a student ID or amount as proof of
ownership: the backend resolves the signed-in user's linked Student record
and calculates payable balances from ERPNext invoices.

## Configuration notes

- **Roles and access:** Staff roles include Administrator, System Manager,
  Education Manager, and Academics User. Student and Instructor portal access
  is scoped to the linked Student or the Instructor's assigned groups.
  Education User is a least-privilege role and is not a staff role.
- **Student linkage:** Portal data is resolved from the signed-in user and
  their Student record (`user`, with email mapping as a fallback). Instructor
  access is resolved from an active Instructor record and its assigned
  student groups.
- **Payment gateway:** Student payment endpoints use ERPNext Payment Requests.
  The amount is computed from submitted invoices and their outstanding
  balances. Creating a request does not itself mark an invoice as paid;
  ERPNext settles it after gateway confirmation.
- **Holiday source:** Configure the active Company's default Holiday List for
  institution-specific dates. The built-in 2026 list is only a reference
  fallback and includes dates that may be tentative; it does not create
  attendance or leave records.
- **Login and notifications:** Student login redirects and notification
  creation/listing are configured through Frappe hooks. Upcoming-class
  reminders require the scheduler to be running.
- **Dependencies:** App and Python dependencies are managed by Bench and
  `pyproject.toml`; this repository has no frontend npm dependency manifest.

## Troubleshooting

### App install or migration reports a missing DocType

Check that compatible ERPNext and Frappe Education apps are installed on the
same site before installing or migrating `backend_custom_app`. Then run
`bench --site your-site.example migrate` again.

### A student or instructor receives a permission error

Confirm the user's role and account status, then verify the user is linked to
an enabled Student record or an active Instructor record. For instructors,
check their Student Group assignments. Run migrations after changing app
versions or installing on a new site so the app's role and permission patches
are applied.

### Online payment is unavailable

Configure an active Company and an enabled Payment Gateway, then create a
Payment Gateway Account for that Company with a payment account. Check that
the student has submitted Sales Invoices with an outstanding balance and a
linked Customer. The payment status response includes configuration details
when the gateway setup is incomplete.

### No holidays appear

Set the Company's default Holiday List and confirm it contains holiday rows
for the requested year. Without a configured list, built-in reference data is
available only for 2026.

### Upcoming-class reminders are missing

Verify that the site scheduler is enabled and running, that relevant course
schedules exist for the next day, and that active students belong to the
corresponding student groups.

### Desk changes are not visible

Rebuild assets with `bench build --app backend_custom_app`, clear the site
cache with `bench --site your-site.example clear-cache`, and reload the Desk.

### Inspect site errors

Use the Bench logs and Frappe Error Log to diagnose failed requests and
scheduled jobs:

```bash
bench --site your-site.example logs
bench --site your-site.example console
```

## Contributing

1. Create a branch for your change and keep the change focused.
2. Follow the existing Frappe app structure and Python formatting conventions.
3. Add or update focused tests for behavior changes. Tests are located under
   `backend_custom_app/api/`.
4. Run the relevant tests on a development site:

   ```bash
   bench --site your-test-site.example run-tests --module backend_custom_app.api.test_student_notifications
   bench --site your-test-site.example run-tests --module backend_custom_app.api.test_assessment_plan_notifications
   ```

5. Install and run the repository's pre-commit hooks:

   ```bash
   cd apps/backend_custom_app
   pre-commit install
   pre-commit run --all-files
   ```

6. Update this README when setup steps, app dependencies, API behavior, or
   configuration requirements change.

## License

This project is distributed under the MIT License. See [`license.txt`](./license.txt).
