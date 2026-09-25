---
name: vikunja
description: Create and list tasks in the self-hosted Vikunja task manager on this server.
required_environment_variables:
  - name: VIKUNJA_TOKEN
    prompt: Vikunja API token (projects read, tasks read/create)
    help: Vikunja → Settings → API Tokens
    required_for: creating tasks
---

# Vikunja (self-hosted tasks)

Vikunja runs on this server at `http://127.0.0.1:3456/api/v1`. Authenticate every request with
`-H "Authorization: Bearer $VIKUNJA_TOKEN"`.

- Projects: `GET /projects` → `[].id`, `[].title`
- Find tasks: `GET /tasks?s=<search text>` (searches titles across all projects)
- Create a task: `PUT /projects/<id>/tasks` with JSON
  `{"title": "...", "description": "...", "due_date": "2026-10-09T12:00:00+02:00"}`

Before creating a task, search for its title and do not create a duplicate. Use German task titles. After creating, report the task id, title and due date.
