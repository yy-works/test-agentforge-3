AgentForge API Reference
========================
Updated  : 2026-3-26 08:25:21
Base URL : http://10.8.8.2:8900
Auth     : none (internal service)
Format   : JSON (Content-Type: application/json)

This service receives tasks from agents (e.g. OpenClaw), runs an AI coding
pipeline (build → review → test) inside an isolated git worktree. All git
operations (commit, push, merge) are handled by OpenCode inside the workspace.
See /install for this document.


QUICK START
-----------
1. Register a project (once):
   POST /project
   {"name":"my-repo","git_url":"git@github.com:org/repo.git","branch":"main"}

2. Dispatch a task:
   POST /task
   {"project":"my-repo","pipeline":"default","prompt":"<what to build>","session_key":"<caller-session-key>"}
   => {"task_id":"task_abc123","status":"pending","branch":"af/task_abc123"}

3. Poll for status:
   GET /task/task_abc123

4. If status == "waiting_for_input", reply to the intervention:
   POST /task/task_abc123/reply
   {"reply":"fix_and_continue"}

5. When status == "done", the pipeline is complete.
   Use POST /task/:id/intervene with action=guide to send messages to
   the task's OpenCode session (e.g. ask it to push, create PR, etc).


TASK STATUS VALUES
------------------
pending            Created, queued for execution
running            Pipeline stages executing
waiting_for_input  Paused — needs a reply (see intervention field in GET /task/:id)
done               Pipeline completed successfully
error              Pipeline failed
cancelled          Cancelled by request


TASK APIs
---------
POST /task
  Create and dispatch a coding task.
  Body:
    project        (string, required)  Project name registered via POST /project
    pipeline       (string, required)  Pipeline name, e.g. "default"
    prompt         (string, required)  Natural-language task description
    session_key    (string, required)  Caller's session key (e.g. OpenClaw current session key);
                                       used to send status-change and approval callbacks
    base_branch    (string, optional)  Branch to base worktree on; defaults to project.branch
    fetch_latest   (bool,   optional)  Run git fetch before creating worktree (default: false)
    model          (string, optional)  Override LLM model, e.g. "claude-sonnet-4-6"
    profile_id     (string, optional)  Tool Profile ID for permission overrides
    docker         (bool,   optional)  Run in Docker container; defaults to project.docker then
                                       global config.docker.enabled (default: true)
  Response: {task_id, status, branch, base_branch?}

GET /task/:id
  Get task status, stage results, and any pending intervention.
  Response: TaskRecord
    task_id, project, pipeline, prompt, status, branch, current_stage,
    stages[]{agent, status, summary, diff_count},
    intervention?{id, question, options[], option_labels?, stage,
                  review_issues?, test_results?, permission_id?, created_at},
    opencode_url?   URL of the OpenCode server for this task (docker mode only, present when port allocated)
    preview_urls    Array of preview service entries (always present, empty array if no ports allocated):
                      [{url: string, status: "up"|"down"}]
                    status reflects live health check (HTTP probe with 2s timeout)
    error?, created_at, updated_at

GET /tasks[?project=&status=&archived=true]
  List tasks, optionally filtered by project name or status.
  Add archived=true to list only archived tasks.
  Response: TaskRecord[]

POST /task/:id/reply
  Reply to a waiting_for_input intervention.
  Body:
    reply            (string, required)  For review/test: one of the options[] values
                                         (fix_and_continue | ignore_and_continue | cancel)
                                         For permission interventions: free-form approval text
    intervention_id  (string, optional)  Intervention ID for idempotency
    message          (string, optional)  Extra instruction appended to the agent session
  Response: {task_id, status}

POST /task/:id/intervene
  Actively intervene in a running or completed task.
  Body:
    action                (string, required)  "guide" | "rerun_from" | "cancel"
    message               (string, optional)  Instruction for guide/rerun_from.
                                               Use guide to send messages to the OpenCode session
                                               (e.g. "push the branch", "create a PR").
    stage                 (string, optional)  Stage name to restart from (rerun_from only)
    inject_previous_result (bool, optional)  Inject prior stage result as context (rerun_from only)
  Response: {task_id, status, message?}

DELETE /task/:id
  Delete a terminal task and clean up its git worktree.
  Only allowed when status is: done | error | cancelled
  Response: {task_id, deleted:true}

POST /task/:id/archive
  Archive a terminal task (hidden from normal listings).
  Only allowed when status is: done | error | cancelled
  Response: {task_id, archived:true}

POST /task/:id/restore
  Restore an archived task back to normal visibility.
  Response: {task_id, archived:false}

POST /task/:id/cancel
  Cancel a running or waiting task.
  Response: {task_id, status}

POST /task/:id/restart
  Re-run a failed or cancelled task in-place (reuse same task ID and branch).
  Body (all optional overrides): prompt, pipeline, base_branch, fetch_latest, model, profile_id, docker
  Response: {task_id, status, branch}


PROJECT APIs
------------
POST /repo
  Create a new GitHub repository with initial files and commit AgentForge.md.
  Requires GITHUB_TOKEN environment variable.
  Body:
    repo_name  (string,   required)  Repository name to create
    org        (string,   optional)  GitHub org or user (default: "yyworks")
    files      (array,    optional)  Additional files to commit:
                                     [{path: string, content: string}, ...]
                                     README.md is auto-created (empty) if not included.
                                     AgentForge.md is always committed automatically.
  Response: {url}  — full clone URL ending with .git, e.g.
                     https://github.com/yyworks/my-repo.git

POST /project
  Register a git repository. Clones it to local repos_dir. Only needed once.
  Body:
    name     (string, required)  Unique project identifier used in all task calls
    git_url  (string, required)  SSH or HTTPS clone URL
    branch   (string, optional)  Default branch (default: "main")
    docker   (bool,   optional)  Default docker mode for tasks in this project;
                                 overrides global config.docker.enabled
  Response: {name, status, local_path}

GET /projects
  List all registered projects.
  Response: ProjectRecord[]

GET /project/:name
  Get a single project's info.
  Response: ProjectRecord {name, git_url, local_path, branch, docker?}

PATCH /project/:name
  Update project settings.
  Body (all optional):
    branch   (string)  Change default branch
    docker   (bool)    Change default docker mode for tasks in this project
  Response: ProjectRecord

POST /project/:name/ask
  Ask a question about the project codebase (read-only, does not affect any task).
  Body:   {question: string}
  Response: {answer, session_id}


UTILITY APIs
------------
GET /pipelines
  List pipeline definitions from config.yaml.
  Response: [{name, stages[{agent, require_approval, description, inject_diff}]}]

GET /models
  List available LLM model identifiers.
  Response: string[]

GET /logs[?task_id=&limit=]
  Fetch structured log entries (default limit: 200).
  Response: [{timestamp, level, task_id?, event, data}]


WEBHOOK CALLBACKS
-----------------
When a task reaches a notable state, AgentForge POSTs to the URL configured
in config.yaml at openclaw.url (retried twice: at +5s and +15s).

Payload:
  {
    "task_id": "task_abc123",
    "status":  "waiting_for_input",   // new status
    "branch":  "af/task_abc123",
    "intervention": { ... }           // present when status == waiting_for_input
  }


INTERVENTION FLOW
-----------------
Passive (auto-triggered):
  Review or test stage blocks, or OpenCode requests a tool permission.
  Task enters waiting_for_input and webhook fires.
  Reply with POST /task/:id/reply using one of intervention.options[]:
    fix_and_continue     Re-run build then retry the blocked stage
    ignore_and_continue  Skip the issue and continue pipeline
    cancel               Abort and discard

Active (any time, including after pipeline completion):
  POST /task/:id/intervene with action=guide to send a message to the
  task's OpenCode session. Use this to instruct OpenCode to push, create
  PRs, or perform any git operations.
  Use action=rerun_from to restart from a specific stage.
  Use action=cancel to abort immediately.
