# Active Record Migrations

## Learning Goals

- Create, connect to, and manipulate a SQLite database using Active Record

## Setup

We're going to use the `activerecord` gem to create a mapping between our
database and model. Clone down this lesson and code-along to get to the
solution.

## Migrations

From [the _RailsGuides_ section on Migrations][guide-migrations]:

> Migrations are a convenient way for you to alter your database in a structured
> and organized manner. You could edit fragments of SQL by hand but you would
> then be responsible for telling other developers that they need to go and run
> them. You’d also have to keep track of which changes need to be run against
> the production machines next time you deploy.

> Migrations also allow you to describe these transformations using Ruby. The
> great thing about this is that (like most of Active Record’s functionality)
> it is database-independent: you don’t need to worry about the precise syntax
> of `CREATE TABLE` any more than you worry about variations on `SELECT *` (you
> can drop down to raw SQL for database-specific features). For example, you
> could use SQLite3 in development, but MySQL in production.

Another way to think of migrations is like version control for your database.
You might create a table, add some data to it, and then make some changes to it
later on. By adding a new migration for each change you make to the database,
you won't lose any data you don't want to, and you can easily revert changes.

Executed migrations are tracked by ActiveRecord in your database so that they
aren't used twice. Using the migrations system to apply the schema changes is
easier than keeping track of the changes manually and executing them manually
at the appropriate time.

### Setting Up Your Migration

1. Create a directory called `db` at the top level of the lesson's directory.
   Then, within the `db` directory, create a `migrate` directory. The
   `mkdir` command is the appropriate tool to use here.
2. In the `db/migrate ` directory, create a file called `01_create_artists.rb`
   (we'll talk about why we added the `01` later).

```text
mechanics-of-migrations-v-000/
  config/
    environment.rb
  db/
    migrate/
      01_create_artists.rb
  spec/
    artist_spec.rb
    spec_helper.rb
  .gitignore
  .learn
  .rspec
  artist.rb
  CONTRIBUTING.md
  Gemfile
  Gemfile.lock
  LICENSE.md
  Rakefile
  README.md
```

With the file created, we'll need to add in the migration code:

```ruby
# db/migrate/01_create_artists.rb

class CreateArtists < ActiveRecord::Migration[5.2]
  def up
  end

  def down
  end
end
```

#### Active Record 5.x Migration Syntax Update

**IMPORTANT**: Active Record is primarily used in Rails applications and as of
Active Record 5.x, we must specify which version of Rails the migration
was written for, even in situations like this lab where we do not have Rails
configured.

This lesson was originally created with gem versions that support Rails **4.2**,
so we need to make have our `CreateArtist` migration inherit from
`ActiveRecord::Migration[4.2]`.

Don't worry too much about this until you get to the Rails section. Until then,
if you encounter an error like this...

```text
Caused by:
StandardError: Directly inheriting from ActiveRecord::Migration is not supported. Please specify the Rails release the migration was written for:

  class CreateArtists < ActiveRecord::Migration[4.2]
```

...simply add `[4.2]` or whatever number is displayed to the end of
`ActiveRecord::Migration` in your migration file, exactly as the error message
instructs.

### Active Record Migration Methods: up, down, change

Here we're creating a class called `CreateArtists` that inherits from
ActiveRecord's `ActiveRecord::Migration` module. Within the class, we have an
`up` method to define the code to execute when the migration is run and a
`down` method to define the code to execute when the migration is rolled back.
Think of it like "do" and "undo."

Another method is available to use besides `up` and `down`: `change`, which is
more common for basic migrations.

```ruby
# db/migrate/01_create_artists.rb

class CreateArtists < ActiveRecord::Migration[4.2]
  def change
  end
end

```

From [the Active Record Migrations RailsGuide][change-method]:

> The change method is the primary way of writing migrations. It works for the
> majority of cases, where Active Record knows how to reverse the migration
> automatically

Let's take a look at how to finish off our `CreateArtists` migration, which
will generate our `artists` table with the appropriate columns.

### Creating a Table

Remember how we created a table using SQL with ActiveRecord?

> **NOTE**: Recall, we can do this with IRB: `irb -r active_record`

First, we connect to a database, then write the necessary SQL to create the
table. So, first, we'd have to connect to a database:

```ruby
ActiveRecord::Base.establish_connection(
  :adapter => "sqlite3",
  :database => "db/artists.sqlite"
)
```

Then write some SQL to create the table:

```ruby
sql = <<-SQL
  CREATE TABLE IF NOT EXISTS artists (
  id INTEGER PRIMARY KEY,
  name TEXT,
  genre TEXT,
  age INTEGER,
  hometown TEXT
  )
SQL

ActiveRecord::Base.connection.execute(sql)
```

Using migrations, we will still need establish Active Record's connection to the
database, but **_we no longer need the SQL!_** Instead of dealing with SQL
directly, we provide the migrations we want and Active Record takes care of creating 

Since we still need to connect to the database, let's make the connection
inside `config/environment.rb`:

```ruby
# config/environment.rb
require 'rake'
require 'active_record'
require 'yaml/store'
require 'ostruct'
require 'date'

require 'bundler/setup'
Bundler.require

# put the code to connect to the database here
ActiveRecord::Base.establish_connection(
  :adapter => "sqlite3",
  :database => "db/artists.sqlite"
)

require_relative "../artist.rb"
```

> **Reminder**: The `environment.rb` file commonly contains things we want to
> read and make available to our Ruby application when it starts. It isn't
> necessary that you totally grasp all the parts of this file, but looking through
> it with this in mind, you might be able to gather what is happening: some gems,
> including `active_record` are required; something happens with `Bundler`; our
> database connection is established; the `artist.rb` file is read.

With the connection to the database configured, we should have access to
`ActiveRecord::Migration`, and can create tables using only Ruby!

```ruby
# db/migrate/01_create_artists.rb
def change
  create_table :artists do |t|
  end
end
```

Here we've added the `create_table` method and passed the name of the table we
want to create as a symbol. Pretty simple, right? Other methods we can use here
are things like `remove_table`, `rename_table`, `remove_column`, `add_column`
and others. See [this list][writing-mig] for more.

No point in having a table that has no columns in it, so let us add a few:

```ruby
# db/migrate/01_create_artists.rb

class CreateArtists < ActiveRecord::Migration[4.2]
  def change
    create_table :artists do |t|
      t.string :name
      t.string :genre
      t.integer :age
      t.string :hometown
    end
  end
end
```

Looks a little familiar? On the left we've given the data type we'd like to
cast the column as, and on the right, we've given the name we'd like to give the
column. The only thing that we're missing is the primary key. Active Record
will generate that column for us, and for each row added, a key will be
auto-incremented.

And that's it! You've created your first Active Record migration. Next, we're
going to see it in action!

### Running Migrations

The simplest way is to run our migrations through a Rake task that we're given
through the `activerecord` gem. How do we access these?

Run `rake -T` to see the list of commands we have.

> **Note**: If you get an error when trying to run `rake` commands, you may have
> a newer version of rake already installed compared to this lesson, causing a
> conflict. To avoid this error, run `bundle exec rake -T`. Adding `bundle exec`
> indicates that you want `rake` to run within the context of this lesson's
> bundle (defined in the `Gemfile`), not the default version of `rake` you have
> installed globally on your computer.

Let's look at the `Rakefile`. The commands listed when running `rake -T` are
made available as Rake tasks through `require 'sinatra/activerecord/rake'`.

Now take a look again at `environment.rb`, which our `Rakefile` also requires:

```ruby
# config/environment.rb

require 'bundler/setup'
Bundler.require

ActiveRecord::Base.establish_connection(
  :adapter => "sqlite3",
  :database => "db/artists.sqlite"
)
```

This file is requiring the gems in our Gemfile and giving our program access to
them. We're using the `establish_connection` method from `ActiveRecord::Base`
to connect to our `artists` database, which will be created in the migration
via SQLite3 (the adapter).

After we've added the above code to `config/environment.rb`, it's time to run
`rake db:migrate`.

**Note**: Here again, if you encounter an error after running `rake db:migrate`,
try running `bundle exec rake db:migrate`.

Take a look at `artist.rb`. Let's create an Artist class.

```ruby
# artist.rb

class Artist
end
```

Next, we'll extend the class with `ActiveRecord::Base`

```ruby
# artist.rb

class Artist < ActiveRecord::Base
end
```

To test our newly-created class out, let's use the rake task `rake console` (or
`bundle exec rake console`), which we've created in the `Rakefile`.

### Try Out The Following:

Check that the class exists:

```ruby
Artist
# => Artist (call 'Artist.connection' to establish a connection)
```

View the columns in its corresponding table in the database:

```ruby
Artist.column_names
# => ["id", "name", "genre", "age", "hometown"]
```

Instantiate a new Artist named Jon, set his age to 30, and save him to the database:

```ruby
a = Artist.new(name: 'Jon')
# => #<Artist id: nil, name: "Jon", genre: nil, age: nil, hometown: nil>

a.age = 30
# => 30

a.save
# => true
```

The `.new` method creates a new instance in memory, but for that instance to
persist, we need to save it. If we want to create a new instance and save it all
in one go, we can use `.create`.

```ruby
Artist.create(name: 'Kelly')
# => #<Artist id: 2, name: "Kelly", genre: nil, age: nil, hometown: nil>
```

Return an array of all Artists from the database:

```ruby
Artist.all
# => [#<Artist id: 1, name: "Jon", genre: nil, age: 30, hometown: nil>,
 #<Artist id: 2, name: "Kelly", genre: nil, age: nil, hometown: nil>]
```

Find an Artist by name:

```ruby
Artist.find_by(name: 'Jon')
# => #<Artist id: 1, name: "Jon", genre: nil, age: 30, hometown: nil>
```

There are several methods you can now use to create, retrieve, update, and
delete data from your database, and a whole lot more.

Take a look at these [CRUD methods][crud], and play around with them.

## Using Migrations To Manipulate Existing Tables

Here is another place where migrations really shine. Let's add a
`favorite_food` column to our `artists` table. Remember that Active Record
keeps track of the migrations we've already run, so adding the new code to our
`01_create_artists.rb` file won't work. Since we aren't rolling back our
previous migration (or dropping the entire table), the `01_create_artists.rb`
migration won't be re-executed when we run `rake db:migrate` again. Generally,
the best practice for database management (especially in a production
environment) is creating new migrations to modify existing tables. That way,
we'll have a clear, linear record of all of the changes that have led to our
current database structure.

To make this change we're going to need a new migration, which we'll call
`02_add_favorite_food_to_artists.rb`.

```ruby
# db/migrate/02_add_favorite_food_to_artists.rb

class AddFavoriteFoodToArtists < ActiveRecord::Migration[4.2]
  def change
    add_column :artists, :favorite_food, :string
  end
end
```

Pretty awesome, right? We just told Active Record to add a column to the
`artists` table called `favorite_food` and that it will contain a string.

Notice how we incremented the number in the file name there? Imagine for a
minute that you deleted your original database and wanted to execute the
migrations again. Active Record is going to execute each file, but it does so
in alpha-numerical order. If we didn't have the numbers, our `add_column`
migration would have tried to run first (`[a]dd_favorite...` comes before
`[c]reate_artists...`), and our `artists` table wouldn't have even been created
yet! So we used some numbers to make sure the migrations execute in order. In
reality, our two-digit system is very rudimentary. As you'll see later on,
frameworks like Rails have generators that create migrations with very accurate
timestamps, so you'll never have to worry about hand-numbering.

Now that you've saved the migration, go back to the terminal to run 
`rake db:migrate`.

Awesome! Now go back to the console with the `rake console` command, and check
it out:

```ruby
Artist.column_names
# => ["id", "name", "genre", "age", "hometown", "favorite_food"]
```

Great!

Nope- wait. Word just came down from the boss- you weren't supposed to ship
that change yet! OH NO! No worries, we'll roll back to the first migration.

Run `rake -T`. Which command should we use?

```bash
rake db:rollback
```

Then drop back into the console to double check:

```ruby
Artist.column_names
# => ["id", "name", "genre", "age", "hometown"]
```

Oh good, your job is saved. Thanks, Active Record! When the boss says it's
actually time to add that column, you can just run `rake db:migrate` again!

Woohoo!

[guide-migrations]: http://guides.rubyonrails.org/v3.2.8/migrations.html
[repo]: https://github.com/learn-co-students/mechanics-of-migrations-v-000
[change-method]: http://edgeguides.rubyonrails.org/active_record_migrations.html#using-the-change-method
[writing-mig]: http://guides.rubyonrails.org/migrations.html#writing-a-migration
[crud]: http://guides.rubyonrails.org/active_record_basics.html#crud-reading-and-writing-data
