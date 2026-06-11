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

### Step 2: Scenario Branching

#### Scenario A: Repository is Empty (or Initial Setup)
1. **Initialize Index**: Create a root `README.md` with an introduction and a "Project List" section.
2. **Prepare Project**: 
   - Create a subfolder using the **exact name** of the current project folder.
   - Move all project files into this subfolder.
3. **Upload**: Stage all changes, commit, and push to the `main` branch.
4. **Update Index**: Ensure the root `README.md` reflects the new project and its description.

#### Scenario B: Repository has Existing Content
1. **Check for Existence**: Verify if a folder with the same name as the new project already exists in the remote repository.
2. **If Project DOES NOT Exist (New Project)**:
   - Create the project-specific subfolder in the local workspace.
   - Place project files inside.
   - Upload the new folder to the remote repository.
   - Update the root `README.md` index.
3. **If Project ALREADY Exists (Conflict/Update)**:
   - **STOP** and request user intervention.
   - Ask the user for the preferred action:
     - **(A) Update Local**: Pull changes from remote to local.
     - **(B) Update Remote**: Push local changes to remote (overwrite/update).
     - **(C) Cancel**: No action taken.

## 3. Implementation Rules

- **Naming Rule**: Subfolders MUST use the current project's folder name. No manual renaming unless requested by the user.
- **Index Maintenance**: Every new project upload must be followed by an update to the root `README.md` Project List section.
- **English Requirement**: All content in the root `README.md` must be in English.
- **Safety**: Never perform a `force push` or major structural change without explicit user confirmation when a project already exists on the remote.

---
*Created by Antigravity AI Assistant.*
