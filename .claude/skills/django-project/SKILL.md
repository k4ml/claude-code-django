---
name: django-project
description: Generate a new Django project from the labzero-project starter template. Use when user wants to create a new Django project.
---

# Django Project Generator

## Overview

Generate a new Django project based on the labzero-project starter template. This skill automates the setup process by cloning the template and running the rename script.

## When to Use

- User wants to create a new Django project
- User asks to "start a new project", "create django app", or similar
- User wants to scaffold a Django project

## Instructions

### Step 1: Gather Requirements

Ask the user for:
1. **Project name** (required) - e.g., `myproject`, `blog`, `inventory`
2. **Project directory** (optional) - defaults to current directory or `{project_name}`

### Step 2: Clone the Template

Clone the labzero-project template to the target directory or current directory if user didn't specify **Project directory**:

```bash
git clone https://github.com/lalokalabs/labzero-project.git {project_name}
cd {project_name}
```
Or:-

```bash
git clone https://github.com/lalokalabs/labzero-project.git .
```

### Step 3: Run Rename Script

Run the rename script to update project name from `myapp` to the user's desired name:

```bash
./rename.sh myapp {project_name}
```

### Step 4: Initialize Submodules

```bash
git submodule update --init --recursive
```

### Step 5: Setup Environment

```bash
cp .env.example .env
```

### Step 6: Summary

Provide the user with:
- The project location
- Next steps for development

## Example Output

```
✓ Project 'myblog' created at ./myblog

Next steps:
  cd myblog
  make up # run postgres and redis
  make dev && make run # in separate terminal
  Visit http://localhost:8000/dashboard/
  Login: admin@myblog.co / picard data
```

## Notes

- The rename script updates: database name, settings, file/directory names
- Default user model is set to `{project_name}_user.User` - customize as needed
- For Vite frontend in dev mode, add to `.env`:
  ```
  DJANGO_UMIN_VITE_DEV_MODE=True
  DJANGO_UMIN_VITE_HMR_PORT=5173
  ```
