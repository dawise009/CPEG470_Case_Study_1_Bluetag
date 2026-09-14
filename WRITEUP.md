# BlueTag Security Writeup

## Hosted URL

[Add hosted URL after deployment]

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
