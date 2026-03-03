# Notes App Database Schema (PostgreSQL)

This container runs PostgreSQL and exposes a connection string in `db_connection.txt`:

```bash
cat db_connection.txt
# psql postgresql://appuser:dbuser123@localhost:5000/myapp
```

All schema objects below are created in the `public` schema.

## Extensions

- `citext`  
  Used for case-insensitive `email` and `tag name` uniqueness.
- `pgcrypto`  
  Used for `gen_random_uuid()` UUID primary keys.
- `pg_trgm`  
  Used for trigram search indexes on note `title`/`content`.

## Tables

### `users`

Stores user accounts for email/password authentication.

Columns:
- `id uuid primary key default gen_random_uuid()`
- `email citext not null`
- `password_hash text not null`
- `created_at timestamptz not null default now()`
- `updated_at timestamptz not null default now()`

Constraints:
- Unique email:
  - `users_email_uq` on `(email)`
- Basic email format check:
  - `users_email_format_chk` ensures `position('@' in email) > 1`

Triggers:
- `users_set_updated_at` sets `updated_at = now()` on every update.

---

### `notes`

Stores notes owned by a user.

Columns:
- `id uuid primary key default gen_random_uuid()`
- `user_id uuid not null references users(id) on delete cascade`
- `title text not null default ''`
- `content text not null default ''`
- `created_at timestamptz not null default now()`
- `updated_at timestamptz not null default now()`

Constraints:
- Ownership / access control is enforced structurally by `user_id` FK.

Indexes:
- `notes_user_id_idx` on `(user_id)`
- `notes_user_updated_at_idx` on `(user_id, updated_at desc)`
- `notes_title_trgm_idx` GIN(trgm) on `title` (for fast `LIKE/ILIKE` search)
- `notes_content_trgm_idx` GIN(trgm) on `content` (for fast `LIKE/ILIKE` search)

Triggers:
- `notes_set_updated_at` sets `updated_at = now()` on every update.

---

### `tags`

Stores per-user tags (each user has an independent tag namespace).

Columns:
- `id uuid primary key default gen_random_uuid()`
- `user_id uuid not null references users(id) on delete cascade`
- `name citext not null`
- `created_at timestamptz not null default now()`

Constraints:
- Non-blank tag name:
  - `tags_name_not_blank_chk` ensures `length(trim(name::text)) > 0`
- Unique tag name per user:
  - `tags_user_name_uq` on `(user_id, name)`

Indexes:
- `tags_user_id_idx` on `(user_id)`

---

### `note_tags`

Join table between notes and tags.

Columns:
- `note_id uuid not null references notes(id) on delete cascade`
- `tag_id uuid not null references tags(id) on delete cascade`
- `created_at timestamptz not null default now()`

Constraints:
- Composite primary key (prevents duplicates):
  - `primary key (note_id, tag_id)`

Indexes:
- `note_tags_tag_id_idx` on `(tag_id)` (supports filtering notes by a tag)

## Shared trigger function

A shared function is used by `users` and `notes`:

- `set_updated_at()` (plpgsql): sets `NEW.updated_at = now()`.

## Notes on alignment with the app

- UUID primary keys work well with API-driven clients and avoid leaking sequential IDs.
- `citext` makes email and tag-name uniqueness case-insensitive (e.g., `Alice@x.com` equals `alice@x.com`).
- `pg_trgm` indexes support efficient substring search on notes for the app’s search UX.
- `ON DELETE CASCADE` ensures that deleting a user removes their notes/tags, and deleting a note or tag cleans up join rows.
