# AGENT.md — Folio Book Finder

This file tells any AI coding agent everything it needs to understand, modify, and extend this project correctly. Read this before touching any file.

---

## Project Summary

**Folio** is a mobile-first, single-page web app (one `index.html` file) that lets users search for books via the OpenLibrary API and save them to a personal reading list stored in Firebase Firestore. Each visitor gets an anonymous Firebase UID so their list is private and persistent across sessions — no account creation required.

---

## Architecture

### Single-file app
Everything — HTML structure, CSS styling, and JavaScript logic — lives in `index.html`. There is no build step, no bundler, no Node.js. The app runs directly in the browser and is hosted on GitHub Pages.

### Modules loaded via CDN
```html
<!-- Firebase v11 via ESM CDN -->
import { initializeApp }   from "https://www.gstatic.com/firebasejs/11.0.0/firebase-app.js";
import { getAuth, ... }    from "https://www.gstatic.com/firebasejs/11.0.0/firebase-auth.js";
import { getFirestore, ... } from "https://www.gstatic.com/firebasejs/11.0.0/firebase-firestore.js";
```
Do NOT switch to npm or a bundler — the GitHub Pages hosting requires zero build step.

### External APIs
- **OpenLibrary Search API** — `https://openlibrary.org/search.json?title=...&fields=key,title,author_name,first_publish_year,cover_i,number_of_pages_median,publisher,language`
- **OpenLibrary Works API** — `https://openlibrary.org/works/{wk}.json` — used in the modal to fetch book descriptions
- **OpenLibrary Covers** — `https://covers.openlibrary.org/b/id/{cover_i}-M.jpg` (M = medium, L = large)

No API keys are needed for any of these.

---

## Key State Variables

| Variable | Type | Purpose |
|---|---|---|
| `uid` | string | Firebase anonymous user ID |
| `readingMap` | `{ [key]: bookDoc }` | In-memory mirror of Firestore snapshot |
| `listFilter` | `"all" \| "want" \| "read"` | Active reading list filter |
| `searchFilter` | `"title" \| "author" \| "subject"` | Active OpenLibrary search field |
| `activePanel` | `"search" \| "library"` | Currently visible panel |

---

## Core Functions

### `search(q: string)`
Builds the OpenLibrary URL based on `searchFilter`, fetches results, and calls `renderResults()`. Debounced at 400ms from the input event.

### `saveBook(book: OLBookObject)`
Checks `readingMap` for duplicates first. If clean, writes a new document to `users/{uid}/books/{key}` in Firestore.

### `removeBook(k: string)`
Deletes a document from `users/{uid}/books/{k}`.

### `patchBook(k: string, data: object)`
Updates specific fields on an existing library document (status, rating).

### `openModal(book: OLBookObject)`
Renders the bottom-sheet modal with cover, metadata tags, and an async-loaded description from the Works API.

### `renderLibrary()`
Reads from `readingMap`, applies `listFilter`, splits into want/read sections, and renders `.rl-card` elements into `#library-grid`.

### `subscribeLibrary()`
Called once after auth. Sets up a Firestore `onSnapshot` listener that keeps `readingMap` live and re-renders the library and search badges automatically.

### `key(book)`
Derives a stable Firestore document ID from `book.key` (OpenLibrary path). Strips non-alphanumeric chars, caps at 80 chars.

---

## UI Components

### Bottom Navigation
Two tabs: **Search** and **Library**. Controlled by `.nav-btn[data-nav]` buttons. Switching calls `switchPanel(name)`.

### Search Panel (`#panel-search`)
Contains the results shelf grid. Tall portrait cards (`aspect-ratio: 2/3`). Tapping a card opens the modal.

### Library Panel (`#panel-library`)
Contains filter chips (`data-lf`) and the `#library-grid`. Renders two shelf sections (Want / Read) with `.rl-card` elements.

### Modal (bottom sheet)
Triggered by `openModal()`. On mobile it slides up from the bottom. On desktop (≥640px) it centers as a dialog. Close by tapping the overlay, the ✕ button, or the CTA button.

### Toast
Single `#toast` element. Call `toast("message")` from anywhere. Auto-hides after 2.6s.

---

## CSS Design System

### Design direction
**Editorial literary magazine** — cream paper background, ink tones, Cormorant Garamond serif display font, Jost sans-serif body. Accent color is burnt sienna (`#8b3a2a`).

### Key CSS variables
```css
--cream:   #f5f0e8   /* warm off-white background surface */
--paper:   #faf7f2   /* page background */
--ink:     #1a1612   /* primary text */
--ink3:    #7a6f65   /* muted / placeholder text */
--rule:    #d6cec2   /* borders and dividers */
--accent:  #8b3a2a   /* burnt sienna — CTAs, badges, active states */
--accent2: #c4773a   /* amber — year labels, read-status highlights */
--nav-h:   64px      /* bottom nav height */
--safe-b:  env(safe-area-inset-bottom, 0px)  /* iPhone notch safe area */
```

### Responsive breakpoints
- Mobile-first base styles for ≤479px
- `@media (min-width: 480px)` — wider shelf columns
- `@media (min-width: 640px)` — modal becomes centered dialog, drag handle hidden
- `@media (min-width: 700px)` — even wider shelf columns
- `max-width: 900px` on all content containers — centers on large screens

---

## Firestore Data Schema

Collection path: `users/{uid}/books/{bookKey}`

```js
{
  key:     string,   // Firestore doc ID, derived from OL book.key
  title:   string,
  author:  string,   // first author only
  year:    string,
  coverId: number | null,  // OpenLibrary cover ID
  olKey:   string,   // e.g. "/works/OL123W"
  status:  "want" | "read",
  rating:  number,   // 0 = unrated, 1–5 stars
  addedAt: number,   // Date.now() timestamp
}
```

---

## Security Rules (Firestore)

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /users/{userId}/books/{bookId} {
      allow read, write: if request.auth != null && request.auth.uid == userId;
    }
  }
}
```

---

## What to Watch Out For

1. **Firebase config placeholder** — `index.html` ships with `"YOUR_API_KEY"` etc. Any agent modifying the file must NOT overwrite a real config if one is already present.

2. **No build step** — do not introduce `import` from `node_modules`, `npm install`, webpack, vite, or any bundler. All dependencies load from CDN URLs.

3. **`x()` escaping function** — always wrap user-facing strings and API data in `x()` before inserting into `innerHTML` to prevent XSS.

4. **`key()` function** — generates Firestore doc IDs. Do not change its logic; existing saved books depend on it being stable.

5. **Async modal description** — the Works API call in `openModal` is non-blocking. Always check that `document.getElementById("modal-desc")` still exists before writing to it (the user may have closed the modal).

6. **`onSnapshot` is the source of truth** — never update `readingMap` directly. All writes go to Firestore and flow back through the snapshot listener.

7. **Safe area insets** — `var(--safe-b)` and `env(safe-area-inset-bottom)` are critical for iPhone home indicator clearance. Don't remove them.

---

## Common Tasks

### Add a new field to saved books
1. Add it to the `setDoc` call in `saveBook()`
2. Use it in `rlCard()` for display
3. Add a `patchBook()` call for any UI control that changes it

### Add a new search filter
1. Add a `<button class="chip" data-f="newtype">` in the chips row
2. Add the new type to the `map` object inside `search()`

### Add a new bottom nav tab
1. Add a `<button class="nav-btn" data-nav="name">` in `.bottom-nav`
2. Add a `<div class="panel" id="panel-name">` in `.content`
3. Handle any data loading in `switchPanel()`

### Change the color scheme
Update CSS variables in `:root`. The entire UI is token-driven — no hardcoded colors elsewhere.

---

## Out of Scope (don't add without discussion)
- Google / email authentication (planned but not yet scoped)
- Backend functions or Cloud Functions
- Any npm dependency or build tool
- Server-side rendering
