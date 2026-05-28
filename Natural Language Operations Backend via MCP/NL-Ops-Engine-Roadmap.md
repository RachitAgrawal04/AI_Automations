# NL-Ops Engine — Complete Project Roadmap
### A Natural Language Operations Backend via MCP
**Built by:** Rachit Agrawal | **Stack:** Python + FastMCP + SQLite + Claude Desktop

---

## Before You Read This

This document is your single source of truth for the entire project. Read it once fully before writing a single line of code. The concepts section at the top will save you days of confusion. Every phase builds on the previous one — do not skip phases.

---

## Part 1: What You're Actually Building (And Why It Matters)

### The Problem This Solves

Right now, if you want to manage your CafeCanvas client data — tasks, contacts, project status, follow-ups — you either open a spreadsheet, open a CRM like Notion, or try to remember it. These tools are rigid: they accept data in their format, on their terms.

What you're building is a backend that **accepts plain English as its interface**. You type:

> *"Which clients haven't been followed up with in the last 10 days?"*

And it queries your database and tells you. No UI to navigate. No SQL to write. Just English.

Under the hood, Claude (the AI) interprets your sentence, decides which tools to call, calls your MCP server, your server runs the actual database query, and Claude narrates the result back. You are building the server — the engine that does the real work.

### Why This Is Portfolio-Worthy

A task manager MCP is a tutorial. What you're building is different:

- **Multi-entity relational data** (tasks + contacts + projects + interactions) with JOINs
- **Tool design that guides LLM behavior** — writing tool descriptions that make Claude call the right tool at the right time
- **Observability from day one** — logging every tool call with input/output/timestamp
- **Production patterns** — input validation, error recovery, connection management

This is the same architecture used in Notion's AI, Salesforce Einstein Copilot, and Linear's AI features. You're building a micro version of enterprise AI tooling.

---

## Part 2: Core Concepts You Must Understand First

### What Is MCP?

MCP stands for **Model Context Protocol**. It is a standard that Anthropic created so that AI clients (like Claude Desktop) can talk to external servers in a predictable way.

Think of it like this: HTTP is a protocol that lets browsers talk to web servers. MCP is a protocol that lets AI assistants talk to "tool servers."

```
Without MCP:               With MCP:
Claude knows only           Claude can call YOUR code
what it was trained on      and use real, live data
```

### The Three Moving Parts

**1. The MCP Client** — This is Claude Desktop. It's the AI that talks to you, decides when to call tools, and narrates results. You do NOT build this. It already exists.

**2. The MCP Server** — This is YOUR Python code. It defines tools (functions), exposes them over the MCP protocol, and executes them when Claude asks. This is what you're building.

**3. The Tools** — These are Python functions decorated with `@mcp.tool()`. Each tool has a name, a description (critical!), input parameters with types, and a return value. Claude reads the description and decides whether to call the tool.

### The Exact Flow (Memorize This)

```
You type:  "Add a task to send proposal to Sharma Clinic by Friday"
    ↓
Claude Desktop receives your message
    ↓
Claude thinks: "The user wants to add a task. I have an add_task tool.
               Its description says: 'Create a new task with title, optional
               due date, priority, and project.' That matches. I'll call it."
    ↓
Claude calls your MCP server: add_task(title="Send proposal to Sharma Clinic", due_date="2026-05-30")
    ↓
Your Python function runs → inserts into SQLite → returns {"status": "success", "task_id": 42}
    ↓
Claude receives the result
    ↓
Claude replies to you: "Done! I've added 'Send proposal to Sharma Clinic' 
                        due this Friday."
```

**The key insight:** Claude decides WHEN and WHETHER to call your tool. Your code decides HOW to execute it. They never share memory — they communicate only through the MCP protocol.

### Why Tool Descriptions Are Everything

This is the most important concept in the entire project.

Claude cannot see your Python code. It cannot see your database. The ONLY thing Claude sees is:
- The tool's **name**
- The tool's **description**
- The tool's **parameter names and types**

When you type something, Claude pattern-matches your sentence against all available tool descriptions to decide what to do. If your description is vague, Claude will either:
- Call the wrong tool
- Skip your tool entirely and make something up
- Call the tool with wrong parameters

**Bad description:**
```python
@mcp.tool()
def add_task(title: str, due_date: str) -> str:
    """Adds task"""  # ← Claude has no idea when to use this
```

**Good description:**
```python
@mcp.tool()
def add_task(title: str, due_date: str = None, priority: str = "medium") -> str:
    """
    Create a new task in the task management system.
    Use this when the user wants to add, create, or schedule a task or to-do item.
    
    Args:
        title: The task description (e.g. 'Send proposal to Sharma Clinic')
        due_date: Optional deadline in YYYY-MM-DD format. If the user says 'tomorrow'
                  or 'Friday', convert to the actual date before calling this tool.
        priority: Task priority - 'low', 'medium', or 'high'. Default is 'medium'.
    
    Returns a confirmation with the new task's ID.
    """
```

Notice what the good description does:
- Tells Claude **when** to use it ("when the user wants to add, create, or schedule")
- Tells Claude **how** to handle edge cases ("if user says 'tomorrow', convert first")
- Is specific about parameter formats ("YYYY-MM-DD")

You will spend more time writing tool descriptions than writing SQL. That is correct and intentional.

### What FastMCP Is

The official MCP Python SDK (by Anthropic) requires you to write a lot of boilerplate code to define tools, handle the protocol, manage the server lifecycle, etc.

FastMCP is a third-party library that wraps the official SDK and lets you define tools with just a Python decorator. The underlying protocol is 100% identical — Claude Desktop cannot tell the difference. FastMCP just removes friction.

```python
# Without FastMCP (official SDK) — ~40 lines of boilerplate
# With FastMCP — just this:

from fastmcp import FastMCP

mcp = FastMCP("nl-ops-engine")

@mcp.tool()
def ping() -> str:
    """Test if the server is running."""
    return "pong"

if __name__ == "__main__":
    mcp.run()
```

### What stdio Transport Means

When Claude Desktop launches your MCP server, it does so by running your Python script as a subprocess. Claude and your server communicate via standard input/output (stdin/stdout) — the same pipes that let you do `echo "hello" | python script.py` in a terminal.

This is the "stdio transport." You don't need to run a web server. You don't need a port. Claude Desktop handles the process management. Your script just needs to call `mcp.run()` and FastMCP handles the rest.

This is why the `claude_desktop_config.json` setup looks the way it does — it's telling Claude Desktop "to start this MCP server, run THIS Python executable with THIS script."

---

## Part 3: Project Architecture

### Final Database Schema

```
┌─────────────┐       ┌──────────────┐
│   projects  │       │   contacts   │
├─────────────┤       ├──────────────┤
│ id (PK)     │       │ id (PK)      │
│ name        │       │ name         │
│ client_name │       │ phone        │
│ status      │       │ email        │
│ deadline    │       │ source       │  ← 'lead' or 'client'
│ budget      │       │ project_id (FK)→ projects
│ notes       │       │ notes        │
│ created_at  │       │ created_at   │
└──────┬──────┘       └──────┬───────┘
       │                     │
       │          ┌──────────┘
       ▼          ▼
┌─────────────────────────┐     ┌──────────────────────┐
│         tasks           │     │     interactions     │
├─────────────────────────┤     ├──────────────────────┤
│ id (PK)                 │     │ id (PK)              │
│ title                   │     │ contact_id (FK)      │
│ priority                │     │ type   ← call/email/ │
│ status  ← todo/done/    │     │          meeting     │
│          in_progress    │     │ summary              │
│ due_date                │     │ follow_up_date       │
│ project_id (FK) ────────┘     │ created_at           │
│ created_at                    └──────────────────────┘
└─────────────────────────┘
```

### Final Folder Structure

```
nl-ops-engine/
│
├── server.py              ← Main FastMCP server (entry point)
├── database.py            ← All SQLite connection and setup logic
├── tools/
│   ├── __init__.py
│   ├── tasks.py           ← add_task, list_tasks, update_task, delete_task
│   ├── contacts.py        ← add_contact, list_contacts, get_contact
│   ├── projects.py        ← add_project, list_projects, update_project
│   └── interactions.py    ← log_interaction, get_contact_history
├── utils/
│   ├── __init__.py
│   ├── logger.py          ← Structured tool call logging
│   └── validators.py      ← Input sanitization and validation
├── data/
│   └── nl_ops.db          ← SQLite database (auto-created)
├── logs/
│   └── tool_calls.jsonl   ← One JSON line per tool invocation
├── requirements.txt
└── README.md
```

You will build this folder structure incrementally. Do not create it all at once on Day 1.

---

## Part 4: Environment Setup (Do This First)

### Step 1: Check Your Python Version

Open a terminal (Command Prompt or PowerShell on Windows) and run:

```bash
python --version
```

You need Python 3.10 or higher. If you have an older version, download Python 3.12 from python.org.

### Step 2: Find Your Exact Python Path

This is critical for the Claude Desktop config. Run:

```bash
where python      # Windows
which python      # Mac/Linux
```

Copy the full path. It will look something like:
`C:\Users\Rachit\AppData\Local\Programs\Python\Python312\python.exe`

### Step 3: Create Your Project

```bash
mkdir nl-ops-engine
cd nl-ops-engine
python -m venv venv
```

This creates a virtual environment — an isolated Python installation just for this project. Good practice always.

Activate it:
```bash
# Windows (PowerShell):
.\venv\Scripts\Activate.ps1

# Windows (Command Prompt):
.\venv\Scripts\activate.bat

# Mac/Linux:
source venv/bin/activate
```

You should see `(venv)` at the start of your terminal prompt.

### Step 4: Install Dependencies

```bash
pip install fastmcp
pip install python-dateparser
```

Create `requirements.txt`:
```
fastmcp==0.4.2
python-dateparser==1.2.0
```

Pin your versions. Check what installed with `pip show fastmcp` and use that exact version number. This prevents "it worked yesterday" bugs.

### Step 5: Install Claude Desktop

Download from: https://claude.ai/download

After installing, find the config file:
- **Windows:** `%APPDATA%\Claude\claude_desktop_config.json`
- **Mac:** `~/Library/Application Support/Claude/claude_desktop_config.json`

If the file doesn't exist, create it. This is where you register your MCP server.

---

## Part 5: Phase 0 — Understand Before You Build (Day 1)

**Goal:** Build the mental model before touching code. This phase has no coding.

### What To Do

Read these in order — each takes about 20 minutes:

1. **FastMCP README** — https://github.com/jlowin/fastmcp
   Focus: Quickstart section, Tool definition section, running the server

2. **MCP Architecture Overview** — https://modelcontextprotocol.io/docs/concepts/architecture
   Focus: What is a server, what is a client, what is a tool

3. **Claude Desktop MCP Setup** — https://modelcontextprotocol.io/docs/claude-desktop/usage
   Focus: The claude_desktop_config.json format

### Questions To Be Able to Answer After Phase 0

- What is the difference between the MCP client and the MCP server?
- Who calls the tools — you, or Claude?
- Why does the tool description matter more than the code?
- What is stdio transport and why does it not need a web server?
- What does `mcp.run()` actually do?

If you can answer these, move to Phase 1.

---

## Part 6: Phase 1 — Minimal Working Server (Days 2–3)

**Goal:** Get Claude Desktop to successfully call a tool you wrote. Nothing else.

**Why this phase exists:** Every tutorial says "just start building." That's wrong. Your Phase 1 must validate your entire setup — Python path, FastMCP install, Claude Desktop config, stdio communication — before you write any real logic. If something is broken in Phase 1, you find out with a `ping` tool, not after you've written 200 lines.

### What To Build

Create `server.py`:

```python
from fastmcp import FastMCP
import logging

# Set up basic logging so you can see what's happening
logging.basicConfig(level=logging.DEBUG)

# Create your MCP server instance
# The name "nl-ops-engine" appears in Claude Desktop's UI
mcp = FastMCP("nl-ops-engine")

@mcp.tool()
def ping() -> str:
    """
    Test if the NL-Ops Engine server is running and reachable.
    Use this to verify the MCP connection is working.
    Returns 'pong' if everything is working correctly.
    """
    return "pong"

@mcp.tool()
def hello(name: str = "World") -> str:
    """
    Return a greeting. Use this to test parameter passing.
    
    Args:
        name: The name to greet. Defaults to 'World'.
    """
    return f"Hello, {name}! The NL-Ops Engine is running."

if __name__ == "__main__":
    mcp.run()
```

### Configure Claude Desktop

Open `claude_desktop_config.json` and add:

```json
{
  "mcpServers": {
    "nl-ops-engine": {
      "command": "C:\\Users\\Rachit\\AppData\\Local\\Programs\\Python\\Python312\\python.exe",
      "args": ["C:\\path\\to\\your\\nl-ops-engine\\server.py"]
    }
  }
}
```

**Replace both paths with your actual paths.** On Windows, use double backslashes `\\` or forward slashes `/`. The `command` must be the full path from `where python` — not just `python`.

Restart Claude Desktop completely after saving this file.

### How To Test

Open Claude Desktop and type:

- "Ping the server"
- "Run the ping tool"
- "Hello, can you greet me as Rachit?"

If Claude responds with "pong" or a greeting, Phase 1 is complete. You have a working MCP server.

### Common Problems And Fixes

**Problem:** Claude Desktop doesn't show the MCP server in its tool list
**Fix:** The JSON is malformed. Validate it at jsonlint.com. Even one missing comma breaks it.

**Problem:** Server shows as "failed to connect"
**Fix:** The Python path is wrong. Open terminal, activate venv, run `where python`, copy that exact path.

**Problem:** Claude says "I don't have any tools to ping"
**Fix:** You haven't restarted Claude Desktop after saving the config. Fully quit and reopen.

**Problem:** You see an error like "ModuleNotFoundError: No module named 'fastmcp'"
**Fix:** Your `command` path points to the system Python, not your venv Python. The venv Python is at `.\venv\Scripts\python.exe` inside your project folder.

**Do not proceed to Phase 2 until ping works.**

---

## Part 7: Phase 2 — Single Entity: Tasks (Days 4–7)

**Goal:** Build a complete task management system with 4 tools. Introduce SQLite. Add logging from day one.

### Step 1: Create database.py

This file handles all database setup. Keep it separate from your tools.

```python
import sqlite3
import os
from pathlib import Path

# The database file lives in a 'data' folder next to this script
DB_PATH = Path(__file__).parent / "data" / "nl_ops.db"

def get_connection():
    """
    Return a SQLite connection.
    check_same_thread=False allows use across threads (needed when we add FastAPI later).
    """
    os.makedirs(DB_PATH.parent, exist_ok=True)
    conn = sqlite3.connect(str(DB_PATH), check_same_thread=False)
    conn.row_factory = sqlite3.Row  # This makes rows accessible like dicts: row["title"]
    return conn

def initialize_database():
    """Create all tables if they don't exist. Safe to call on every startup."""
    conn = get_connection()
    cursor = conn.cursor()
    
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS tasks (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            title TEXT NOT NULL,
            priority TEXT DEFAULT 'medium' CHECK(priority IN ('low', 'medium', 'high')),
            status TEXT DEFAULT 'todo' CHECK(status IN ('todo', 'in_progress', 'done')),
            due_date TEXT,  -- stored as YYYY-MM-DD string
            project_id INTEGER,  -- FK to projects table (added in Phase 3)
            created_at TEXT DEFAULT (datetime('now', 'localtime'))
        )
    """)
    
    conn.commit()
    conn.close()
```

### Step 2: Create utils/logger.py

Add this before writing any tools. This is your telemetry foundation.

```python
import json
import logging
from datetime import datetime
from pathlib import Path

LOG_PATH = Path(__file__).parent.parent / "logs" / "tool_calls.jsonl"

def log_tool_call(tool_name: str, inputs: dict, output: str, error: str = None):
    """
    Log every tool call to a JSONL file (one JSON object per line).
    JSONL is easy to parse, grep, and analyze later.
    
    This is your observability layer — you should be able to look at
    tool_calls.jsonl and reconstruct exactly what happened in a session.
    """
    Path(LOG_PATH).parent.mkdir(exist_ok=True)
    
    entry = {
        "timestamp": datetime.now().isoformat(),
        "tool": tool_name,
        "inputs": inputs,
        "output": output,
        "error": error,
        "status": "error" if error else "success"
    }
    
    with open(LOG_PATH, "a") as f:
        f.write(json.dumps(entry) + "\n")
    
    # Also log to console for development
    if error:
        logging.error(f"[{tool_name}] ERROR: {error}")
    else:
        logging.info(f"[{tool_name}] OK | inputs={inputs}")
```

### Step 3: Create tools/tasks.py

Build tools one at a time in this order. Test each one in Claude Desktop before writing the next.

```python
import sqlite3
from database import get_connection
from utils.logger import log_tool_call

def add_task(title: str, due_date: str = None, priority: str = "medium") -> str:
    """
    Create a new task in the task management system.
    Use when the user wants to add, create, or schedule a task or to-do item.
    
    Args:
        title: Clear description of what needs to be done.
        due_date: Deadline in YYYY-MM-DD format. If user says 'Friday' or 'tomorrow',
                  you must convert to actual date first, then call this tool.
        priority: One of 'low', 'medium', or 'high'. If user says 'urgent' use 'high'.
                  Default is 'medium'.
    
    Returns: Confirmation message with the new task ID.
    """
    inputs = {"title": title, "due_date": due_date, "priority": priority}
    
    # --- Input Validation ---
    # Never trust inputs blindly, even from an AI
    if not title or not title.strip():
        error = "Task title cannot be empty."
        log_tool_call("add_task", inputs, None, error)
        return f"Error: {error}"
    
    valid_priorities = ["low", "medium", "high"]
    if priority not in valid_priorities:
        priority = "medium"  # Graceful fallback, don't crash
    
    try:
        conn = get_connection()
        cursor = conn.cursor()
        
        cursor.execute(
            "INSERT INTO tasks (title, due_date, priority) VALUES (?, ?, ?)",
            (title.strip(), due_date, priority)
        )
        task_id = cursor.lastrowid
        conn.commit()
        conn.close()
        
        result = f"Task created successfully. ID: {task_id}, Title: '{title}', Priority: {priority}"
        if due_date:
            result += f", Due: {due_date}"
        
        log_tool_call("add_task", inputs, result)
        return result
        
    except Exception as e:
        error = f"Database error: {str(e)}"
        log_tool_call("add_task", inputs, None, error)
        return f"Error creating task: {error}"


def list_tasks(status: str = None, priority: str = None) -> str:
    """
    Retrieve tasks from the database, with optional filtering.
    Use when the user asks to see, show, list, or display tasks.
    
    Args:
        status: Filter by status. One of: 'todo', 'in_progress', 'done'.
                If user says 'pending' or 'open', use 'todo'.
                If user says 'all', pass None to get everything.
        priority: Filter by priority. One of: 'low', 'medium', 'high'.
                  Pass None to get all priorities.
    
    Returns: Formatted list of tasks or a message if none found.
    """
    inputs = {"status": status, "priority": priority}
    
    try:
        conn = get_connection()
        cursor = conn.cursor()
        
        # Build query dynamically based on which filters are provided
        query = "SELECT id, title, priority, status, due_date, created_at FROM tasks WHERE 1=1"
        params = []
        
        if status:
            query += " AND status = ?"
            params.append(status)
        if priority:
            query += " AND priority = ?"
            params.append(priority)
        
        query += " ORDER BY due_date ASC, priority DESC"
        
        cursor.execute(query, params)
        tasks = cursor.fetchall()
        conn.close()
        
        if not tasks:
            result = "No tasks found matching the criteria."
            log_tool_call("list_tasks", inputs, result)
            return result
        
        # Format the output clearly for Claude to read and narrate
        lines = [f"Found {len(tasks)} task(s):\n"]
        for task in tasks:
            due = f" | Due: {task['due_date']}" if task['due_date'] else ""
            lines.append(
                f"ID {task['id']}: [{task['status'].upper()}] {task['title']} "
                f"(Priority: {task['priority']}){due}"
            )
        
        result = "\n".join(lines)
        log_tool_call("list_tasks", inputs, f"Returned {len(tasks)} tasks")
        return result
        
    except Exception as e:
        error = f"Database error: {str(e)}"
        log_tool_call("list_tasks", inputs, None, error)
        return f"Error listing tasks: {error}"


def update_task_status(task_id: int, new_status: str) -> str:
    """
    Update the status of an existing task.
    Use when user says a task is done, completed, finished, or wants to mark it
    as in-progress or reopen it.
    
    Args:
        task_id: The numeric ID of the task to update. If you don't know the ID,
                 use list_tasks first to find it.
        new_status: The new status. Must be one of: 'todo', 'in_progress', 'done'.
                    If user says 'done' or 'completed' or 'finished', use 'done'.
                    If user says 'started' or 'working on it', use 'in_progress'.
    
    Returns: Confirmation or an error if the task ID doesn't exist.
    """
    inputs = {"task_id": task_id, "new_status": new_status}
    
    valid_statuses = ["todo", "in_progress", "done"]
    if new_status not in valid_statuses:
        error = f"Invalid status '{new_status}'. Must be one of: {valid_statuses}"
        log_tool_call("update_task_status", inputs, None, error)
        return f"Error: {error}"
    
    try:
        conn = get_connection()
        cursor = conn.cursor()
        
        # First check the task exists
        cursor.execute("SELECT title FROM tasks WHERE id = ?", (task_id,))
        task = cursor.fetchone()
        
        if not task:
            error = f"No task found with ID {task_id}. Use list_tasks to find valid IDs."
            conn.close()
            log_tool_call("update_task_status", inputs, None, error)
            return f"Error: {error}"
        
        cursor.execute("UPDATE tasks SET status = ? WHERE id = ?", (new_status, task_id))
        conn.commit()
        conn.close()
        
        result = f"Task {task_id} ('{task['title']}') status updated to '{new_status}'."
        log_tool_call("update_task_status", inputs, result)
        return result
        
    except Exception as e:
        error = f"Database error: {str(e)}"
        log_tool_call("update_task_status", inputs, None, error)
        return f"Error updating task: {error}"


def delete_old_tasks(days_old: int = 7) -> str:
    """
    Delete tasks that are marked as 'done' and older than a specified number of days.
    Use only when user explicitly asks to delete or clean up old completed tasks.
    Never delete tasks without explicit user instruction.
    
    Args:
        days_old: Delete done tasks older than this many days. Default is 7.
                  Minimum value: 1 (never 0, to prevent accidental mass deletion).
    
    Returns: How many tasks were deleted.
    """
    inputs = {"days_old": days_old}
    
    if days_old < 1:
        days_old = 1  # Safety guard
    
    try:
        conn = get_connection()
        cursor = conn.cursor()
        
        cursor.execute("""
            DELETE FROM tasks 
            WHERE status = 'done' 
            AND created_at < datetime('now', ? || ' days')
        """, (f"-{days_old}",))
        
        deleted_count = cursor.rowcount
        conn.commit()
        conn.close()
        
        result = f"Deleted {deleted_count} completed task(s) older than {days_old} days."
        log_tool_call("delete_old_tasks", inputs, result)
        return result
        
    except Exception as e:
        error = f"Database error: {str(e)}"
        log_tool_call("delete_old_tasks", inputs, None, error)
        return f"Error deleting tasks: {error}"
```

### Step 4: Update server.py to Register These Tools

```python
from fastmcp import FastMCP
from database import initialize_database
from tools.tasks import add_task, list_tasks, update_task_status, delete_old_tasks
import logging

logging.basicConfig(level=logging.INFO)

mcp = FastMCP("nl-ops-engine")

# Initialize DB on startup (creates tables if they don't exist)
initialize_database()

# Register task tools
mcp.tool()(add_task)
mcp.tool()(list_tasks)
mcp.tool()(update_task_status)
mcp.tool()(delete_old_tasks)

if __name__ == "__main__":
    mcp.run()
```

### Phase 2 Test Checklist

Test each of these in Claude Desktop before moving on:

- [ ] "Add a task to finalize the BMS IT Solutions contract by Friday, high priority"
- [ ] "Show me all my tasks"
- [ ] "Show me only high priority tasks"
- [ ] "Mark task 1 as in progress"
- [ ] "Which tasks are still pending?"
- [ ] "Delete tasks older than 3 days"
- [ ] "Add a task with no title" ← should get a graceful error, not a crash

Check `logs/tool_calls.jsonl` — every call should be logged. This file is your audit trail.

---

## Part 8: Phase 3 — Multi-Entity System (Days 8–11)

**Goal:** Add contacts, projects, and interactions. Your database becomes relational. You learn JOINs.

**Why this is where the project becomes impressive:** Phase 2 is a feature. Phase 3 is an architecture. When a recruiter asks "what was technically interesting about this?" the answer is here: you designed a multi-entity relational backend that an LLM can query in natural language across joined tables.

### Step 1: Expand database.py

Add these tables to your `initialize_database()` function:

```python
cursor.execute("""
    CREATE TABLE IF NOT EXISTS projects (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        name TEXT NOT NULL,
        client_name TEXT,
        status TEXT DEFAULT 'active' CHECK(status IN ('active', 'paused', 'completed', 'cancelled')),
        deadline TEXT,
        budget REAL,
        notes TEXT,
        created_at TEXT DEFAULT (datetime('now', 'localtime'))
    )
""")

cursor.execute("""
    CREATE TABLE IF NOT EXISTS contacts (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        name TEXT NOT NULL,
        phone TEXT,
        email TEXT,
        source TEXT DEFAULT 'lead' CHECK(source IN ('lead', 'client', 'partner', 'vendor')),
        project_id INTEGER REFERENCES projects(id),
        notes TEXT,
        created_at TEXT DEFAULT (datetime('now', 'localtime'))
    )
""")

cursor.execute("""
    CREATE TABLE IF NOT EXISTS interactions (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        contact_id INTEGER NOT NULL REFERENCES contacts(id),
        type TEXT CHECK(type IN ('call', 'email', 'meeting', 'message', 'other')),
        summary TEXT NOT NULL,
        follow_up_date TEXT,
        created_at TEXT DEFAULT (datetime('now', 'localtime'))
    )
""")

# Also update tasks to support project_id
# SQLite doesn't support ADD COLUMN with constraints easily, 
# so add a migration check:
try:
    cursor.execute("ALTER TABLE tasks ADD COLUMN project_id INTEGER REFERENCES projects(id)")
except Exception:
    pass  # Column already exists — safe to ignore
```

### Step 2: Create tools/contacts.py

Key tools: `add_contact`, `list_contacts`, `get_contacts_without_recent_followup`

The third tool is the one that makes this a real CRM feature:

```python
def get_contacts_without_recent_followup(days: int = 7) -> str:
    """
    Find contacts who have not been interacted with in the past N days.
    Use when user asks 'who haven't I followed up with?', 'which clients
    need attention?', or 'who should I call this week?'.
    
    This is a JOIN query — it looks at both the contacts table and the
    interactions table to find contacts with no recent activity.
    
    Args:
        days: How many days to look back. Default is 7. If user says 
              'recently' without specifying, use 7.
    
    Returns: List of contacts needing follow-up, or confirmation all are current.
    """
    inputs = {"days": days}
    
    try:
        conn = get_connection()
        cursor = conn.cursor()
        
        # This is a JOIN with a subquery — your first cross-entity query
        cursor.execute("""
            SELECT 
                c.id, c.name, c.phone, c.email, c.source,
                MAX(i.created_at) as last_contact
            FROM contacts c
            LEFT JOIN interactions i ON c.id = i.contact_id
            GROUP BY c.id
            HAVING last_contact IS NULL 
               OR last_contact < datetime('now', ? || ' days')
            ORDER BY last_contact ASC
        """, (f"-{days}",))
        
        contacts = cursor.fetchall()
        conn.close()
        
        if not contacts:
            return f"All contacts have been followed up with in the last {days} days."
        
        lines = [f"Contacts needing follow-up ({len(contacts)} found):\n"]
        for c in contacts:
            last = c['last_contact'] or "Never contacted"
            lines.append(f"ID {c['id']}: {c['name']} ({c['source']}) | Last contact: {last}")
            if c['phone']:
                lines.append(f"  Phone: {c['phone']}")
        
        result = "\n".join(lines)
        log_tool_call("get_contacts_without_recent_followup", inputs, f"Returned {len(contacts)} contacts")
        return result
        
    except Exception as e:
        error = f"Database error: {str(e)}"
        log_tool_call("get_contacts_without_recent_followup", inputs, None, error)
        return f"Error: {error}"
```

### Step 3: Create tools/projects.py and tools/interactions.py

Follow the same pattern: `add_project`, `list_projects`, `get_overdue_projects`, `log_interaction`, `get_contact_history`.

The `get_contact_history` tool should return all interactions for a given contact — this requires a JOIN between interactions and contacts.

### Step 4: Cross-Entity Tool — The Star of the Project

Add this to a new `tools/reports.py` file. This is a multi-table query that showcases your architecture:

```python
def get_project_summary(project_id: int = None) -> str:
    """
    Get a full summary of one or all projects, including related tasks and contacts.
    Use when user asks for a project overview, status report, or wants to know
    everything about a project.
    
    This pulls data from projects, tasks, AND contacts in a single query —
    showing the full operational picture of the project.
    
    Args:
        project_id: Optional. If provided, show details for that specific project.
                    If None, show summary of all active projects.
    """
    # This tool will use multiple queries and JOIN them in Python
    # You'll implement this as you reach Phase 3
    pass
```

### Phase 3 Test Checklist

- [ ] "Add a project called 'Dr. Sangeeta Website' for client Dr. Sangeeta Agrawal, deadline end of June"
- [ ] "Add Rahul Sharma as a lead contact with phone 9876543210"
- [ ] "Log that I called Rahul today and he's interested in a quote"
- [ ] "Which contacts haven't I followed up with in 5 days?"
- [ ] "Show me all active projects"
- [ ] "Which projects are overdue?"
- [ ] "Summarize the Dr. Sangeeta Website project"

---

## Part 9: Phase 4 — Intelligence Layer (Days 12–14)

**Goal:** Handle ambiguity, partial information, and complex queries.

This is where your MCP server starts to feel like a real AI-native application rather than a structured form.

### 1. Fuzzy Name Matching

When a user says "find the Sharma guy," they mean one of your contacts. You need to handle partial, approximate names.

```python
def search_contacts(query: str) -> str:
    """
    Search for contacts by partial name, phone, email, or notes.
    Use when user mentions a person by partial name, nickname, or approximate
    description. This is a fuzzy search — exact match is not required.
    
    Args:
        query: Any part of the contact's name, phone, email, or notes.
               E.g., 'Sharma', '98765', 'doctor', 'from Agra'
    """
    conn = get_connection()
    cursor = conn.cursor()
    
    # SQLite's LIKE operator with wildcards on both sides = substring search
    search_term = f"%{query}%"
    
    cursor.execute("""
        SELECT id, name, phone, email, source, notes
        FROM contacts
        WHERE name LIKE ? 
           OR phone LIKE ? 
           OR email LIKE ? 
           OR notes LIKE ?
    """, (search_term, search_term, search_term, search_term))
    
    results = cursor.fetchall()
    conn.close()
    
    if not results:
        return f"No contacts found matching '{query}'. Try a different search term."
    
    lines = [f"Found {len(results)} contact(s) matching '{query}':\n"]
    for c in results:
        lines.append(f"ID {c['id']}: {c['name']} ({c['source']})")
        if c['phone']: lines.append(f"  Phone: {c['phone']}")
        if c['email']: lines.append(f"  Email: {c['email']}")
    
    return "\n".join(lines)
```

### 2. Natural Language Date Handling

Install `python-dateparser` and create a utility:

```python
# utils/date_utils.py
import dateparser
from datetime import datetime

def parse_date_to_iso(date_string: str) -> str:
    """
    Convert natural language dates to YYYY-MM-DD format for database storage.
    
    Examples:
        "tomorrow"      → "2026-05-29"
        "next Friday"   → "2026-06-01"  
        "end of month"  → "2026-05-31"
        "15 June"       → "2026-06-15"
    
    Returns the date string in ISO format, or None if parsing fails.
    """
    if not date_string:
        return None
    
    # Already in correct format
    if len(date_string) == 10 and date_string[4] == "-":
        return date_string
    
    parsed = dateparser.parse(
        date_string,
        settings={
            "PREFER_DATES_FROM": "future",
            "DATE_ORDER": "DMY"  # Adjust to YMD if you prefer
        }
    )
    
    if parsed:
        return parsed.strftime("%Y-%m-%d")
    
    return None  # Return None instead of crashing; let the caller handle it
```

Add this utility to your `add_task` tool — if `due_date` contains words instead of a date format, run it through `parse_date_to_iso` first.

### 3. The `get_schema` Debug Tool

Add this to `server.py`. It's invaluable for debugging and will impress interviewers because it shows systems thinking:

```python
@mcp.tool()
def get_database_schema() -> str:
    """
    Return the current database schema — all tables and their columns.
    Use this when you need to understand what data is available before
    constructing a complex query, or when the user asks what data is stored.
    This is a diagnostic and exploration tool.
    """
    conn = get_connection()
    cursor = conn.cursor()
    
    cursor.execute("SELECT name FROM sqlite_master WHERE type='table' ORDER BY name")
    tables = cursor.fetchall()
    
    output = []
    for table in tables:
        table_name = table[0]
        cursor.execute(f"PRAGMA table_info({table_name})")
        columns = cursor.fetchall()
        
        cursor.execute(f"SELECT COUNT(*) as count FROM {table_name}")
        count = cursor.fetchone()["count"]
        
        col_info = ", ".join([f"{col['name']} ({col['type']})" for col in columns])
        output.append(f"Table '{table_name}' ({count} rows): {col_info}")
    
    conn.close()
    return "\n".join(output)
```

---

## Part 10: Phase 5 — Production Hardening (Days 15–17)

**Goal:** The difference between a demo and a real system.

### 1. Input Sanitization

SQL injection through natural language is a real vector. If Claude passes user input directly to a query, a malicious user could say "delete tasks where title = '' OR 1=1--" and cause damage.

**Always use parameterized queries** — never string-format SQL:

```python
# NEVER do this:
cursor.execute(f"SELECT * FROM tasks WHERE title = '{user_input}'")

# ALWAYS do this (the ? placeholder prevents injection):
cursor.execute("SELECT * FROM tasks WHERE title = ?", (user_input,))
```

Also add a sanitizer utility:

```python
# utils/validators.py
import re

def sanitize_text_input(text: str, max_length: int = 500) -> str:
    """Remove null bytes and truncate overly long inputs."""
    if not text:
        return ""
    cleaned = text.replace("\x00", "")  # Remove null bytes
    cleaned = cleaned[:max_length]       # Truncate
    return cleaned.strip()

def validate_date_format(date_str: str) -> bool:
    """Check if a date string is valid YYYY-MM-DD format."""
    if not date_str:
        return True  # None is valid (optional field)
    pattern = r"^\d{4}-\d{2}-\d{2}$"
    return bool(re.match(pattern, date_str))
```

### 2. Structured Logging Upgrade

Upgrade your logger to include session tracking:

```python
# Add to utils/logger.py
import uuid

# Generate a session ID when the server starts
SESSION_ID = str(uuid.uuid4())[:8]

def log_tool_call(tool_name: str, inputs: dict, output: str, error: str = None):
    entry = {
        "session": SESSION_ID,  # Groups all calls from one Claude session
        "timestamp": datetime.now().isoformat(),
        "tool": tool_name,
        "inputs": inputs,
        "output_preview": str(output)[:200] if output else None,  # Don't log huge outputs
        "error": error,
        "status": "error" if error else "success"
    }
    # ... rest of function
```

### 3. n8n Webhook Bridge (Bonus — connects to your existing work)

You already have n8n running. Add a FastAPI endpoint that n8n can call to trigger your tools:

```bash
pip install fastapi uvicorn
```

Create `webhook_server.py`:

```python
from fastapi import FastAPI
from tools.tasks import add_task
from tools.contacts import add_contact, log_interaction

app = FastAPI(title="NL-Ops Engine Webhook Bridge")

@app.post("/webhook/add-task")
async def webhook_add_task(data: dict):
    """
    n8n calls this endpoint when a new task needs to be created.
    Example: when a form submission comes in via n8n, auto-create a follow-up task.
    """
    result = add_task(
        title=data.get("title"),
        due_date=data.get("due_date"),
        priority=data.get("priority", "medium")
    )
    return {"result": result}

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

In n8n, add an HTTP Request node that POSTs to `http://localhost:8000/webhook/add-task`. Now your n8n automations (like your KP Coaching lead capture) can write directly to your NL-Ops database. **This is the production integration that makes this project real.**

---

## Part 11: What To Do When Things Break

### Debugging Strategy (In Order)

**Step 1: Check Claude Desktop logs**

Claude Desktop writes server logs to a file. Find it at:
- Windows: `%APPDATA%\Claude\logs\`
- Mac: `~/Library/Logs/Claude/`

Look for error messages from your server. This is the first place to check.

**Step 2: Run your server manually**

```bash
python server.py
```

If it crashes, you'll see the full Python traceback. Fix the crash before testing with Claude Desktop.

**Step 3: Check tool_calls.jsonl**

Every tool call is logged. If a tool was called but returned an error, it's in this file with the full error message.

**Step 4: Add a test input directly**

Add a temporary `if __name__ == "__main__"` block to your tool file and test the function directly with Python, bypassing Claude entirely:

```python
if __name__ == "__main__":
    print(add_task("Test task", "2026-06-01", "high"))
    print(list_tasks())
```

**Step 5: Claude isn't calling your tool**

This is almost always a description problem. Make the description more explicit: add more trigger phrases, more examples of when to use it.

### The "Works Locally, Fails With Claude" Pattern

If your tool works when you call it directly in Python but Claude can't use it correctly:
- The tool description doesn't match the user's phrasing
- The parameter names are confusing Claude
- Claude is passing the right intent but wrong format (e.g., "Friday" instead of "2026-05-30")

Solution: Add explicit format guidance in the description, and add conversion logic in your tool function for common mistakes.

---

## Part 12: Talking About This Project in Interviews

You will be asked: *"Tell me about a project you built."*

Here is the answer structure:

**The problem:** "I wanted to build an AI-native operations backend that could be queried in plain English — something I could actually use for managing my agency's clients, projects, and tasks."

**The architecture:** "I built an MCP server using FastMCP and Python. The server exposes a set of tools over the Model Context Protocol that Claude Desktop can call. Claude interprets the user's natural language query, decides which tool to call, my server executes the SQL against a SQLite database, and Claude narrates the result."

**The hard part:** "The technically interesting challenge was designing tool descriptions that reliably guide Claude's tool selection. A tool isn't just a function — the description is effectively a prompt that trains Claude when to call it. Getting this right required iteration, and I learned a lot about how LLMs make decisions at the tool-selection level."

**The production angle:** "I built observability in from Phase 1 — every tool call is logged as structured JSON with timestamp, inputs, outputs, and error status. I also added a FastAPI webhook bridge that lets n8n automations write directly to the database, so it integrates with my existing automation workflows."

**The scale:** "It's a multi-entity relational system — tasks, contacts, projects, and interactions — with JOIN queries for cross-entity questions like 'which clients haven't been followed up with this week?'"

---

## Part 13: Quick Reference Card

### The Three Questions To Ask Before Writing Any Tool

1. **When should Claude call this?** → Write that in the description, with examples
2. **What can go wrong with the input?** → Add validation before any database call
3. **Is this being logged?** → Every tool must call `log_tool_call` before returning

### SQLite Cheat Sheet

```sql
-- Insert
INSERT INTO tasks (title, priority) VALUES (?, ?)

-- Select with filter
SELECT * FROM tasks WHERE status = ? AND priority = ?

-- Update
UPDATE tasks SET status = ? WHERE id = ?

-- Delete with condition
DELETE FROM tasks WHERE status = 'done' AND created_at < datetime('now', '-7 days')

-- JOIN (tasks with their project names)
SELECT t.title, p.name as project_name
FROM tasks t
LEFT JOIN projects p ON t.project_id = p.id

-- Aggregate (count tasks per status)
SELECT status, COUNT(*) as count FROM tasks GROUP BY status
```

### Phase Completion Checklist

- [ ] Phase 0: Can answer all 5 conceptual questions
- [ ] Phase 1: `ping` and `hello` work in Claude Desktop
- [ ] Phase 2: 4 task tools work + logging is running + tested all 8 test cases
- [ ] Phase 3: 3 more entities added + cross-entity query works + JOIN tested
- [ ] Phase 4: Fuzzy search works + date parsing handles "tomorrow" and "next week"
- [ ] Phase 5: Input sanitization in all tools + session logging + n8n webhook working

---

## Final Note

The project is a vehicle. The real learning is:
- How AI clients and tool servers communicate (MCP protocol)
- How to design tool interfaces that LLMs use reliably (AI tool design)
- How to build observable systems where you can audit exactly what happened (telemetry)
- How relational data models support complex natural language queries (database design)

Every one of these transfers directly to production AI engineering roles.

Build in phases. Test each phase fully. Don't skip the logging. Come back and ask questions when you're stuck — you will get stuck, and that's where the real learning happens.
