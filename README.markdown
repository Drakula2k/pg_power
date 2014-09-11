# PgPower

[![Build Status](https://secure.travis-ci.org/TMXCredit/pg_power.png)](http://travis-ci.org/TMXCredit/pg_power)
[![Dependency Status](https://gemnasium.com/TMXCredit/pg_power.png)](https://gemnasium.com/TMXCredit/pg_power)
[![Code Climate](https://codeclimate.com/badge.png)](https://codeclimate.com/github/TMXCredit/pg_power)

ActiveRecord extension to get more from PostgreSQL:

* Create/drop schemas.
* Set/remove comments on columns and tables.
* Use foreign keys.
* Use partial indexes.
* Run index creation concurrently.

## Environment notes

It was tested with Rails 3.1.x and 3.2.x, Ruby 1.8.7 REE and 1.9.3.

NOTE: JRuby is not yet supported. The current ActiveRecord JDBC
adapter has its own Rails4-compatible method named "create_schema" which
conflicts with this gem.

## Schemas

### Create schema

In migrations you can use `create_schema` and `drop_schema` methods like this:
```ruby
class ReplaceDemographySchemaWithPolitics < ActiveRecord::Migration
  def change
    drop_schema 'demography'
    create_schema 'politics'

    drop_schema_if_exists('demography')
    create_schema_if_not_exists('politics')
  end
end
```
### Create table

Use schema `:schema` option to specify schema name:
```ruby
create_table "countries", :schema => "demography" do |t|
  # columns goes here
end
```
### Move table to another schema

Move table `countries` from `demography` schema to `public`:
```ruby
move_table_to_schema 'demography.countries', :public
```
## Table and column comments

Provides the following methods to manage comments:

* set\_table\_comment(table\_name, comment)
* remove\_table\_comment(table\_name)
* set\_column\_comment(table\_name, column\_name, comment)
* remove\_column\_comment(table\_name, column\_name, comment)
* set\_column\_comments(table\_name, comments)
* remove\_column\_comments(table\_name, *comments)


### Examples

Set a comment on the given table.
```ruby
set_table_comment :phone_numbers, 'This table stores phone numbers that conform to the North American Numbering Plan.'
```
Sets a comment on a given column of a given table.
```ruby
set_column_comment :phone_numbers, :npa, 'Numbering Plan Area Code - Allowed ranges: [2-9] for first digit, [0-9] for second and third digit.'
```
Removes any comment from the given table.
```ruby
remove_table_comment :phone_numbers
```
Removes any comment from the given column of a given table.
```ruby
remove_column_comment :phone_numbers, :npa
```
Set comments on multiple columns in the table.
```ruby
set_column_comments :phone_numbers, :npa => 'Numbering Plan Area Code - Allowed ranges: [2-9] for first digit, [0-9] for second and third digit.',
                                    :nxx => 'Central Office Number'
```
Remove comments from multiple columns in the table.
```ruby
remove_column_comments :phone_numbers, :npa, :nxx
```
PgPower also adds extra methods to change_table.

Set comments:
```ruby
change_table :phone_numbers do |t|
  t.set_table_comment 'This table stores phone numbers that conform to the North American Numbering Plan.'
  t.set_column_comment :npa, 'Numbering Plan Area Code - Allowed ranges: [2-9] for first digit, [0-9] for second and third digit.'
end

change_table :phone_numbers do |t|
  t.set_column_comments :npa => 'Numbering Plan Area Code - Allowed ranges: [2-9] for first digit, [0-9] for second and third digit.',
                        :nxx => 'Central Office Number'
end
```
Remove comments:
```ruby
change_table :phone_numbers do |t|
  t.remove_table_comment
  t.remove_column_comment :npa
end

change_table :phone_numbers do |t|
  t.remove_column_comments :npa, :nxx
end
```
## Foreign keys

We imported some code of [foreigner](https://github.com/matthuhiggins/foreigner)
gem and patched it to be schema-aware. We also added support for index auto-generation.

You should disable `foreigner` in your Gemfile if you want to use `pg_power`.

If you do not want to generate an index, pass the :exclude_index => true option.

The syntax is compatible with `foreigner`:


Add foreign key from `comments` to `posts` using `post_id` column as key by default:
```ruby
add_foreign_key(:comments, :posts)
```
Specify key explicitly:
```ruby
add_foreign_key(:comments, :posts, :column => :blog_post_id)
```
Specify name of foreign key constraint:
```ruby
add_foreign_key(:comments, :posts, :name => "comments_posts_fk")
```
It works with schemas as expected:
```ruby
add_foreign_key('blog.comments', 'blog.posts')
```
Adds the index 'index_comments_on_post_id':
```ruby
add_foreign_key(:comments, :posts)
```
Does not add an index:
```ruby
add_foreign_key(:comments, :posts, :exclude_index => true)
```
## Partial Indexes

We used a Rails 4.x [pull request](https://github.com/rails/rails/pull/4956) as a
starting point, backported to Rails 3.1.x and patched it to be schema-aware.

### Examples

Add a partial index to a table
```ruby
add_index(:comments, [:country_id, :user_id], :where => 'active')
```
Add a partial index to a schema table
```ruby
add_index('blog.comments', :user_id, :where => 'active')
```
## Indexes on Expressions

PostgreSQL supports indexes on expressions. Right now, only basic functional
expressions are supported.

### Examples

Add an index to a column with a function

```ruby
add_index(:comments, "lower(text)")
```

You can also specify index access method

```ruby
create_extension 'btree_gist'
create_extension 'fuzzystrmatch'
add_index(:comments, 'dmetaphone(author)', :using => 'gist')
```

## Concurrent index creation

PostgreSQL supports concurent index creation. We added that feature to migration
DSL on index and foreign keys creation.

### Examples

Add an index concurrently to a table

```ruby
add_index :table, :column_id, :concurrently => true
```

Add an index concurrently along with foreign key

```ruby
add_foreign_key :table1, :table2, :column => :column_id, :concurrent_index => true
```

## Loading/Unloading postgresql extension modules

Postgresql is shipped with a number of [extension modules](http://www.postgresql.org/docs/9.1/static/contrib.html).
PgPower provides some tools
to [load](http://www.postgresql.org/docs/9.1/static/sql-createextension.html)/[unload](http://www.postgresql.org/docs/9.1/static/sql-dropextension.html)
such modules by the means of migrations.

Please note. CREATE/DROP EXTENSION command has been introduced in postgresql 9.1 only. So this functionality will not be
available for the previous versions.

### Examples

Load [fuzzystrmatch](http://www.postgresql.org/docs/9.1/static/fuzzystrmatch.html) extension module
and create its objects in schema *public*:

```ruby
create_extension "fuzzystrmatch"
```


Load version *1.0* of the [btree_gist](http://www.postgresql.org/docs/9.1/static/btree-gist.html) extension module
and create its objects in schema *demography*.

```ruby
create_extension "btree_gist", :schema_name => "demography", :version => "1.0"
```

Unload extension module:

```ruby
drop_extension "fuzzystrmatch"
```

## Views

Version 1.6.0 introduces experimental support for creating views. This API should only be used with the understanding
that it is preliminary 'alpha' at best.

### Example

```ruby
create_view "demography.citizens_view", "select * from demography.citizens"
```

## Roles

If you want to execute migration as specific PostgreSQL role you can use
`set_role` method:

```ruby
class CreateRockBands < ActiveRecord::Migration
  set_role "rocker"

  def change
    create_table :rock_bands do |t|
      # create columns
    end
  end
end
```

Technically it is equal to the following:

```ruby
class CreateRockBands < ActiveRecord::Migration
  def change
    execute "SET ROLE rocker"
    create_table :rock_bands do |t|
      # create columns
    end
  ensure
    execute "RESET ROLE"
  end
end
```

You may force all migration to have `set_role`, for this configre PgPower with
`ensure_role_set=true`:

```ruby
PgPower.configre do |config|
  config.ensure_role_set = true
end
```

## Tools

PgPower::Tools provides number of useful methods:
```ruby
PgPower::Tools.create_schema "services"                 # => create new PG schema "services"
PgPower::Tools.create_schema "nets"                     # => create new PG schema "nets"
PgPower::Tools.drop_schema "services"                   # => remove the PG schema "services"
PgPower::Tools.schemas                                  # => ["public", "information_schema", "nets"]
PgPower::Tools.index_exists?(table, columns, options)   # => returns true if an index exists for the given params
```

## Rails 3

If you are using rails 3.x, use previous pg_power version:

```ruby
gem 'pg_power', '~> 1.6.4'
```

## Running tests:

* Ensure your postgresql has postgres-contrib (Ubuntu) package installed. Tests depend on btree_gist and fuzzystrmatch extensions
 * If on Mac, see below for installing contrib packages
* Configure `spec/dummy/config/database.yml` for development and test environments.
* Run `rake spec`.
* Make sure migrations don't raise exceptions and all specs pass.

### Installing contrib packages on Mac OS X:
* This assumes you are using MacPorts to install Postgres.  If using homebrew or the Postgres App, you will need to adjust the instructions accordingly (please add to this README when you do)
* Assuming you installed with default options (including auto-clean), you will need to rebuild the postgresql port and keep the build files
 * sudo port -k -s build postgresql91
 * (adjust the version number above appropriately)
* Now you can make and install the btree_gist and any other contrib modules
 * cd `port work postgresql91`/postgresql-9.1.7/contrib/btree_gist
 * (again, you may need to adjust the version number to your specific version)
 * sudo make all
 * sudo make install
* Done!

## TODO:

Support for JRuby:

* Jdbc driver provides its own `create_schema(schema, user)` method - solve conflicts.

## Credits

* [Potapov Sergey](https://github.com/greyblake) - schema support
* [Arthur Shagall](https://github.com/albertosaurus) - thanks for [pg_comment](https://github.com/albertosaurus/pg_comment)
* [Matthew Higgins](https://github.com/matthuhiggins) - thanks for [foreigner](https://github.com/matthuhiggins/foreigner), which was used as a base for the foreign key support
* [Artem Ignatyev](https://github.com/cryo28) - extension modules load/unload support
* [Marcelo Silveira](https://github.com/mhfs) - thanks for rails partial index support that was backported into this gem

## Copyright and License

* Copyright (c) 2012 TMX Credit.
* Initial foreign key code taken from foreigner, Copyright (c) 2009 Matthew Higgins
* pg_comment Copyright (c) 2011 Arthur Shagall
* Partial index Copyright (c) 2012 Marcelo Silveira

Released under the MIT License.  See the MIT-LICENSE file for more details.

## Contributing

Contributions are welcome.  However, before issuing a pull request, please make sure of the following:

* All specs are passing (under both ree and 1.9.3)
* Any new features have test coverage.
* Anything that breaks backward compatibility has a very good reason for doing so.
