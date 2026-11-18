# CS2 Admin System

A Django-based academic administration backend for managing Computer Science 2 (CS2) student records, analysing performance, handling student requests, and producing downloadable reports.

The project exposes HTTP endpoints for a separate frontend and includes Django's administrative interface.

> **Project status:** Academic project / development codebase. Review the [Security and deployment notes](#security-and-deployment-notes) before using it with real student information or deploying it publicly.

## Features

### Student and course administration
- Store students' numbers, names, CS1 grades and CS2 grades.
- Import student records and grade data from text or spreadsheet files.
- Look up individual students and compare their CS1 and CS2 results.
- Categorise students by their CS2 assignment averages to identify those who may need academic support.

### Student requests
- Submit requests with supporting files, a motivation and relevant dates.
- Retrieve requests submitted by a student.
- View and filter requests by status.
- Update request decisions and reasons, and delete requests.
- Export approved requests for administrative adjustments.

### Reporting
- Download Excel reports showing students in need of support.
- Generate a DP/DPR report based on CS2 assignment averages.
- Export approved extension/adjustment requests.
- Trigger an email summary of pending student requests.

### Authentication and administration
- Register users and log in through API endpoints.
- Issue authentication tokens using Django REST Knox in the login and registration views.
- Log out and access user details through the supplied endpoints.
- Use Django Admin with Jazzmin styling.

## Tech stack

| Component | Technology |
| --- | --- |
| Backend | Python, Django 4.1.1 |
| API | Django REST Framework 3.13.1 |
| Database | SQLite (repository's development configuration) |
| Authentication | Django authentication, Django REST Knox |
| Admin interface | Django Admin, Jazzmin |
| File ingestion | Text files and spreadsheets via `xlrd` |
| Report generation | XlsxWriter |
| Cross-origin support | django-cors-headers |
| Notifications | Django email utilities |

Dependencies and pinned versions are in [requirements.txt](requirements.txt).

## Repository structure

```text
cs2-admin-system/
├── analysis/           # Student performance calculations and Excel reports
├── authentication/     # Registration, login and token endpoints
├── cs2admin/           # Django project settings, routes and templates
├── database/           # Student, grade and request models
├── sample data/        # Example student and grade import files
├── uploadfile/         # File imports, request processing and reports
├── manage.py           # Django management CLI
├── requirements.txt    # Python dependencies
└── README.md
```

The repository also contains a development SQLite database and collected static/media files. These should be reviewed before sharing or deploying, particularly if they contain student information.

## Getting started

### Prerequisites

- Python and pip (the pinned dependencies are from the Django 4.1 era; a Python 3.10 or 3.11 environment is a practical starting point).
- Git.
- A local environment for running Django.

### 1. Clone the repository

```bash
git clone https://github.com/matimba-maholobela/cs2-admin-system.git
cd cs2-admin-system
```

### 2. Create and activate a virtual environment

```bash
python3 -m venv .venv
source .venv/bin/activate
```

On Windows PowerShell, activate with `.venv\Scripts\Activate.ps1`.

### 3. Install dependencies

```bash
python -m pip install -r requirements.txt
```

### 4. Initialise the database

```bash
python manage.py migrate
python manage.py createsuperuser
```

The project is currently configured to use the local `db.sqlite3` database.

### 5. Start the development server

```bash
python manage.py runserver
```

- API base URL: `http://127.0.0.1:8000/`
- Django Admin: `http://127.0.0.1:8000/admin/`

> **Configuration caveat:** The checked-in settings include static-file directories that may not exist in a fresh clone, and some email settings are placeholders. If Django reports a missing static directory, review `STATICFILES_DIRS` in `cs2admin/settings.py`. Configure a valid email backend before using notification endpoints.

## API overview

The following routes are defined in [cs2admin/urls.py](cs2admin/urls.py) and [authentication/urls.py](authentication/urls.py). Paths are relative to the local server. Some endpoints expect JSON or multipart form data, while others accept query parameters.

| Method | Endpoint | Purpose |
| --- | --- | --- |
| POST | `/api/register/` | Create a user and return a token |
| POST | `/api/login/` | Log in and return a token |
| GET | `/api/user/` | Get the current user's details |
| POST | `/api/logout/` | Log out |
| POST | `/api/logoutall/` | Revoke all Knox tokens for a user |
| GET | `/student-dp/?username=...` | Get a student's CS2 DP average and status |
| GET | `/searchstudent/?username=...` | Look up CS1 and CS2 grades |
| GET | `/allstudents/` | Group students by performance |
| GET, POST | `/request/` | View a student's requests or submit a request |
| GET, POST | `/request-view/` | List all requests or update a request's status |
| GET | `/request-filter?status=Pending` | Filter requests by status |
| POST | `/deleteRequest/` | Delete a request by ID |
| POST | `/addstudents/` | Import student records |
| POST | `/loadcurrentgrades/` | Import CS2 grades |
| POST | `/loadcs1grades/` | Import CS1 grades |
| GET | `/needsupport/` | Download the support-needs spreadsheet |
| GET | `/releasedp/` | Download the DP/DPR spreadsheet |
| GET | `/adjustment/` | Download approved requests |
| GET | `/report/` | Email a pending-request summary |

The authentication routes are also registered at the root (`/login/`, `/register/`, etc.). There is a registration page at `/home/`.

**Important:** This table documents the registered routes, not a security guarantee. The project currently needs an authentication/authorization audit before production use.

### Example: student DP lookup

```bash
curl "http://127.0.0.1:8000/student-dp/?username=STUDENT_NUMBER"
```

Example response shape (illustrative only):

```json
{
  "status": "Good",
  "dp": 55.0,
  "message": "You can visit the Course Convenor for support."
}
```

### Example: request status filter

```bash
curl "http://127.0.0.1:8000/request-filter?status=Pending"
```

## Academic performance rules

The student DP endpoint calculates the mean of six CS2 assignment marks and classifies the result as follows:

| Assignment average | Student-facing status |
| --- | --- |
| 45 or below | Critical |
| Above 45 and up to 60 | Good |
| Above 60 | Excellent |

The downloadable DP report currently labels averages **below 45** as `DPR` and averages **45 or above** as `DP`. This differs at the boundary from the student-facing classification; verify institutional requirements before relying on these thresholds.

## Sample data and imports

Example files are available in [sample data](sample%20data/). The import handlers in `uploadfile/read.py` load student data and CS1/CS2 grades from text files or spreadsheets.

For grade imports, records are expected to contain a student number, two test marks and six assignment marks. Import student records before importing grades, because grades reference existing students.

> The current Excel reader uses `xlrd==1.2.0`, an older dependency, for `.xlsx` files. Compatibility and input validation should be checked before using new files or newer Python environments. Do not assume arbitrary uploads are safe.

## Data models

The central Django models in [database/models.py](database/models.py) are:

- **Student** — student number, first name and surname.
- **CurrentGrade** — a student's CS2 test and assignment marks.
- **cs1Grade** — a student's CS1 test and assignment marks.
- **StudentForm** — a student request, including request type, uploaded file, motivation, dates, status and decision reason.
- **MainData** — uploaded files used for bulk imports.

## Security and deployment notes

**The repository is not production-ready in its current configuration.** Before handling real academic records or exposing the app publicly:

1. Remove the committed Django secret key, rotate it, and load a new value from environment variables or a secrets manager.
2. Set `DEBUG=False`, restrict `ALLOWED_HOSTS`, and replace the permissive CORS configuration with approved origins.
3. Revisit API permissions, token authentication configuration, ownership checks and role-based access for students and administrators. The settings currently define `REST_FRAMEWORK` twice, meaning the later definition overrides the earlier one.
4. Stop sending passwords by email during registration; use a secure account activation or password-reset process instead.
5. Secure uploads, validate file types and sizes, and protect access to supporting documents and student records.
6. Move credentials out of source control, configure SMTP securely and review the pending-request notification recipient.
7. Remove sensitive/generated files from version control as appropriate, including databases, uploaded media and Python bytecode; assess whether existing history contains data that needs remediation.
8. Upgrade and test old dependencies, and run Django's deployment checks (`python manage.py check --deploy`) as part of the release process.

These are recommended changes, **not** fixes performed by this README update.

## Contributing

Contributions and improvements are welcome. Open an issue to discuss a bug or enhancement, then submit a pull request with a clear description and testing notes.

## Maintainer

Maintained in [matimba-maholobela/cs2-admin-system](https://github.com/matimba-maholobela/cs2-admin-system).
