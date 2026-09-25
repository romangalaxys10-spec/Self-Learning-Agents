# ZCode agent session learning — 2026-09-24: Restore an accidentally archived project in ZCode

## Symptom
User archived a project (Deep-Init) in the ZCode desktop UI by accident; project/sessions vanished from the list.

## Root cause & fix
Archive is SOFT UI STATE, not deletion. Two stores exist and only one matters:

- `~/.zcode/cli/db/db.sqlite` `session.time_archived` — exists in schema but was NULL for every row; NOT what the desktop UI uses.
- **`~/.zcode/v2/tasks-index.sqlite` `tasks.archived` (INTEGER 0/1)** — this is the desktop app's real store (per workspace_key + task_id).

Fix = one SQL statement:
```sql
UPDATE tasks SET archived=0
 WHERE archived=1 AND workspace_path LIKE '%<ProjectName>%';
```
Verify with `SELECT ... FROM tasks WHERE workspace_path LIKE ...`. The app may cache the list in memory → refresh/reopen the panel (or restart the app) to see the project again.

## How to trace UI persistence in an Electron app (reusable method)
1. `strings` on Local Storage leveldb misses snappy-compressed blocks — don't trust empty grep results there.
2. Instead, extract the app bundle (`npx @electron/asar extract`) and grep the **host/main chunks** for the UI action name (e.g. `archiveTask`) → follow the chain: `archiveTask → updateIndexedTaskState(Bn) → C.updateTaskState → getTasksIndexDatabasePath() → tasks-index.sqlite`.
3. Resolve the actual data root by searching the disk for the filename (`mdfind -name tasks-index.sqlite`) — the desktop app and CLI can point at DIFFERENT roots (`~/.zcode/v2/` vs `~/.zcode/cli/`; the CLI one was 0 bytes).

## Key reassurance for users
Archiving never touches project files on disk. Check `ls -la <project-dir>` (avoid `| head` truncation!) before assuming data loss.
