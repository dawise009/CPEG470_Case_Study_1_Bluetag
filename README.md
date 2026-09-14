# BlueTag

Campus lost-and-found board. Students post what they lost or found, search the board, and mark a listing resolved when the item goes home.

Built as a small Express app with EJS pages and SQLite so it can run on a laptop or a single VM.

## Features

- Register with a `.edu` email, sign in, sign out
- Public board with search, kind, and category filters
- Create a lost or found post
- View a post and contact the poster
- Owners can mark their own posts resolved

## Run locally

You need Node 22.13+ (the app uses the built-in `node:sqlite` module).

```bash
cp .env.example .env
npm install
npm start
```

Open [http://localhost:3000](http://localhost:3000). The database file is created and seeded on first boot at `data/bluetag.db`.

```bash
npm run dev    # restarts on file changes
npm run seed   # no-op if users already exist
```

## Run with Docker

```bash
docker compose up --build
```

The app listens on port 3000. Item data lives in the `bluetag-data` volume.

## Demo accounts

| Email | Password | Notes |
|---|---|---|
| `alex@campus.edu` | `campus123` | Has a couple of lost posts |
| `jordan@campus.edu` | `campus123` | Has found posts |
| `sam@campus.edu` | `foundit!` | Mixed posts |

Register your own account if you want; it just has to end in `.edu`.

## Project layout

```
src/index.js           HTTP server, sessions, static files
src/db.js              SQLite schema
src/seed.js            First-run demo data
src/routes/auth.js     Register / login / logout
src/routes/items.js    Board, search, posts
src/views/             EJS pages
src/public/css/        Styles
```

## Assignment notes

Host this somewhere your classmates and instructor can reach. Walk the running app until you can explain:

- how a request becomes a row in SQLite
- where authentication is enforced
- how search, filters, and item pages load data

Then look for a security defect in the running system, document how to trigger it, and patch it without breaking normal use of the board. Submit the hosted URL, a short architecture sketch, the writeup, and the patched repo.


---

# Security Writeup

# BlueTag Security Writeup

## Hosted URL

https://cpeg470-case-study-1-bluetag.onrender.com

## Architecture

BlueTag is an Express application using EJS for HTML pages and SQLite for data.

- `src/index.js` configures Express, sessions, static files, routes, and error handling.
- `src/routes/auth.js` handles registration, login, and logout.
- `src/routes/items.js` handles the board, search, filters, item creation, item pages, and resolving posts.
- `src/db.js` creates the SQLite tables.
- `users` stores accounts; `items` stores listings and links each listing to its owner through `items.user_id`.

### Request flow

```text
Browser → Express route → SQLite query → EJS template → Browser
```

For a new item, a signed-in user submits `POST /items`. The app takes the user ID from `req.session.user.id`, inserts the item into SQLite, and redirects to the new item page.

Authentication is enforced with `requireAuth`. The resolve route also verifies that `items.user_id` matches the signed-in user before changing a post.

## Vulnerability: SQL Injection

### Affected code

`src/routes/items.js`, function `searchItems()`.

### Problem

The public board search and filters placed user input directly into SQL strings:

```js
sql += ` AND items.category = '${category}'`;
```

The same issue existed for `q` and `kind`. An attacker could place SQL syntax in a URL parameter and alter the database query.

### Reproduction

1. Start BlueTag locally.
2. Open:

   ```text
   http://localhost:3000/?q='
   ```

3. Before the fix, BlueTag displayed:

   ```text
   Search could not run. Try a simpler phrase.
   ```

The quote broke the SQL query because it was inserted directly into the `LIKE` clause.

### Impact

The search endpoint is public, so no account is required. An attacker could cause query errors, bypass expected filtering, or potentially access database data through SQL injection.

## Fix

I changed the query to use SQLite placeholders and bound parameters:

```js
const params = [];

if (q) {
  sql += `
    AND (
      items.title || ' ' || items.description || ' ' || items.location
    ) LIKE ?
  `;
  params.push(`%${q}%`);
}

if (category && category !== "all") {
  sql += " AND items.category = ?";
  params.push(category);
}

if (kind && kind !== "all") {
  sql += " AND items.kind = ?";
  params.push(kind);
}

return db.prepare(sql).all(...params);
```

I also allowlisted valid `category` and `kind` values before running the query.

## Validation

After the patch, I verified:

- Keyword search works.
- Category and lost/found filters work.
- Combined search and filters work.
- `/?q='` no longer causes a query error.
- Invalid filter values are safely treated as `all`.
- Login, posting, item pages, and resolving the owner’s own post still work.
- Users cannot resolve posts they do not own.
