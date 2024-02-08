---%%%---
title = "Converting Phoenix user ids from :integer to :binary_id"
description = """\
  How to convert Phoenix ids from :integer to :binary_id \
  (if you forgot to do this when you used phx.gen.auth)
"""
published_date = "2024-02-06"
template = "post"
tags = [ "elixir", "programming", "phoenix"]
draft = true
---%%%---

_Note: This post uses Phoenix 1.7.10, Elixir 1.15.7, and OTP 26.2.1 as well as Postgres 16.0_

## I've made a mistake

I've been working on a bookmarking app (working name of Bookmark, because I'm a thought-leader)
using Elixir and Phoenix.
Like many apps, I'm supporting multiple users
(even though I'll probably be the only one using it (it's custom-made for my needs, so don't feel bad for me)).
So, I need (at minimum) pages for user sign in, user registration, forgotten passwords, account confirmation,
password resets, sign out, and settings.

This sounds like a lot of work and I'm coming from Django, where auth is pretty built out, so
`phx.gen.auth` was welcome in Phoenix. `phx.gen.auth` is a generator that creates
a "flexible, pre-built authentication system" for your app. It creates controllers, schemas, contexts,
and views (either LiveViews or dead views) for your app's auth.

This is great, but comes with some options that aren't covered in the
[hexdocs guide](https://hexdocs.pm/phoenix/1.7.10/mix_phx_gen_auth.html)
(but to be fair, are covered in the separate [Mix.Tasks.Phx.Gen.Auth docs](https://hexdocs.pm/phoenix/Mix.Tasks.Phx.Gen.Auth.html)
-- maybe I should read the docs in full before making large application decisions 🤷).

The option I missed was `--binary-id`. This uses `:binary_id` (an `Ecto.UUID`)
for the primary key of the `:users` table
(and the `:users_tokens` table, but I haven't converted that yet 🙃)
in the migration that `phx.gen.auth` generates.

Is this a huge mistake? No -- but I did use the `--binary-id` flag for the 
other tables that I generated (using `phx.gen.html`! How handy!),
and I'd like to keep this constant throughout the app.
There are other reasons to use UUIDs as identifiers, but I'll leave you to
search this on your own time.

## What I've got to do about it

So, at this point in the app's life, I have a few tables other than my `:users`
table. What they're named and what they do aren't super relevant (though I am excited to
dive into the application in other posts), since the plan is the same for each foreign key (FK) to `:users`:

1. Create a new column of type `:binary_id` that will be the new UUID primary key (PK) for `:users`.
2. Create columns for each current FK to `:users` but using `:binary_id` as well.
If my `:links` table has a `:user_id` column, I'll need a `:user_uuid` column -- and so on for other FKs.
3. Generate UUIDs using `Ecto.UUID`
4. Map the existing `id` -> `<FK>_id` values to `uuid` -> `<FK>_uuid`.
5. Drop the existing foreign key constraints
6. Drop the existing `id` and `<FK>_id` columns
7. Rename the `uuid` and `<FK>_uuid` columns to `id` and `<FK>_id`
8. Add new foreign key constraints
9. Modify the Ecto schema to mark the new primary key of `:users` as a `:binary_id`


## More in-depth example with code!

As an example, I'll use my `:registration_tokens` table that stores tokens used in the
account registration flow. The `:registration_tokens` table has two (count 'em two!) foreign keys to `:users`.

To begin, the `:users` table looks like this:

```
Table "public.users"
 
     Column      |              Type              | Collation | Nullable | Default
-----------------+--------------------------------+-----------+----------+---------
 id              | bigint                         |           | not null | nextval('users_id_seq'::regclass)
 email           | citext                         |           | not null |
 hashed_password | character varying(255)         |           | not null |
 inserted_at     | timestamp(0) without time zone |           | not null |
 updated_at      | timestamp(0) without time zone |           | not null |
Indexes:
    "users_pkey" PRIMARY KEY, btree (id)
    "users_email_index" UNIQUE, btree (email)
```

and the `:registration_tokens` table looks like this (minus some irrelevant columns):

```
Table "public.registration_tokens"

        Column        |              Type              | Collation | Nullable |                     Default
----------------------+--------------------------------+-----------+----------+-------------------------------------------------
 id                   | bigint                         |           | not null | nextval('registration_tokens_id_seq'::regclass)
 inserted_at          | timestamp(0) without time zone |           | not null |
 updated_at           | timestamp(0) without time zone |           | not null |
 generated_by_user_id | bigint                         |           |          |
 used_by_user_id      | bigint                         |           |          |
Indexes:
    "registration_tokens_pkey" PRIMARY KEY, btree (id)
Foreign-key constraints:
    "registration_tokens_generated_by_user_id_fkey" FOREIGN KEY (generated_by_user_id) REFERENCES users(id)
    "registration_tokens_used_by_user_id_fkey" FOREIGN KEY (used_by_user_id) REFERENCES users(id)
```

I'll need two migrations for this, run separately.

In the first migration, I need to create the new UUID column to be the new primary key on the `:users` table.

```
alter table(:users) do
  add :uuid, :binary_id
end
```

I'll also create UUID columns for the foreign key to the `:users` table on the `:registration_tokens` table.

```
alter table(:registration_tokens) do
  add :generated_by_user_uuid, :binary_id
  add :used_by_user_uuid, :binary_id
end

flush()  # this makes sure the columns are available further into the migration
```

Then, I'll create the new UUIDs for each row in `:users` using `Ecto.UUID`.

```
users = from(u in User, select: u) |> Repo.all()

Enum.each(users, fn user ->
  uuid = Ecto.UUID.bingenerate()

  from(u in "users", where: u.id == ^user.id, update: [set: [uuid: ^uuid]])
  |> Repo.update_all([])
end)
```

And map the existing `:users.id` relations to the new `uuid` field

```
from(rt in "registration_tokens",
  join: u in "users",
  on: rt.generated_by_user_id == u.id,
  update: [set: [generated_by_user_uuid: u.uuid]]
)
|> Repo.update_all([])

from(rt in "registration_tokens",
  join: u in "users",
  on: rt.used_by_user_id == u.id,
  update: [set: [used_by_user_uuid: u.uuid]]
)
|> Repo.update_all([])
```

And that's it for the first migration! Note that we haven't done anything destructive yet,
and can remove the `:binary_id` columns without losing any data.

The second migration will be the destructive one, so consider whether you trust me and maybe backup your DB.

Here, we'll drop our FK constraints so we can later drop the columns that they reference:

```
drop(constraint(:registration_tokens, "registration_tokens_generated_by_user_id_fkey"))
drop(constraint(:registration_tokens, "registration_tokens_used_by_user_id_fkey"))
```


