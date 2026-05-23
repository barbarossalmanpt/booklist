# booklist# 📚 Folio — Personal Book Finder

A mobile-first web app to search any book or author, view details, and build a personal reading list that syncs across all your devices via Firebase Firestore.

Live on GitHub Pages · Data persists via Firestore · No login required

---

## Features

- **Instant search** — debounced live search against the OpenLibrary API (free, no API key)
- **Search filters** — switch between Title, Author, or Subject
- **Tall portrait shelf grid** — covers displayed like a real bookshelf
- **Book detail bottom sheet** — tap any cover to see description, year, page count, publisher, and language
- **Personal reading list** — saved to Firestore per anonymous user session
- **Want to Read / Read toggle** — one tap to move books between statuses
- **Star ratings** — rate books you've finished (1–5 stars)
- **Duplicate detection** — warns you if a book is already in your library
- **Offline banner** — notifies you when you lose connection
- **Mobile-first UI** — bottom navigation, bottom sheet modal, safe-area insets for iOS/Android

---

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend | HTML, CSS, Vanilla JS (ES Modules) |
| Database | Firebase Firestore |
| Auth | Firebase Anonymous Auth |
| Book Data | OpenLibrary API |
| Hosting | GitHub Pages |
| Fonts | Cormorant Garamond + Jost (Google Fonts) |

---

## Setup

### 1. Clone the repository

```bash
git clone https://github.com/YOUR_USERNAME/folio-book-finder.git
cd folio-book-finder
```

### 2. Create a Firebase project

1. Go to [console.firebase.google.com](https://console.firebase.google.com)
2. Click **Add project** and follow the steps
3. In your project, go to **Build → Firestore Database** and click **Create database**
   - Choose **Start in test mode** (you can tighten rules later)
   - Pick a region close to you
4. Go to **Build → Authentication**, click **Get started**, then enable **Anonymous** sign-in

### 3. Get your Firebase config

1. In your Firebase project, click the ⚙️ gear icon → **Project settings**
2. Under **Your apps**, click the **</>** (Web) icon to register a web app
3. Copy the `firebaseConfig` object

### 4. Add your config to `index.html`

Open `index.html` and find this block near the bottom:

```js
const firebaseConfig = {
  apiKey:            "YOUR_API_KEY",
  authDomain:        "YOUR_PROJECT.firebaseapp.com",
  projectId:         "YOUR_PROJECT_ID",
  storageBucket:     "YOUR_PROJECT.appspot.com",
  messagingSenderId: "YOUR_SENDER_ID",
  appId:             "YOUR_APP_ID"
};
```

Replace every `"YOUR_..."` value with your actual Firebase values.

### 5. Set Firestore security rules

In the Firebase console go to **Firestore → Rules** and paste:

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

This ensures each user can only read and write their own books.

### 6. Deploy to GitHub Pages

```bash
git add .
git commit -m "Initial deploy"
git push origin main
```

Then in your GitHub repo: **Settings → Pages → Source → Deploy from branch → main → / (root) → Save**

Your app will be live at `https://YOUR_USERNAME.github.io/folio-book-finder/`

---

## Project Structure

```
folio-book-finder/
├── index.html     # Entire app — HTML, CSS, and JS in one file
├── README.md      # This file
└── AGENT.md       # AI agent instructions for future development
```

---

## Firestore Data Model

```
users/
  {uid}/
    books/
      {bookKey}/
        key       : string   — unique ID derived from OpenLibrary key
        title     : string
        author    : string
        year      : string
        coverId   : number | null
        olKey     : string   — OpenLibrary /works/... path
        status    : "want" | "read"
        rating    : number   — 0–5
        addedAt   : number   — Unix ms timestamp
```

---

## Roadmap

- [ ] Google Sign-In (upgrade from anonymous auth, keeping existing books)
- [ ] Notes field on Read books
- [ ] Sort reading list by date / title / rating
- [ ] Share reading list via public URL
- [ ] PWA manifest + service worker for true offline support
- [ ] Swipe to delete on mobile

---

## Credits

- Book data — [OpenLibrary](https://openlibrary.org) (Internet Archive)
- Database — [Firebase](https://firebase.google.com)
- Fonts — [Google Fonts](https://fonts.google.com) (Cormorant Garamond, Jost)
