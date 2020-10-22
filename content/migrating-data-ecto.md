+++
title = "Migrating legacy data in Elixir using Ecto"
date = 2020-10-22
[taxonomies]
tags = ["elixir", "ecto", "postgres"]
categories = ["web_dev"]
+++

Recently I needed to migrate some data from one postgres server to another postgres server. This 
project in particular happens to be a [Phoenix](https://www.phoenixframework.org/) application, and
coming from other languages and frameworks I was dreading migrating this data because usually this 
type of thing is painful. Like any programmer, I cannot be bothered to *actually* write SQL in 2020.
So I set out to write a plain old elixir script, and I was pleasantly surprised at how simple it was
to do in elixir using [ecto](https://hexdocs.pm/ecto/Ecto.html).

First goal was to simply make a connection to the old database in a script file,
`priv/repo/migrate-legacy.exs`. I Started by defining an `Ecto.Repo` to bring in all the ecto 
goodies. I mark the Repo `read_only` so I don't mess with any production data on accident, and I 
load the connection string from the environment.

```elixir
legacy_db_url = System.get_env("LEGACY_DB_URL") || raise("Missing LEGACY_DB_URL")

defmodule LegacyRepo do
  use Ecto.Repo,
    otp_app: :my_app,
    adapter: Ecto.Adapters.Postgres,
    # For safety
    read_only: true
end
```

Now that I have a `LegacyRepo` setup, I can attempt to make the connection and run a simple
query from a table I know exists in the old database, `users`.

```elixir
LegacyRepo.start_link(url: legacy_db_url, ssl: true)

IO.inspect(email: LegacyRepo.all(from "users", select: [:email], limit: 1))
```

If all goes well you should see a result in the console with the data returned from the query. In my
case, my email printed to the console.

```text
[email: [%{email: "nicetryspammer@example.com"}]]
```

With all that working I already felt dangerous, but I wanted to be able to use Ecto schemas. In the
initial test query I used what's known as a
[schemaless query](https://hexdocs.pm/ecto/schemaless-queries.html) in ecto. I specified the table
as a binary, `from "users"`, and while this works fine, I wanted to take advantage of changesets.

Turns out it's dead simple to do, just `use Ecto.Schema` the same way you do in Phoenix. In this
case I model the schema after the existing table in my legacy database.

```elixir
defmodule User do
  use Ecto.Schema

  schema "users" do
    field :email, :string
    field :given_name, :string
    field :family_name, :string
  end
end
```

And now I can use this in my `LegacyRepo` functions.

```elixir
# Using Ecto expressions
IO.inspect(User |> limit(1) |> LegacyRepo.all())
```

Now for the good stuff, since this script is in my project, I can just bring in the repo and new
schema in my app. In this case I was mapping the old "users" table to the new "employees" table.
And yes, I can even bring in my context module with all my helper functions.

```elixir
alias MyApp.Repo
alias MyApp.Accounts
alias MyApp.Accounts.Employee

defmodule LegacyMigrator do
  def migrate_user(%User{} = user) do
    attrs = %{
      email: user.email,
      given_name: user.given_name,
      family_name: user.family_name
    }

    changeset = Accounts.change_employee(%Employee{}, attrs)

    case Accounts.create_employee(changeset) do
      {:ok, employee} ->
        Logger.info("Migrated user: #{employee.email}")

      {:error, cs} ->
        Logger.error("Failed to migrate user: #{cs.changes.email}")
    end
  end
end
```

I created a `LegacyMigrator` to house my `migrate_user` function and perform the logic to map the
old fields together. I could have passed the whole `User` struct to the changeset but it has other
fields I don't want casted, like `id`, so I decided to be deliberate about the fields I want.

With that I can go ahead and migrate all the users if they don't exist in the new system.

```elixir
Enum.each(LegacyRepo.all(User), fn u ->
  if Repo.exists?(from e in Employee, where: e.email == ^u.email) do
    Logger.info("User #{u.email} already exits, skipping...")
  else
    LegacyMigrator.migrate_user(u)
  end
end)
```

That's it! I was incredibly impressed with how easy it was to wire up a datasource and begin running
queries, all in a simple script. For those interested the final code is below, and feel free to ask
me any questions on [twitter, @Khampton_03](https://twitter.com/Khampton_03).

```elixir
legacy_db_url = System.get_env("LEGACY_DB_URL") || raise("Missing LEGACY_DB_URL")

defmodule LegacyRepo do
  use Ecto.Repo,
    otp_app: :my_app,
    adapter: Ecto.Adapters.Postgres,
    # For safety
    read_only: true
end

LegacyRepo.start_link(url: legacy_db_url, ssl: true)

defmodule User do
  use Ecto.Schema

  schema "users" do
    field :email, :string
    field :given_name, :string
    field :family_name, :string
  end
end

alias MyApp.Repo
alias MyApp.Accounts
alias MyApp.Accounts.Employee

defmodule LegacyMigrator do
  def migrate_user(%User{} = user) do
    attrs = %{
      email: user.email,
      given_name: user.given_name,
      family_name: user.family_name
    }

    changeset = Accounts.change_employee(%Employee{}, attrs)

    case Accounts.create_employee(changeset) do
      {:ok, employee} ->
        Logger.info("Migrated user: #{employee.email}")

      {:error, cs} ->
        Logger.error("Failed to migrate user: #{cs.changes.email}")
    end
  end
end

Enum.each(LegacyRepo.all(User), fn u ->
  if Repo.exists?(from e in Employee, where: e.email == ^u.email) do
    Logger.info("User #{u.email} already exits, skipping...")
  else
    LegacyMigrator.migrate_user(u)
  end
end)
```
