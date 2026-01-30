# Storage-Based Statement Management System

## Overview

This plan implements a comprehensive statement management system that sources statements directly from your Supabase Storage folder structure (`statements/kpcxprism/{month}/{client_name}/`) rather than relying solely on database records. This provides better file visibility, easier uploads, and automatic discovery of new statements.

---

## Current State Analysis

**Existing Implementation:**
- Statements currently come from `pending_investments.statement_path` column
- Paths follow pattern: `org-name/YYYY-MM/client-name.pdf`
- Hook: `useAllClientStatements.ts` queries database to build statement list
- Display: Grouped by Organization → Advisor → Client

**Your New Storage Structure:**
```text
statements/
└── kpcxprism/
    ├── January 2026/
    │   ├── JOHN_M_BEYER_IRA_ROLLOVER.pdf
    │   └── ROBERT_ANDREW_FLORES_MD_IRA.pdf
    └── December 2025/
        └── ...
```

**Current UI Features** (from screenshot):
- Organization grouping with advisor/client counts
- Sorting by Name, Date, Account
- Expand/Collapse controls
- "Verify Uploads" button
- "Select All" for bulk operations
- Document type sidebar (Statements, Subscription Docs, Tax Info)

---

## Proposed Solution

### Architecture

```text
┌─────────────────────────────────────────────────────────────────┐
│                    Admin Statement Manager                       │
├─────────────────────────────────────────────────────────────────┤
│  Storage Browser                │  Database Records             │
│  ──────────────────             │  ────────────────             │
│  • List folders/files           │  • pending_investments        │
│  • Upload new PDFs              │  • Auto-sync on upload        │
│  • Rename/move files            │  • Client associations        │
│  • Discover orphaned files      │  • Statement dates            │
└─────────────────────────────────────────────────────────────────┘
```

---

## Implementation Tasks

### Phase 1: Storage Browser Edge Function

Create `supabase/functions/list-storage-folder/index.ts`:

- Lists all files and folders in the statements bucket
- Returns hierarchical structure (org → month → files)
- Extracts metadata from filenames (client name, date)
- Compares with database to identify:
  - ✅ Synced (file exists + DB record exists)
  - ⚠️ Orphaned (file exists, no DB record)
  - ❌ Missing (DB record exists, no file)

**API Response Shape:**
```typescript
{
  folders: [{
    name: "kpcxprism",
    subfolders: [{
      name: "January 2026",
      files: [{
        name: "JOHN_M_BEYER_IRA_ROLLOVER.pdf",
        path: "kpcxprism/January 2026/JOHN_M_BEYER_IRA_ROLLOVER.pdf",
        size: 245000,
        syncStatus: "synced" | "orphaned" | "missing",
        linkedClientId: "uuid" | null,
        linkedClientName: "John M Beyer IRA" | null
      }]
    }]
  }]
}
```

### Phase 2: Admin Storage Manager Page

Create `src/pages/admin/StatementStorageManager.tsx`:

**UI Features:**
1. **Folder Tree Browser** - Navigate kpcxprism/months/files
2. **File List with Status** - Show sync status badges
3. **Bulk Actions:**
   - Sync orphaned files to database
   - Link files to existing clients
   - Delete files
4. **Upload Zone** - Drag-drop to specific month folder
5. **Reconciliation Dashboard** - Summary cards showing synced/orphaned/missing counts

**Mockup Layout:**
```text
┌────────────────────────────────────────────────────────────────┐
│ Statement Storage Manager                          [Upload] [Sync All]│
├────────────────────────────────────────────────────────────────┤
│ ┌──────────────┐ ┌─────────────────────────────────────────────┐│
│ │ Folder Tree  │ │ Files in: kpcxprism / January 2026          ││
│ │              │ │                                              ││
│ │ ▼ kpcxprism  │ │ ☐  JOHN_M_BEYER_IRA.pdf        ✅ Synced    ││
│ │   ▶ Jan 2026 │ │ ☐  ROBERT_FLORES_MD_IRA.pdf   ✅ Synced    ││
│ │   ▶ Dec 2025 │ │ ☐  DAVID_MONFORT_ROLLOVER.pdf ⚠️ Orphaned  ││
│ │   ▶ Nov 2025 │ │                                              ││
│ └──────────────┘ └─────────────────────────────────────────────┘│
└────────────────────────────────────────────────────────────────┘
```

### Phase 3: Auto-Sync Hook

Create `src/hooks/useStorageStatements.ts`:

- Fetches folder structure from edge function
- Provides file list with sync status
- Methods for syncing files to database
- Used by the Storage Manager page

### Phase 4: Enhanced Upload Flow

Update `src/components/documents/DocumentUploadModal.tsx`:

- Add folder picker for target month folder
- Auto-create month folders (e.g., "January 2026")
- On upload success, create corresponding `pending_investments` record
- Parse client name from filename or manual selection

### Phase 5: Integration with Existing Statements Page

Update `src/hooks/useAllClientStatements.ts`:

- Option to source from storage (live file list) OR database
- Toggle between "Database View" and "Storage View"
- Enables catching files that haven't been synced yet

---

## Technical Details

### Folder Naming Convention

Standardize on: `{org-slug}/{Month YYYY}/{CLIENT_NAME_FORMATTED}.pdf`

Examples:
- `kpcxprism/January 2026/JOHN_M_BEYER_IRA_ROLLOVER.pdf`
- `plangroup-financial/December 2025/TRANSFORMATION_TRUST.pdf`

### Client Name Parsing

Extract display name from filename:
```typescript
// Input: "INVCSD1610389_JOHN_M_BEYER_IRA_ROLLOVER_2..."
// Output: "John M Beyer IRA Rollover"

function parseClientNameFromFilename(filename: string): string {
  return filename
    .replace(/^INVCSD\d+_/, '')  // Remove invoice prefix
    .replace(/\.pdf$/i, '')      // Remove extension
    .replace(/_/g, ' ')          // Underscores to spaces
    .replace(/\b\w/g, c => c.toUpperCase()) // Title case
    .trim();
}
```

### Date Parsing from Folder Name

```typescript
// Input: "January 2026"
// Output: "2026-01-31" (last day of month for statement_date)

function parseStatementDate(folderName: string): string {
  const date = parse(folderName, 'MMMM yyyy', new Date());
  return format(endOfMonth(date), 'yyyy-MM-dd');
}
```

---

## Files to Create

| File | Purpose |
|------|---------|
| `supabase/functions/list-storage-folder/index.ts` | List storage contents with sync status |
| `supabase/functions/sync-storage-statements/index.ts` | Bulk sync orphaned files to database |
| `src/pages/admin/StatementStorageManager.tsx` | Admin UI for storage management |
| `src/hooks/useStorageStatements.ts` | Hook for storage-based statement list |
| `src/components/storage/FolderTreeBrowser.tsx` | Folder navigation component |
| `src/components/storage/FileListTable.tsx` | File list with status badges |
| `src/components/storage/StorageSyncButton.tsx` | Sync action button |

## Files to Modify

| File | Changes |
|------|---------|
| `src/App.tsx` | Add route `/admin/statement-storage` |
| `src/components/documents/DocumentUploadModal.tsx` | Add folder picker, auto-sync |
| `src/hooks/useAllClientStatements.ts` | Add storage source option |
| `src/pages/Statements.tsx` | Add toggle for storage view |

---

## Route Addition

```typescript
// In App.tsx
<Route 
  path="/admin/statement-storage" 
  element={<ProtectedRoute><StatementStorageManager /></ProtectedRoute>} 
/>
```

Add navigation link in admin sidebar or header.

---

## Security Considerations

- Edge function uses `SUPABASE_SERVICE_ROLE_KEY` for storage access
- Admin role check before listing/syncing operations
- RLS on `pending_investments` already restricts access by organization

---

## Benefits

1. **Visual File Management** - See actual files in storage, not just DB records
2. **Orphan Detection** - Find files uploaded but not linked to clients
3. **Missing Detection** - Find DB records pointing to non-existent files
4. **Bulk Sync** - One-click sync all orphaned files
5. **Flexible Upload** - Upload directly to month folders
6. **Month Organization** - Clear folder structure by statement period

---

## Testing Checklist

After implementation:
- [ ] List files in kpcxprism folder structure
- [ ] Verify sync status shows correctly (synced/orphaned/missing)
- [ ] Upload new PDF to specific month folder
- [ ] Sync orphaned file to create database record
- [ ] Confirm synced file appears in main Statements page
- [ ] Test admin-only access restriction
