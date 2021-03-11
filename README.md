## Let's create a Todo app

Install Rails

> gem install rails

Create a new rails app
> rails new todo --database=postgresql

Bundle and voilá!

Let's make a bunch of databases. Update your database.yml like so (much of what comes being taken from the commit's docs):

```ruby
development:
  primary:
    <<: *default
    database: my_primary_database
  primary_replica:
    database: my_primary_database
    replica: true
  primary_shard_one:
    <<: *default
    database: my_primary_shard_one
  primary_shard_one_replica:
    <<: *default
    database: my_primary_shard_one
    replica: true
```
You can use whatever database or adapter you want. I normally hook up Postgres, today playing around with SQLite3. 

In this setup from the docs, we have a primary and primary_shard, and each has a replica. Nifty!

Then run:
```ruby
rails db:create; rails db:migrate
# Created database 'my_primary_database'
# Created database 'my_primary_shard_one'
# Time to fill the database
# Let's generate a model:
rails g model Post title:string
rails db:migrate
```

```
#=>
== 20210311143955 CreatePosts: migrating ======================================
-- create_table(:posts)
   -> 0.0127s
== 20210311143955 CreatePosts: migrated (0.0128s) =============================

== 20210311143955 CreatePosts: migrating ======================================
-- create_table(:posts)
   -> 0.0060s
== 20210311143955 CreatePosts: migrated (0.0060s) =============================
```

And either in a seeds file or our console, run:
```ruby
Post.create!(title: 'Foo')

Post.create!(title: 'Bar')

Post.create!(title: 'Fizz')
```

Playing with the databases:

Now, let's update our application_record.rb like so:
```ruby
class ApplicationRecord < ActiveRecord::Base
  self.abstract_class = true

  connects_to shards: {
    default: { writing: :primary, reading: :primary_replica },
    shard_one: { writing: :primary_shard_one, reading: :primary_shard_one_replica }
  }
end
```
We will now reload our console and see what we can do, using: `ActiveRecord::Base.connected_to` 

```ruby
ActiveRecord::Base.connected_to(role: :reading, shard: :shard_one) do
  Post.first
end
```

> #=> nil


### Now let's write to :shard_one
```ruby
ActiveRecord::Base.connected_to(role: :writing, shard: :shard_one) do
   Post.create!(
     title: 'Buzz'
     )
    Post.create!(
      title: 'Woof'
    )
 end
```
> It works!

### And we'll read from :shard_one's replica:
```ruby
ActiveRecord::Base.connected_to(role: :reading, shard: :shard_one) do
  Post.count
end
```
> #=> 2

### And finally confirm that shard one's data has not been written to primary:

```ruby
ActiveRecord::Base.connected_to(role: :reading, shard: :default) do
  Post.count
end
```
> #=> 3

This is correct, remember the three posts we created earlier who default to the primary database.


That's it!