# Task One (Frontend): Branch Master Page

> This task is **synced** with `backend/tasks/task-one-branch.md`.
> Both files share the same **API Contract** (section 2). Do not change the contract on your own. If it must change, update BOTH files and tell your partner.

## 1. Goal

Build the **Branch Management** page (System Setup > Master Data > Branches). The admin can see all branches in a table, add a branch, edit it, and delete it.

You can start before the backend is ready by using temporary mock data, then switch to the real API.

## 2. API Contract (shared with backend)

Base URL: `http://localhost:5000/api/branches`

| Method | URL | Purpose |
|--------|-----|---------|
| GET | `/api/branches` | List all branches |
| GET | `/api/branches/:id` | Get one branch |
| POST | `/api/branches` | Create a branch |
| PUT | `/api/branches/:id` | Update a branch |
| DELETE | `/api/branches/:id` | Delete a branch |

### Branch object (what you receive)

```json
{
  "_id": "665f...",
  "branch_code": "CSE",
  "branch_name": "Computer Science and Engineering",
  "capacity": 120,
  "status": "Active",
  "createdAt": "2026-11-22T10:00:00.000Z",
  "updatedAt": "2026-11-22T10:00:00.000Z"
}
```

- `status` is exactly `Active` or `Inactive`.
- `capacity` means **annual intake capacity** (a number >= 1, whole number).
- There is **no HOD field** on a branch. The branch table is kept stable. HOD assignment lives in a separate mapping table (`batch_hod_map`) built in a later task. Do not add an HOD field, column or dropdown.

### What you send (POST and PUT body)

```json
{ "branch_code": "CSE", "branch_name": "Computer Science and Engineering", "capacity": 120, "status": "Active" }
```

### Responses

Success: `{ "success": true, "message": "...", "data": ... }` (the list response also has `count`).
Error: `{ "success": false, "message": "...", "errors": { "capacity": "..." } }` (`errors` is optional, field name -> message).

| Case | Status code | What the UI should do |
|------|-------------|----------------------|
| OK | 200 / 201 | Success toast, refresh the table |
| Validation failed | 400 | Show `errors[field]` under each input, or `message` as a toast |
| Not found | 404 | Error toast, refresh the table |
| Duplicate code | 409 | Show the message under the Branch Code input |
| Server error | 500 | Error toast |

## 3. What the page contains

1. **Page header**: breadcrumb `System Setup / Master Data / Branches`, title "Branch Management", short subtitle, and a **+ Add Branch** button.
2. **Summary cards (3)**: Total Branches, Active Branches, Inactive Branches. **Calculate these from the list**, there is no separate API.
3. **Branch table (AG Grid)** with columns:
   - Branch Code
   - Branch Name
   - Annual Intake Capacity
   - Status (shown as a badge: green for Active, grey for Inactive)
   - Created At (formatted date)
   - Actions (View, Edit, Delete icons)
   - Do NOT show an HOD column yet (the faculty module doesn't exist).
4. **Add/Edit modal**: one component used for both modes. Fields: Branch Code, Branch Name, Annual Intake Capacity, Status. Buttons: Save, Cancel.
5. **Delete confirmation dialog** before deleting.
6. **View**: a simple read-only modal or side panel showing all branch details.

## 4. Folder structure to create

Inside `frontend/src/` (this is the pattern every later module will follow):

```
frontend/src/
├── main.jsx                           (existing)
├── App.jsx                            (edit: render layout + branch page)
├── api/
│   ├── axiosInstance.js               (axios with baseURL from env)
│   └── branchApi.js                   (one function per endpoint)
├── components/
│   └── common/
│       ├── ConfirmDialog.jsx          (reusable "Are you sure?" dialog)
│       ├── StatusBadge.jsx            (green/grey badge)
│       └── StatCard.jsx               (icon + label + number)
├── layouts/
│   └── MainLayout.jsx                 (sidebar + content area)
├── pages/
│   └── systemSetup/
│       └── masterData/
│           └── branch/
│               ├── BranchPage.jsx         (page: state, API calls, composes the rest)
│               ├── BranchTable.jsx        (the AG Grid)
│               ├── BranchFormModal.jsx    (add/edit form)
│               └── BranchViewModal.jsx    (read-only details)
├── hooks/
│   └── useBranches.js                 (optional: fetch + CRUD state logic)
└── utils/
    └── formatDate.js
```

Also create `frontend/.env` with `VITE_API_URL=http://localhost:5000/api` and `.env.example` with the same key. Confirm `.env` is in `.gitignore`.

## 5. Step by step

### Step 1: Install packages
`axios`, `ag-grid-community`, `ag-grid-react`, `lucide-react`, `react-hot-toast` (for toasts). Also install `react-router-dom` if you want the sidebar to navigate (a simple sidebar with only Branches active is fine for now).

### Step 2: API layer (`src/api/`)
- `axiosInstance.js`: create an axios instance with `baseURL` = `import.meta.env.VITE_API_URL`.
- `branchApi.js`: export `getBranches`, `getBranchById`, `createBranch`, `updateBranch`, `deleteBranch`. Each just calls axios and returns the data. **No UI logic here.** Components never call axios directly.

### Step 3: Common components (`src/components/common/`)
- `StatusBadge`: takes `status`, renders a small rounded badge (green = Active, grey = Inactive).
- `StatCard`: takes an icon, a label and a value.
- `ConfirmDialog`: takes `open`, `title`, `message`, `onConfirm`, `onCancel`, and a loading flag. It will be reused for every delete in the project, so keep it generic.

### Step 4: Branch components (`pages/systemSetup/masterData/branch/`)

**`BranchPage.jsx`** is the "brain":
- Holds state: list of branches, loading, error, which modal is open, and the selected branch.
- Fetches the list on mount; refetches after every create/update/delete.
- Calculates the 3 card numbers from the list.
- Passes data and callbacks down to the table and the modals.

**`BranchTable.jsx`** is the AG Grid:
- Receives `rows`, `loading`, `onView`, `onEdit`, `onDelete` as props.
- Define column definitions as described in section 3.
- Use a custom cell renderer for Status (uses `StatusBadge`) and for Actions (Lucide `Eye`, `Pencil`, `Trash2` icons with tooltips/aria-labels).
- Turn on: sorting, a quick filter wired to a search input above the grid, pagination with a page-size selector, and a status filter. Show total record count.
- Show a loading state while fetching and a clear empty state ("No branches yet. Add your first branch.") when empty.
- Use `_id` as the row id (`getRowId`).
- AG Grid v33+ note: register modules once (`ModuleRegistry.registerModules([AllCommunityModule])`) and import its theme as the docs say. Check the version you installed.

**`BranchFormModal.jsx`**: used for Add and Edit:
- Props: `open`, `mode` (`add` or `edit`), `initialData`, `onSubmit`, `onClose`.
- Fields and validation (validate on submit and show inline errors under each input):
  - Branch Code: required, letters/numbers only, auto-uppercased as the user types, max ~10 chars.
  - Branch Name: required.
  - Annual Intake Capacity: required, whole number, at least 1.
  - Status: dropdown, Active / Inactive, default Active.
- Disable Save and show a spinner while the request runs (prevents double submit).
- If the backend returns 400 `errors` or 409, put those messages under the matching inputs.
- Edit mode pre-fills the form. Resetting the form when the modal closes avoids stale data.

**`BranchViewModal.jsx`**: read-only display of all fields.

### Step 5: Layout and App
- `MainLayout.jsx`: a left sidebar (logo/app name, System Setup > Master Data > Branches highlighted, other items as non-functional placeholders: User Management, Batch Management, etc.) and a content area. Use Lucide icons only.
- `App.jsx`: render `MainLayout` containing `BranchPage`, and mount the toast container once.

### Step 6: Behaviour rules
- After every successful action: toast + refetch the list.
- Delete: open `ConfirmDialog` -> confirm -> call API -> toast -> refetch. Cancel does nothing.
- Never show raw error objects. Use the `message` from the response, or a friendly fallback.
- No hard-coded URL anywhere except `.env`.

## 6. How to test (before telling your partner it's done)

With the backend running and seeded:

- [ ] Page loads with the 6 seed branches; cards show 6 / 5 / 1.
- [ ] Skeleton or loading state is visible while fetching; empty state appears if the list is empty.
- [ ] Search, sorting, pagination (change page size) and status filter work.
- [ ] Add a branch with valid data: success toast, appears in the table, cards update.
- [ ] Add with empty fields / capacity `0` / `10.5`: inline errors, no request sent.
- [ ] Add a duplicate code: error under the Branch Code input.
- [ ] Typing `cse` shows `CSE` in the input.
- [ ] Edit pre-fills the form; saving updates the row.
- [ ] View shows all details.
- [ ] Delete shows the confirmation; Cancel keeps the row; Confirm removes it.
- [ ] Double-clicking Save does not create two branches.
- [ ] Stop the backend and reload: friendly error, no crash.
- [ ] `npm run lint` and `npm run build` pass.

## 7. Using mock data while the backend isn't ready

In `branchApi.js` you can temporarily return the seed list below, shaped exactly like the contract. Remove the mock before finishing.

| branch_code | branch_name | capacity | status |
|---|---|---|---|
| CSE | Computer Science and Engineering | 120 | Active |
| ECE | Electronics and Communication Engineering | 120 | Active |
| EEE | Electrical and Electronics Engineering | 60 | Active |
| MECH | Mechanical Engineering | 60 | Active |
| CIVIL | Civil Engineering | 60 | Active |
| CHEM | Chemical Engineering | 60 | Inactive |

## 8. Definition of done

- All checks in section 6 pass.
- Folder structure matches section 4; components are small and reusable.
- Table is AG Grid and all icons are Lucide.
- No HOD field anywhere.
- No hard-coded API URL, no leftover mock data.
- Tell the backend partner if anything in the contract needed to change.
