Task Management Tool: Product Requirements Document
Version: 1.0
Author: Ibrahim
Status: Approved for development
Type: Public portfolio project (built from scratch)
1. Overview
The Task Management Tool is a web application where users organize, assign, and track their tasks. It has a REST API built with ASP.NET Core 8 and a React single-page frontend. It supports registration and login, role-based access (Admin and User), task CRUD with priorities, categories and due dates, search and filtering, and a dashboard showing task counts by status.
The project also serves as a portfolio piece, so engineering quality is a goal in its own right: clean structure, tests, logging, error handling, containerization, CI/CD, and documentation.
2. Goals and non-goals
Goals
• Deliver a working, deployed full-stack application with a public live demo.
• Demonstrate production-style practices: layered architecture, validation, global exception handling, structured logging, automated tests, Docker, and a CI pipeline.
• Keep the codebase small enough that the author can explain every decision in an interview.
Non-goals (version 1)
• Teams, workspaces, projects, or sub-tasks.
• File attachments, comments, or notifications by email.
• Refresh tokens and social login.
• Mobile app.
• Multi-language support.
3. Users and roles
Role
How they get it
What they can do
User
Self-registration
Create tasks for themselves; view, update, and delete only their own tasks; view categories; see their own dashboard and profile
Admin
Seeded at startup (credentials from configuration, never hardcoded)
Everything a User can do, plus: view and manage all tasks, assign tasks to any user, manage categories, view all users, see the global dashboard
There is no public endpoint that creates Admins.
4. Scope
MVP (must have)
1. Registration and login with JWT.
2. Role-based authorization (Admin, User).
3. Task CRUD with priority, status, due date, category, and assignment.
4. Categories (Admin-managed).
5. Search, filter, sort, and pagination for the task list.
6. Dashboard with task counts by status.
7. User profile and logout.
8. Application logging with Serilog.
9. Global exception handling with consistent error responses.
10. Unit and integration tests.
11. React frontend covering all screens in section 8.
12. Dockerfile, docker-compose, and a GitHub Actions CI pipeline.
Stretch (should have, in this order)
1. Deployment to Azure with CD.
2. Real-time task status updates with SignalR.
3. Export tasks to CSV, and import tasks from CSV.
5. Functional requirements
5.1 Authentication
• FR-A1: A visitor can register with full name, email, and password. New accounts get the User role.
• FR-A2: Passwords must be at least 8 characters with an uppercase letter, a lowercase letter, and a digit. Emails must be unique.
• FR-A3: A user can log in with email and password and receives a JWT access token (expiry 60 minutes) and basic user info.
• FR-A4: Failed logins return a generic message that doesn't reveal whether the email exists.
• FR-A5: Repeated failed logins lock the account temporarily (Identity lockout).
• FR-A6: Logout is handled client-side by discarding the token. The API is stateless.
• FR-A7: Unauthenticated requests to protected endpoints return 401. Authenticated requests without permission return 403.
5.2 Tasks
• FR-T1: An authenticated user can create a task with: title (required, max 200), description (optional, max 2000), priority (Low, Medium, High; default Medium), due date (optional), category (optional), and assignee.
• FR-T2: A User can only assign a task to themselves. An Admin can assign to any user.
• FR-T3: New tasks start as Pending. A task's due date can't be in the past when it is created.
• FR-T4: A task has three statuses: Pending, InProgress, Completed. It can move freely between them. Moving to Completed sets CompletedAt, and moving away from Completed clears it.
• FR-T5: A User can view, edit, change the status of, and delete only tasks assigned to them. An Admin can do this for any task. Accessing someone else's task as a User returns 404, so the existence of the task isn't revealed.
• FR-T6: Deleting a task is a soft delete. Deleted tasks never appear in any API response.
• FR-T7: Every task records who created it, who it is assigned to, and when it was created and last updated.
5.3 Task list: search, filter, sort, pagination
• FR-L1: The list returns only the tasks the caller is allowed to see (own tasks for User, all tasks for Admin).
• FR-L2: Filters: status, priority, category, assignee (Admin only), due date range, and overdue only.
• FR-L3: Free-text search on title and description.
• FR-L4: Sorting by due date, priority, created date, or title, ascending or descending.
• FR-L5: Server-side pagination with page and page size (default 10, max 50). The response includes total count and page info.
5.4 Categories
• FR-C1: Categories have a unique name and an optional description.
• FR-C2: All authenticated users can list categories. Only Admins can create, edit, or delete them.
• FR-C3: A category that still has tasks can't be deleted. The API returns a clear error.
5.5 Dashboard
• FR-D1: Show counts of Pending, InProgress, and Completed tasks, plus the number of overdue tasks.
• FR-D2: A User sees counts for their own tasks. An Admin sees counts across all tasks.
5.6 Users and profile
• FR-U1: A user can view their profile (name, email, role, join date).
• FR-U2: An Admin can list users (used for the assignee dropdown).
5.7 Logging and error handling
• FR-L6: Structured logs (Serilog) written to console and a rolling file. They include request logging, authentication events, task create/update/delete with the acting user ID, and all unhandled exceptions.
• FR-E1: A global exception middleware catches unhandled exceptions, logs them, and returns a standard ProblemDetails response without leaking stack traces.
• FR-E2: Validation errors return 400 with field-level messages in one consistent shape.
• FR-E3: Not found returns 404, conflicts (such as a duplicate email or category name) return 409.
5.8 Stretch features
• Real-time (SignalR): when a task's status changes, connected clients who can see that task get the update without refreshing.
• Export/Import: export the filtered task list to CSV. Import a CSV of tasks with row-level validation and a summary of accepted and rejected rows.
6. Data model
ApplicationUser (ASP.NET Core Identity): Id (string), Email, UserName, PasswordHash, FullName (100), CreatedAt.
Category: Id (int), Name (100, unique), Description (500, nullable).
TaskItem: Id (int), Title (200), Description (2000, nullable), Status (string enum), Priority (string enum), DueDate (nullable, UTC), CategoryId (nullable FK), CreatedByUserId (FK), AssignedToUserId (FK), CreatedAt, UpdatedAt (nullable), CompletedAt (nullable), IsDeleted (bit, default 0).
Relationships: a user creates many tasks and is assigned many tasks (two separate relationships). A category has many tasks (optional on the task side). Deleting a user or category that has tasks is restricted.
Indexes: unique on Categories.Name; composite on TaskItems (AssignedToUserId, Status); on TaskItems.DueDate; on TaskItems.CategoryId.
Conventions: all dates stored in UTC, enums stored as strings, and a global query filter hides soft-deleted tasks.
7. API design
Base path /api. All endpoints except register and login require a bearer token. Responses are JSON.
Method
Endpoint
Access
Purpose
POST
/auth/register
Public
Create a User account
POST
/auth/login
Public
Get a JWT
GET
/users/me
Authenticated
Current user's profile
GET
/users
Admin
List users
GET
/tasks
Authenticated
List tasks (filter, search, sort, paginate)
GET
/tasks/{id}
Owner or Admin
Task detail
POST
/tasks
Authenticated
Create a task
PUT
/tasks/{id}
Owner or Admin
Update a task
PATCH
/tasks/{id}/status
Owner or Admin
Change status only
DELETE
/tasks/{id}
Owner or Admin
Soft delete
GET
/categories
Authenticated
List categories
POST
/categories
Admin
Create category
PUT
/categories/{id}
Admin
Update category
DELETE
/categories/{id}
Admin
Delete category (if unused)
GET
/dashboard/summary
Authenticated
Status counts (scoped by role)
GET
/health
Public
Health check
Stretch: GET /tasks/export, POST /tasks/import, and a SignalR hub at /hubs/tasks.
The API is versioned in documentation through Swagger/OpenAPI, with JWT support in the Swagger UI.
8. Frontend screens
Screen
Contents
Behavior
Sign up
Name, email, password form
Validates input, registers, redirects to login
Log in
Email, password form
Authenticates and redirects to dashboard
Dashboard
Cards for Pending, InProgress, Completed, and Overdue counts
Counts scoped by role
Task list
Table or list with search, filters, sorting, pagination, and a New Task button
Loads from API, links to detail
Task detail
All task fields, status change control, edit and delete actions
Fetches by ID
New / Edit task
Form with title, description, priority, due date, category, and assignee (Admin only)
Creates or updates through the API
Categories (Admin)
List with add, edit, delete
Shows an error if a category is in use
User profile
Name, email, role, join date, logout button
Logout clears the session
UI requirements: responsive layout, loading and empty states, friendly error messages, protected routes, and role-aware navigation.
9. Non-functional requirements
Security
• Passwords hashed by ASP.NET Core Identity. No secrets in the repo (user secrets locally, environment variables in deployment).
• JWT signed with a strong key, validated for issuer, audience, lifetime, and signature.
• Authorization enforced on the server for every endpoint and every task access. The frontend hiding a button is never the only protection.
• CORS limited to the known frontend origin. HTTPS in deployment.
• Input validated on the server (FluentValidation). EF Core parameterization prevents SQL injection.
Performance
• List endpoints are paginated. Queries use the indexes defined above and avoid loading unneeded data (projection to DTOs, AsNoTracking for reads).
• Typical API response under 300 ms on the demo deployment.
Reliability and maintainability
• Consistent error format. Health check endpoint.
• Nullable reference types enabled and warnings kept at zero.
• Consistent code style enforced through .editorconfig.
Testing
• Unit tests (xUnit, Moq) for services and validators.
• Integration tests with WebApplicationFactory against a real SQL Server instance (Testcontainers or docker-compose) covering auth, authorization rules, task CRUD, and filtering.
• Target: the business rules in section 5 each have at least one test.
10. Architecture and tech stack
Backend: ASP.NET Core 8 Web API, EF Core 8, SQL Server, ASP.NET Core Identity, JWT bearer authentication, AutoMapper or manual mapping, FluentValidation, Serilog, Swagger.
Frontend: React with Vite, React Router, Axios, and a UI approach such as Tailwind CSS.
Tests: xUnit, Moq, FluentAssertions, Testcontainers.
Tooling: Git and GitHub, Docker, docker-compose, GitHub Actions.
Solution structure (layered, kept simple):
• TaskManager.Domain: entities and enums, no dependencies.
• TaskManager.Application: services, DTOs, interfaces, validators, business rules.
• TaskManager.Infrastructure: EF Core, DbContext, configurations, migrations, Identity, token generation.
• TaskManager.Api: controllers, middleware, configuration, startup.
• TaskManager.UnitTests and TaskManager.IntegrationTests.
• client/: the React app.
Dependencies point inward: Api and Infrastructure depend on Application, and Application depends on Domain.
11. DevOps and delivery
• Docker: a multi-stage Dockerfile for the API, a Dockerfile (or static hosting) for the client, and a docker-compose.yml that starts SQL Server, the API, and the client with one command.
• CI (GitHub Actions): on every push and pull request, restore, build, and run all tests. A status badge appears in the README.
• CD (stretch): on merge to main, build the image and deploy to Azure.
• Git strategy: main is always deployable. Work happens on short-lived feature branches (feature/auth, feature/tasks-crud) merged through pull requests, with small commits and Conventional Commit messages.
• Configuration: appsettings.json for defaults, user secrets locally, environment variables in Docker and Azure.
12. Acceptance criteria
The project counts as done when:
1. A new visitor can register, log in, create a task, change its status, and see the dashboard update.
2. A User cannot see or change another user's tasks (verified by tests).
3. An Admin can see all tasks, assign tasks, and manage categories.
4. Invalid input and server errors return the standard error format, and exceptions appear in the logs.
5. The full test suite passes in CI.
6. docker compose up starts a working application from a fresh clone.
7. The README explains the project, shows screenshots and an architecture diagram, and lists setup steps and design decisions.
8. A live demo is deployed and linked from the README.
13. Milestones
#
Milestone
Outcome
1
Requirements and database design
PRD and ERD approved
2
Repo, solution structure, entities, migrations
Database created from code
3
Authentication and authorization
Register, login, roles working
4
Task and category APIs
Full CRUD with business rules
5
Validation, error handling, logging
Consistent errors and structured logs
6
Search, filter, pagination, dashboard
List and summary endpoints complete
7
Unit and integration tests
Business rules covered
8
Docker and CI
One-command startup, green pipeline
9
React frontend
All screens working against the API
10
Deployment and README
Live demo and polished documentation
11
Stretch features
SignalR, CSV export/import
14. Risks and mitigations
Risk
Mitigation
Scope grows and the project never finishes
MVP is fixed. Stretch features only start after the MVP is deployed
Over-engineering a small app
Keep the layers simple and explain each choice in the README
Authorization bugs that leak other users' tasks
Central rule in the service layer plus integration tests for every task endpoint
Time lost on frontend styling
Use a component approach and a minimal design, and focus on function
Secrets committed by accident
.gitignore from day one, user secrets, and a pre-push check of the repo
15. Future enhancements
Refresh tokens, email notifications, comments and attachments, task activity history, team workspaces, full-text search, and rate limiting.