# Software Requirements Specification — miniCHAT

**Version:** 1.0
**Status:** Locked scope — do not expand without explicit sign-off.

---

## 1. Purpose

miniCHAT is a minimal Chrome extension that lets two people chat via text using a shared Chat ID, with no accounts, no login, and no backend server. It exists primarily as a **learning project** to introduce an intern to core web development concepts: DOM manipulation, event handling, async JS, working with an external API, and basic Chrome extension architecture.

The project is intentionally small. Simplicity is a requirement, not a shortcut.

---

## 2. Scope

### In Scope
- A Chrome extension (Manifest V3) with a single popup UI
- Entering a Chat ID and a Name to join/create a chat room
- Real-time 1-to-1 text chat between exactly two participants
- Ending the chat via an "End" button, which deletes the room's data for both participants
- Data storage via the Firebase Realtime Database **REST API** (not the Firebase JS SDK)

### Out of Scope (v1)
- Any authentication or account system
- Handling a user closing the browser/popup without pressing "End" (no `onDisconnect` cleanup)
- Support for more than 2 participants in a room
- Message history/persistence beyond the live session
- Chat ID collision detection (see §7, Known Limitations)
- Styling polish / responsive design beyond basic usability

Anything not listed under "In Scope" is explicitly not part of v1. If it comes up during build, it goes on a future-improvements list, not into this version.

---

## 3. Definitions

| Term | Meaning |
|---|---|
| Chat ID | A string that identifies a chat room. The first person to enter one creates the room; the second person joins by entering the same string. |
| Room | A single node in the Firebase Realtime Database, keyed by Chat ID, holding both participants' info and messages. |
| Participant | A user who has joined a room (max 2). |

---

## 4. Functional Requirements

### FR1 — Join / Create Flow
- On opening the extension, the user is shown two inputs: **Chat ID** and **Name**.
- Submitting checks whether that Chat ID already exists as a room:
  - If it doesn't exist → create it, register the user as Participant 1.
  - If it exists with 1 participant → register the user as Participant 2.
  - If it exists with 2 participants already → reject the join with a clear message ("Room is full").
- *Clue: think about what a "room already has 2 participants" check requires you to read before you write.*

### FR2 — Chat Screen
- Once joined, show a simple message list and a text input + send control.
- Messages should appear for both participants without needing a manual refresh.
- *Clue: this is the part of the project that benefits most from research into how Realtime Database is meant to be read continuously rather than fetched once.*

### FR3 — Sending Messages
- A sent message should record at minimum: sender name, text, timestamp.
- Empty messages should not be sendable.

### FR4 — Ending the Chat
- Either participant can press "End".
- Pressing End must:
  1. Delete the entire room node (both participants' data and all messages) from the database.
  2. Log the pressing user out back to the join screen.
  3. Also return the **other** participant to the join screen — since the room data (and their connection to it) no longer exists, this should be a natural side effect of how the chat screen is kept in sync, not a separate mechanism.
- This is a deliberate design point: "End" is a shared action, not a personal logout.

### FR5 — No Auth
- No sign-up, sign-in, password, or session token of any kind.
- Anyone with a Chat ID can attempt to join it, subject to FR1's 2-participant cap.

---

## 5. Non-Functional Requirements

- **NFR1 — Simplicity:** Prefer plain JS, minimal dependencies. No frameworks unless there's a specific reason.
- **NFR2 — Communication method:** Firebase Realtime Database must be accessed via its REST API (plain `fetch` calls), not the Firebase JS SDK. This avoids known Manifest V3 content-security-policy conflicts with the SDK and doubles as a good exercise in working with HTTP/JSON directly.
- **NFR3 — UI:** Single popup window (`default_popup` in the manifest). No new-tab page, no side panel.
- **NFR4 — No local persistence:** The extension should not cache chat data in `localStorage`/`chrome.storage` beyond what's needed to hold the current session in memory.

---

## 6. Data Model (starting shape — not prescriptive)

Rooms should be stored under a top-level node, keyed by Chat ID. Each room needs to track its participants and its messages somehow.

*Clue: think about whether messages should be a list or a keyed object in Realtime Database, and why one might be easier to listen to for "new messages only" than the other.*

---

## 7. Known Limitations (accepted for v1, documented intentionally)

- **Open database rules:** Since there is no auth, Firebase security rules will be permissive. This is acceptable for a learning project but is a real security tradeoff — the intern should understand *why* this would not fly in a production system.
- **Chat ID collisions:** Two people could independently type the same Chat ID and be merged into a room together unintentionally. Not handled in v1.
- **No disconnect handling:** If a participant simply closes the popup, the other participant will not be notified and the room will remain until someone presses "End."

---

## 8. Future Improvements (explicitly not v1)

- Anonymous auth to tighten security rules without adding a login UX
- `onDisconnect` handling for graceful cleanup
- Chat ID collision avoidance (e.g. random suggested IDs)
- Message read receipts / typing indicators

---

## 9. Success Criteria

The project is complete when two separate installs of the extension, on two machines/profiles, can:
1. Create and join the same Chat ID
2. Exchange messages in near real time
3. Have either side press "End" and have both sides cleanly return to the join screen with the room data gone from the database
