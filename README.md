# Smart Student Skill Exchange & Peer Learning Management System

A full-stack web application where students teach what they know and learn what they want. Built with **Flask** and **SQLite**, it matches students with complementary skills, lets them schedule peer-learning sessions, and builds trust through ratings and reviews.

---

## Table of Contents

- [Features](#features)
- [Tech Stack](#tech-stack)
- [Project Structure](#project-structure)
- [Getting Started](#getting-started)
- [Usage Guide](#usage-guide)
- [How Matching Works](#how-matching-works)
- [Session Lifecycle](#session-lifecycle)
- [Database Schema](#database-schema)
- [Routes](#routes)
- [Security Notes](#security-notes)
- [Configuration](#configuration)
- [Troubleshooting](#troubleshooting)
- [Future Improvements](#future-improvements)

---

## Features

- **Authentication**: registration and login with hashed passwords (Werkzeug)
- **Student profiles**: name, bio, skills offered, and skills wanted
- **Smart matching**: ranks other students by two-way skill compatibility
- **Session requests**: ask a peer to teach you a skill at a chosen date and time
- **Workflow management**: accept, reject, complete, or cancel sessions
- **Ratings & reviews**: 1-5 star feedback after a session is completed
- **Dashboard**: overview of your skills and upcoming pending sessions
- **Auto-initialised database**: SQLite tables are created on first run
- **Responsive UI**: works on desktop and mobile

## Tech Stack

| Layer     | Technology                         |
|-----------|------------------------------------|
| Backend   | Python 3.10+, Flask 3.1.2          |
| Database  | SQLite (via the built-in `sqlite3`)|
| Templates | Jinja2                             |
| Security  | Werkzeug password hashing, Flask sessions |

## Project Structure

```
skill-exchange/
├── app.py              # Flask app: routes, DB setup, business logic
├── requirements.txt    # Python dependencies
├── README.md           # Project documentation
├── skill_exchange.db   # SQLite database (auto-created on first run)
├── templates/          # Jinja2 HTML templates
│   ├── index.html
│   ├── register.html
│   ├── login.html
│   ├── profile.html
│   ├── dashboard.html
│   ├── matches.html
│   ├── request_session.html
│   ├── requests.html
│   └── review.html
├── static/             # CSS, JavaScript, images
└── venv/               # Virtual environment (do not commit)
```

## Getting Started

### Prerequisites

- Python **3.10 or newer**
- `pip` (bundled with Python)

### Installation

1. **Clone or download** the project and open a terminal in its folder.

2. **Create a virtual environment**

   Windows:
   ```bash
   python -m venv venv
   venv\Scripts\activate
   ```

   macOS / Linux:
   ```bash
   python3 -m venv venv
   source venv/bin/activate
   ```

3. **Install dependencies**
   ```bash
   pip install -r requirements.txt
   ```

4. **Run the app**
   ```bash
   python app.py
   ```

5. **Open your browser** at <http://127.0.0.1:5000>

The database file `skill_exchange.db` is created automatically the first time you run the app.

## Usage Guide

1. **Register** with your name, email, and a password (minimum 6 characters).
2. **Complete your profile**: add a short bio, then enter skills as comma-separated lists:
   - *Skills I offer*: `Python, Guitar, Photoshop`
   - *Skills I want to learn*: `Web Development, French`
3. **Visit Matches** to see students whose skills complement yours, ranked by compatibility.
4. **Request a session** with a match for a skill they offer, choosing a date, time, and optional notes.
5. **Manage requests** on the Requests page: the teacher accepts or rejects, and either party can cancel or mark the session complete.
6. **Leave a review** once a session is marked completed.

### Quick demo

Create two accounts to see matching in action:

| Account | Offers          | Wants           |
|---------|-----------------|-----------------|
| A       | Python, C       | Web Development |
| B       | Web Development | Python          |

Log in as either user and open **Matches**. The other account appears as a compatible peer.

## How Matching Works

For every other user, the app compares skill names (case-insensitive) and computes a score:

```
score = |my wants ∩ their offers|  +  |my offers ∩ their wants|
```

- +1 for each skill they can teach you
- +1 for each skill you can teach them

Users with a score above 0 are shown, sorted from highest to lowest. Each match card also lists which skills they can teach you and which you can teach them.

## Session Lifecycle

```
            ┌──────────► rejected
            │
pending ────┼──────────► accepted ──────► completed ──► (review)
            │
            └──────────► cancelled
```

New requests start as `pending`. Valid actions are `accept`, `reject`, `complete`, and `cancel`. Reviews are only allowed on `completed` sessions, and only one review is stored per session.

## Database Schema

| Table         | Purpose                                              | Key columns |
|---------------|------------------------------------------------------|-------------|
| `users`       | Student accounts                                     | `id`, `name`, `email` (unique), `password` (hash), `bio` |
| `skills`      | Shared catalogue of skill names                      | `id`, `name` (unique) |
| `user_skills` | Links users to skills as `offer` or `learn`          | `user_id`, `skill_id`, `type` |
| `sessions`    | Peer-learning session requests                       | `requester_id`, `teacher_id`, `skill_id`, `scheduled_at`, `status`, `notes` |
| `reviews`     | Rating and comment for a completed session           | `session_id` (unique), `reviewer_id`, `reviewee_id`, `rating` (1-5), `comment` |

Foreign keys are enforced (`PRAGMA foreign_keys = ON`) with `ON DELETE CASCADE`.

## Routes

| Route                                  | Methods    | Auth | Description |
|----------------------------------------|------------|------|-------------|
| `/`                                    | GET        | No   | Landing page |
| `/register`                            | GET, POST  | No   | Create an account |
| `/login`                               | GET, POST  | No   | Sign in |
| `/logout`                              | GET        | No   | Sign out |
| `/profile`                             | GET, POST  | Yes  | View and edit profile and skills |
| `/dashboard`                           | GET        | Yes  | Skills overview and pending sessions |
| `/matches`                             | GET        | Yes  | Compatible peers |
| `/request/<teacher_id>/<skill_id>`     | GET, POST  | Yes  | Request a session |
| `/requests`                            | GET        | Yes  | All sessions you're involved in |
| `/session/<session_id>/<action>`       | POST       | Yes  | Accept, reject, complete, or cancel |
| `/review/<session_id>`                 | GET, POST  | Yes  | Review a completed session |

## Security Notes

Implemented:
- Passwords are hashed with `generate_password_hash`
- All database queries use parameterised statements (protection against SQL injection)
- Protected routes use a `login_required` decorator
- Only session participants can act on or review a session

Before deploying to production:
- **Change `app.secret_key`** to a long random value and load it from an environment variable
- **Turn off debug mode** (`app.run(debug=True)`) and serve with a production server such as Gunicorn or Waitress
- Add **CSRF protection** (for example, Flask-WTF)
- Restrict who may perform each action (for example, only the teacher should accept or reject)
- Make `logout` a POST request and add login rate limiting

## Configuration

| Setting       | Location | Default                | Notes |
|---------------|----------|------------------------|-------|
| `secret_key`  | `app.py` | `"change-this-secret-key"` | Must be changed in production |
| `DB`          | `app.py` | `"skill_exchange.db"`  | SQLite file path |
| Debug mode    | `app.py` | `True`                 | Disable in production |

## Troubleshooting

| Problem | Fix |
|---------|-----|
| `ModuleNotFoundError: No module named 'flask'` | Activate the virtual environment and run `pip install -r requirements.txt` |
| `Address already in use` | Another app is using port 5000. Stop it, or run with `app.run(port=5001)` |
| Template not found | Make sure the `templates/` folder sits next to `app.py` |
| Want a fresh start | Stop the app, delete `skill_exchange.db`, and run it again |
| PowerShell blocks venv activation | Run `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned` |

## Future Improvements

- Email notifications for session requests and status changes
- Search and filter students by skill
- Average rating displayed on profiles
- In-app messaging between matched students
- Calendar integration and reminders
- Profile photos and skill proficiency levels
- Admin dashboard and moderation tools

---

Built as a beginner-friendly full-stack project for learning Flask, SQLite, and web application design.
