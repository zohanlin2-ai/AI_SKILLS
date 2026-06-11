# GitHub Project Upload & Management Workflow

This document defines the standardized SOP for uploading and managing projects within the `AI_SKILLS` repository.

## 1. Core Information
- **Default Repository**: `https://github.com/zohanlin2-ai/AI_SKILLS.git`
- **Structure Principle**: Multi-project architecture. Each project must reside in its own subdirectory named after its original folder.
- **Root Documentation**: The root `README.md` serves as a project index and must be maintained in **English**.

## 2. Standard Workflow (SOP)

### Step 1: Remote Detection
Always start by checking the remote repository state:
```bash
git clone https://github.com/zohanlin2-ai/AI_SKILLS.git
```

### Step 1.5: Root Documentation Audit
Before making any staging or commit operations, verify and manage the root documents:
1. **Check Existence**: Verify if root `README.md` and `CHANGELOG.md` exist at the top level.
   - **If Missing**: Create them immediately using the standard templates.
2. **Audit README Index**:
   - **New Project Folder**: Write a concise, 1-2 sentence description introducing the new folder's contents and add it to the project list in the root `README.md`.
   - **Modified Project Folder**: Inspect the changes made in the folder. Compare them with the folder's existing summary in the root `README.md`. Update the description if the folder's scope, technology stack, or functionality has changed significantly.
3. **Draft CHANGELOG entry**: Document the version release or updates in `CHANGELOG.md`.

### Step 2: Scenario Branching

#### Scenario A: Repository is Empty (or Initial Setup)
1. **Initialize Documents**: Create root `README.md` (with Project Index table) and `CHANGELOG.md` (documenting `v1.0.0` initial upload).
2. **Prepare Projects**: Place each project inside its own subdirectory.
3. **Upload**: Stage all files (including root documents), commit, and push to the `main` branch.

#### Scenario B: Repository has Existing Content
1. **Check Project State**: Verify if a folder with the same name as the target project already exists in the remote repository.
2. **If Project DOES NOT Exist (New Project)**:
   - Create the project subfolder.
   - Put project files inside.
   - Run the **Root Documentation Audit** to add the folder description to root `README.md`.
   - Update `CHANGELOG.md` detailing the added project.
   - Stage, commit, and push.
3. **If Project ALREADY Exists (Conflict/Update)**:
   - Check if the local changes differ from the remote.
   - Request user authorization if major conflicts exist:
     - **(A) Update Local**: Pull remote changes to local.
     - **(B) Update Remote**: Push local changes to remote (overwrite/update).
     - **(C) Cancel**: Abort operation.
   - Run the **Root Documentation Audit** to review if the folder's description in root `README.md` needs revision.
   - Append a change log entry under the `Changed` or `Fixed` sections in `CHANGELOG.md`.
   - Stage, commit, and push.

## 3. Implementation Rules

- **Naming Rule**: Subfolders MUST use the current project's folder name. No manual renaming unless requested by the user.
- **Index Maintenance**: Root `README.md` must be updated under the "Project List" whenever a folder is added or undergoes changes that render the existing description inaccurate.
- **Changelog Requirement**: Every single upload/commit must append a corresponding version entry or changelog details in `CHANGELOG.md` following [Keep a Changelog](https://keepachangelog.com/en/1.0.0/) formats.
- **English Requirement**: All content in the root `README.md` and `CHANGELOG.md` must be written in **English**.
- **Safety**: Never perform a `force push` or major structural change without explicit user confirmation when a project already exists on the remote.

---
*Created by Antigravity AI Assistant.*
