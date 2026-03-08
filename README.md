# CollabSheets

A lightweight, real-time collaborative spreadsheet — built for the Trademarkia frontend engineering assignment by Hariharan G.

**Stack:** Next.js 14 (App Router) · TypeScript · Tailwind CSS · Firebase (Firestore + Auth)

**Live demo:** _add your Vercel URL here_

---

## Features

| Feature | Details |
|---|---|
| **Document Dashboard** | Lists all spreadsheets with title, last-modified timestamp, and creator name. Click to open, trash icon to delete. |
| **Spreadsheet Editor** | 100 × 26 scrollable grid. Rows numbered, columns lettered (A–Z). Keyboard navigation (arrows, Tab, Enter, F2, Delete). |
| **Formula Engine** | Supports `=SUM(A1:B3)`, `=AVERAGE(...)`, `=MIN(...)`, `=MAX(...)`, `=COUNT(...)`, and any arithmetic expression across cell references (`=A1*B2+C3`). Circular-reference guard included. |
| **Real-time sync** | Firestore `onSnapshot` keeps every open session in sync. Optimistic local updates eliminate perceived latency. |
| **Write-state indicator** | A pill badge in the top bar shows **Saving…** → **All changes saved** → idle. Errors surface inline. |
| **Presence** | Each collaborator's cursor position is written to `documents/{id}/presence`. Avatars appear in the top bar; a colored corner dot marks any cell that another user has selected. |
| **Identity** | First-time users set a display name (anonymous Firebase auth) **or** sign in with Google. A deterministic color is derived from the UID and stays consistent across sessions. |
| **No TS errors** | `strict: true` throughout. Zero `// @ts-ignore` comments. Builds clean on Vercel. |

---

## Architecture decisions

### Where state lives
- **Firestore** is the source of truth for cell data and presence.
- A `localCells` ref acts as a short-lived optimistic layer. It is merged on top of server state until the Firestore snapshot confirms the write, at which point it is cleared. This gives zero-flicker edits without a separate state-management library.

### Real-time contention
- Last-write-wins via `setDoc`. Each cell is its own Firestore document (`documents/{id}/cells/{row}:{col}`), keeping writes granular and minimising conflicts.
- Presence uses `setDoc` (not `updateDoc`) so writes are idempotent.

### Formula depth
The formula parser covers:
1. `=SUM / AVERAGE / MIN / MAX / COUNT` with cell ranges (`A1:C3`) or individual refs.
2. Arbitrary arithmetic (`=A1*(B2+3)/C4`) via safe `Function` evaluation after substituting cell refs with their numeric values.
3. Recursive formula resolution (a formula cell can reference another formula cell).
4. Circular-reference detection via a `Set<string>` of visited keys passed through recursive calls.


## Setup

### Prerequisites
- Node.js ≥ 18
- A Firebase project with **Firestore** and **Authentication** (Google + Anonymous providers) enabled.

### 1. Clone & install
```bash
git clone <your-repo-url>
cd collab-sheets
npm install
```

### 2. Configure Firebase
```bash
cp .env.example .env.local
# Fill in your Firebase project values in .env.local
```

### 3. Firestore security rules
Paste these rules in the Firebase console (Firestore → Rules):

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /documents/{docId} {
      allow read, write: if request.auth != null;

      match /cells/{cellId} {
        allow read, write: if request.auth != null;
      }
      match /presence/{userId} {
        allow read: if request.auth != null;
        allow write: if request.auth != null && request.auth.uid == userId;
      }
    }
  }
}
```

### 4. Run locally
```bash
npm run dev
```

Open [http://localhost:3000](http://localhost:3000).

### 5. Deploy to Vercel
```bash
npx vercel --prod
```
Add your `.env.local` variables in the Vercel project settings under **Environment Variables**.

---

## Project structure

```
collab-sheets/
├── app/
│   ├── layout.tsx          # Root layout + AuthProvider
│   ├── page.tsx            # Home — auth gate → Dashboard
│   ├── globals.css
│   └── document/[id]/
│       └── page.tsx        # Editor page
├── components/
│   ├── AuthProvider.tsx    # Firebase auth context + color assignment
│   ├── SignIn.tsx          # Google / display-name sign-in UI
│   ├── Dashboard.tsx       # Document list + create/delete
│   ├── SpreadsheetEditor.tsx  # Main grid
│   ├── FormulaBar.tsx      # Address box + formula input
│   ├── PresenceIndicator.tsx  # Avatar cluster
│   └── SyncIndicator.tsx   # Save-state pill
├── lib/
│   ├── firebase.ts         # Firebase app init
│   ├── types.ts            # Shared TypeScript types
│   ├── formulaParser.ts    # Formula evaluation engine
│   └── hooks/
│       ├── useDocument.ts  # Document metadata real-time hook
│       ├── useCells.ts     # Cell data + sync-status hook
│       └── usePresence.ts  # Presence read/write hook
└── .env.example
```
