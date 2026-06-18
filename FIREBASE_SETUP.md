# Phase 2 — Firebase Firestore cloud sync (multi-user, Google login)

This swaps `localStorage` for Firebase Firestore with **one-click Google sign-in**, so each person who logs in gets their own coffee log, synced live across their devices. Because all persistence in `index.html` is isolated in the **Storage Layer** (`loadData` / `saveData` and the accessors that call them), the data plumbing only touches that one section. You also add a small sign-in gate and a sign-out button.

## The key idea: a cache mirrors Firestore

Your accessors (`getBags`, `upsertBag`, `addLog`, …) are **synchronous** — they call `loadData()` and `saveData()` and then re-render. Firestore is **asynchronous**, so rather than rewriting every accessor with `await`, you keep a small in-memory object (`_cache`) that mirrors the signed-in user's Firestore document:

- `loadData()` returns `_cache` instantly (synchronous, like before).
- `saveData(data)` updates `_cache` **and** pushes it to that user's document in the background.
- A Firestore listener keeps `_cache` fresh and re-renders on any change — including edits made on another device — which is what gives you live sync.

Because the document depends on **who is logged in**, you don't pick it at startup. You attach the listener *after* sign-in and detach it on sign-out, so one user's data never bleeds into another's.

---

## Step 1 — Create the Firebase project & enable Google sign-in

1. Go to https://console.firebase.google.com and **Add project** (any name). Analytics optional.
2. **Build → Firestore Database → Create database** → start in **production mode**, pick the nearest region.
3. **Build → Authentication → Get started → Sign-in method → Google → Enable** (set a support email), then **Save**.
4. **Project settings (gear) → General → Your apps → web icon `</>`**, register an app, and copy the `firebaseConfig` object for Step 3.

## Step 2 — Add the SDK to `index.html`

Your file uses a normal `<script>` (not a module), so use the **compat** builds, which expose a global `firebase`. Add these in `<head>`, **before** your main `<script>`. Match the version shown in your Firebase console snippet:

```html
<script src="https://www.gstatic.com/firebasejs/10.12.2/firebase-app-compat.js"></script>
<script src="https://www.gstatic.com/firebasejs/10.12.2/firebase-firestore-compat.js"></script>
<script src="https://www.gstatic.com/firebasejs/10.12.2/firebase-auth-compat.js"></script>
```

## Step 3 — Replace the Storage Layer

In `index.html`, find the section marked `STORAGE LAYER` / `PHASE 2 — FIREBASE MIGRATION POINT`. Replace **only** `loadData()` and `saveData()` with the block below, and add the init code above them. Leave `genId`, `getBags`, `getBag`, `upsertBag`, `deleteBag`, `addLog`, `updateLog`, `deleteLog` exactly as they are.

Note `DOC` starts as `null` — it's set to the user's document only after they sign in.

```js
// ---- Firebase init ----
var firebaseConfig = {
  apiKey: "…",            // paste your config from Step 1
  authDomain: "…",
  projectId: "…",
  appId: "…"
};
firebase.initializeApp(firebaseConfig);
var db = firebase.firestore();
var DOC = null;            // set after sign-in to users/{uid}

// Optional but recommended: keep working offline, like localStorage did
db.enablePersistence({ synchronizeTabs: true }).catch(function () {});

// ---- in-memory cache mirrors the signed-in user's document ----
var _cache = { bags: [] };

function loadData() {
  // Synchronous read from the cache (the auth listener keeps it fresh)
  return _cache;
}

function saveData(data) {
  _cache = (data && Array.isArray(data.bags)) ? data : { bags: [] };
  if (DOC) DOC.set(_cache).catch(function (e) { console.error("Firestore save failed", e); });
}
```

## Step 4 — Replace the boot line with an auth gate

At the very bottom of the script, replace the single line `render();` with the block below. `onAuthStateChanged` fires on first load (remembering returning users), on sign-in, and on sign-out — and it's where you attach/detach the Firestore listener.

```js
var unsub = null;

firebase.auth().onAuthStateChanged(function (user) {
  if (unsub) { unsub(); unsub = null; }        // detach the previous user's listener
  if (!user) {                                 // signed out
    DOC = null;
    _cache = { bags: [] };
    showLogin();
    return;
  }
  hideLogin();                                 // signed in
  DOC = db.collection("users").doc(user.uid);  // this user's own document
  unsub = DOC.onSnapshot(function (snap) {
    var d = snap.exists ? snap.data() : null;
    _cache = (d && Array.isArray(d.bags)) ? d : { bags: [] };
    render();
  }, function (e) { console.error("Firestore listen failed", e); render(); });
});

function signIn() {
  var provider = new firebase.auth.GoogleAuthProvider();
  // Popups are often blocked on phones — use redirect on mobile, popup on desktop.
  if (/Mobi|Android|iPhone|iPad/i.test(navigator.userAgent)) {
    firebase.auth().signInWithRedirect(provider);
  } else {
    firebase.auth().signInWithPopup(provider).catch(function (e) { console.error(e); });
  }
}

function signOutUser() { firebase.auth().signOut(); }
```

With `signInWithRedirect`, Firebase completes the sign-in automatically when the page reloads and `onAuthStateChanged` fires — no extra handling needed for the basic flow.

## Step 5 — Add the sign-in screen and a sign-out button

Add these two small UI helpers anywhere in the UI section of your script. The login screen reuses your existing `.overlay` / `.recipe-sheet` styles, so it already matches the theme.

```js
function showLogin() {
  if (document.getElementById("loginScreen")) return;
  var s = document.createElement("div");
  s.id = "loginScreen";
  s.className = "overlay";
  s.style.justifyContent = "center";
  s.innerHTML =
    '<div class="recipe-sheet" style="border-radius:22px;max-width:340px;text-align:center;padding:40px 28px;">' +
      '<div style="font-size:2.6rem;">&#9749;</div>' +
      '<div class="recipe-title">Grind Log</div>' +
      '<div class="bag-sub" style="margin:8px 0 22px;">Sign in to sync your coffee logs across your devices.</div>' +
      '<button class="btn btn-primary" id="googleBtn">Sign in with Google</button>' +
    '</div>';
  document.getElementById("app").appendChild(s);
  document.getElementById("googleBtn").onclick = signIn;
}

function hideLogin() {
  var s = document.getElementById("loginScreen");
  if (s) s.remove();
}
```

For sign-out, wire `signOutUser()` to any control you like. The tidiest spot is the Home header — in `renderHome`, the header's action slot is used for "Recipes", so add sign-out to the footer of the Home view instead. For example, just before `view.innerHTML = html;` finishes in `renderHome`, append a button:

```js
// inside renderHome, after building `html`
html += '<button class="btn btn-secondary" style="margin-top:24px;" ' +
        'onclick="signOutUser()">Sign out</button>';
```

(If you don't already have a `.btn-secondary` style, reuse `.btn-danger` or add: `.btn-secondary{background:var(--surface-2);color:var(--text);border:1px solid var(--border);}`)

## Step 6 — Bring existing logs across (one-time, per user)

Old `localStorage` data won't move automatically, and it now belongs to a specific signed-in user. While signed in as that user, run this once in the browser console (or drop it in temporarily and remove it after) to seed *their* document if it's empty:

```js
(function () {
  var u = firebase.auth().currentUser;
  if (!u) { console.log("Sign in first."); return; }
  var doc = firebase.firestore().collection("users").doc(u.uid);
  doc.get().then(function (snap) {
    if (!snap.exists) {
      try {
        var old = JSON.parse(localStorage.getItem("coffeeLog"));
        if (old && old.bags && old.bags.length) doc.set(old);
      } catch (e) {}
    }
  });
})();
```

---

## Step 7 — Security rules (this is what keeps users separate)

In **Firestore → Rules**, publish the per-user rule below. With multiple people using the same project, these rules are the *only* thing preventing one user from reading or overwriting another's logs — don't skip them:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /users/{uid} {
      allow read, write: if request.auth != null && request.auth.uid == uid;
    }
  }
}
```

**Optional — restrict who can sign in.** By default, anyone with a Google account can sign in and get their own private log. To limit it to specific people, add an email allowlist to the rule:

```
allow read, write: if request.auth != null
  && request.auth.uid == uid
  && request.auth.token.email in [
       "you@gmail.com",
       "friend@gmail.com"
     ];
```

(They can still complete Google sign-in, but won't be able to read or write any data.)

---

## Notes

- **Authorized domains:** under **Authentication → Settings → Authorized domains**, add wherever you host it — e.g. `localhost` for testing and `<username>.github.io` for GitHub Pages — or sign-in will be rejected.
- **Cost:** Firestore's free (Spark) tier comfortably covers a personal app with a handful of users — this stays free.
- **You own everyone's data:** other users' logs live in *your* Firebase project, so as the project owner you can see them in the Firestore console and you're responsible for the project. Fine for friends/family; worth being aware of.
- **Document size:** each user's whole dataset is one document (Firestore limit 1 MB — thousands of logs). If anyone ever outgrows it, split their bags into a `users/{uid}/bags` subcollection; only the accessors would change.
- **Offline:** with `enablePersistence`, the app keeps working with no connection and syncs on reconnect, like the localStorage version did.
