# <img src="https://cloud.githubusercontent.com/assets/7833470/10899314/63829980-8188-11e5-8cdd-4ded5bcb6e36.png" height="60"> ActiveRecord Associations

| Objectives |
| :--- |
| Create one-to-many and many-to-many relationships in Rails |
| Modify migrations to add foreign keys to tables |
| Create a join table for a many-to-many relationship |
| Create model instances with associations |

## Associations: Relationships Between Models

| Relationship Type | Abbreviation | Description | Example |
| :--- | :--- | :--- | :--- |
| One-to-Many | 1:N | Parent model is associated with many children from another model | One author can have many books. |
| Many-to-Many | N:N | Two models that can both be associated with many of the other. | Libraries and books. One library can have many books, while one book can be in many libraries. |



# SQL `JOIN`S REVISITED

## Joins

Each table in a relational database is considered a relation. All of the table's data is naturally related by single set of attributes defined for it. However, in order to be relational, we need to be able to make queries between relations or tables of data.

**JOIN**s are our means of implementing queries that combine data and show results from multiple tables.

There are many kinds of joins, based on how you want to combine data.

![](https://raw.githubusercontent.com/sf-wdi-18/notes/master/lectures/week-07/day-1-intro-sql/dawn-simple-queries/images/join.png)

## Foreign Key

To implement a `JOIN` between two tables, one of our tables must have a **foreign key**. A foreign key is a field in one table that uniquely identifies a row of another table. We use the foreign key to **establish and enforce a link between the data in two tables**.

The foreign key always goes on the table with the data that belongs to data from another table. In the example below, a person **has_many** pets, and a pet **belongs_to** a person. The foreign key `person_id` goes on the `pets` table to indicate which person the pet belongs to.

![](https://raw.githubusercontent.com/sf-wdi-18/notes/master/lectures/week-07/day-1-intro-sql/dawn-simple-queries/images/primary_foreign_key.png)


## One-To-Many (1:N) Relationship

**Example:** One owner `has_many` pets and a pet `belongs_to` one owner (our `Pet` model will have a foreign key (FK) `owner_id`).

**Always remember!** Whenever there is a `belongs_to` in the model, there should be a *FK in the matching migration!*

### Set Up

1. In the terminal, set up a new Rails app called `practice_associations`:

  ```
  $ rails new practice_associations -d postgresql
  $ cd practice_associations
  $ rake db:create
  ```

2. Also in the terminal, from the root of your Rails app, generate two models, `Owner` and `Pet`:

  ```
  $ rails g model Owner name:string
  $ rails g model Pet name:string
  ```

3. Open your app in Sublime, and define the relationship in both models:

  ```ruby
  #
  # app/models/owner.rb
  #
  class Owner < ActiveRecord::Base
    has_many :pets, dependent: :destroy
  end
  ```

  ```ruby
  #
  # app/models/pet.rb
  #
  class Pet < ActiveRecord::Base
    belongs_to :owner
  end
  ```

  **Note:** When setting up the `has_many` relationship, we use `dependent: :destroy` to maintain data integrity. This means that whenever an owner is deleted (destroyed), that owner's associated pets are also destroyed.

  `belongs_to` uses the singular form of the class name (`:owner`), while `has_many` uses the pluralized form (`:pets`).

  If you think about it, this is exactly how you'd want to describe the relationship in plain English. For example, if we were discussing the relationship between pets and owners, we'd say:

    * "One owner has many pets"
    * "A pet belongs to one owner"

4. Add a foreign key to the pets migration:

  ```ruby
  #
  # db/migrate/20150804001429_create_pets.rb
  #
  class CreatePets < ActiveRecord::Migration

    def change
      create_table :pets do |t|
        t.string :name
        t.timestamps

        # add this line (MOST CORRECT)
        t.belongs_to :owner

        # OR this line
        t.references :owner

        # OR... this line
        t.integer :owner_id

        # but NOT all three!
      end
    end

  end
  ```

  **Options for adding a foreign key:**
  * `t.integer`: adds an integer column to the table for the foreign key
  * `t.references`: more *rails-y* and semantic with a few benefits:
    * Defines the name of the foreign key column (in this case, `owner_id`) for us
    * Adds a **foreign key constraint** which ensures **referential data integrity**  in our database
  * `t.belongs_to`: even more *rails-y* and semantic, with the same functionality as `t.references`

### Using Your Associations

1. Create your database tables by running your migrations from the terminal:

  ```
  $ rake db:migrate
  ```

2. Still in the terminal, enter the rails console (`rails c`) to create and associate data!

  ```ruby
  Pet.count
  Owner.count
  milo = Pet.create(name: "Milo")
  otis = Pet.create(name: "Otis")
  ben = Owner.create(name: "Ben")
  ben.pets
  milo.owner
  ben.pets << otis # makes Otis one of Ben's pets
  # ^ same as:
  # ben.pets.push(milo)
  ben.pets << lassie # makes Lassie another one of Ben's pets
  ben.pets.count
  ben.pets.map(&:name)
  ben.pets.each do |pet| puts "My pet is named #{pet.name}!" end
  otis.owner

  # What will be returned when we do this?
  otis.owner.name
  ```


  **Note:** We just saw that in Rails, we can associate two model **instances** together using the `<<` operator.

#### Wait!!! What if I forget to add a foreign key before running `rake db:migrate`?

If you accidentally run `rake db:migrate` before adding a foreign key to the table's migration, it's ok. There's no need to panic. You can always fix this by creating a new migration:

```
$ rails g migration AddOwnerIdToPets
```

Then modify the migration to include the following:

```ruby
#
# db/migrate/20150804010921_add_owner_id_to_pets.rb
#
class AddOwnerIdToPets < ActiveRecord::Migration

  change_table :pets do |t|
    # only add ONE OF THESE THREE to your new migration
    t.belongs_to :owner #(MOST CORRECT)

    # OR...
    t.references :owner

    # OR...
    t.integer :owner_id

  end

end
```

## Challenges, Part 1: One-To-Many

Jump over to the [One-To-Many Challenges](one-to-many-challenges.md) where you'll work in pairs on a solution.

## Many-To-Many (N:N) with 'through'

**Example:** A student `has_many` courses and a course `has_many` students. Thinking back to our SQL discussions, recall that we used a *join* table to create this kind of association.

A *join* table has two different foreign keys, one for each model it is associating. In the example below, 3 students have been associated with 4 different courses:

| student_id | course_id |
| ---------- | --------- |
| 1          | 1         |
| 1          | 2         |
| 1          | 3         |
| 2          | 1         |
| 2          | 4         |
| 3          | 2         |
| 3          | 3         |

### Set Up

To create N:N relationships in Rails, we use this pattern: `has_many :related_model, through: :join_table_name`

1. In the terminal, create three models:

  ```
  rails g model Student name:string
  rails g model Course name:string
  rails g model Enrollment
  ```

  `Enrollment` is the model for our *join* table. When naming your join table, you can either come up with a name that makes semantic sense (like "Enrollment"), or you can combine the names of the associated models (e.g. "StudentCourse").

2. Open up the models in Sublime, and edit them so they include the proper associations:

  ```ruby
  #
  # app/models/course.rb
  #
  class Course < ActiveRecord::Base
    has_many :enrollments, dependent: :destroy
    has_many :students, through: :enrollments
  end
  ```

  ```ruby
  #
  # app/models/student.rb
  #
  class Student < ActiveRecord::Base
    has_many :enrollments, dependent: :destroy
    has_many :courses, through: :enrollments
  end
  ```

  ```ruby
  #
  # app/models/enrollment.rb
  #
  class Enrollment < ActiveRecord::Base
    belongs_to :course
    belongs_to :student
  end
  ```

3. Add the foreign keys to the enrollments migration:

  ```ruby
  #
  # db/migrate/20150804040426_create_enrollments.rb
  #
  class CreateEnrollments < ActiveRecord::Migration
    def change
      create_table :enrollments do |t|
        t.timestamps

        # define foreign keys for associated models
        t.belongs_to :student
        t.belongs_to :course
      end
    end
  end
  ```

### Using Your Associations

1. In the terminal, run `rake db:migrate` to create the new tables.

2. Enter the rails console (`rails c`) to create and associate data!

  ```ruby
  # create some students
  sally = Student.create(name: "Sally")
  fred = Student.create(name: "Fred")
  alice = Student.create(name: "Alice")

  # create some courses
  algebra = Course.create(name: "Algebra")
  english = Course.create(name: "English")
  french = Course.create(name: "French")
  science = Course.create(name: "Science")

  # associate our model instances
  sally.courses << algebra
  # ^ same as:
  # sally.courses.push(algebra)
  sally.courses << french

  fred.courses << science
  fred.courses << english
  fred.courses << french

  # here's a little trick: use an array to associate multiple courses with a student in just one line of code
  alice.courses << [english, algebra]
  ```

  **Note:** Because we've used `through`, we can create our associations in the same way we do for a 1:N association (`<<`).

3. Still in the Rails console, test your data to make sure your associations worked:

  ```ruby
  sally.courses.map(&:name)
  # => ["Algebra", "French"]

  fred.courses.map(&:name)
  # => ["Science", "English", "French"]

  alice.courses.map(&:name)
  # => ["English", "Algebra"]
  ```

## Challenges, Part 2: Many-To-Many

Head over to the [Many-To-Many Challenges](many-to-many-challenges.md) and work together in pairs.


## Stretch Challenge: Self-Referencing Associations

Lots of real-world apps create associations between items that are the same type of resource.  Read (or reread) <a href="http://guides.rubyonrails.org/association_basics.html#self-joins" target="_blank">the "self joins" section of the Associations Basics Rails Guide</a>, and try to create a self-referencing association in your `practice_associations` app. (Classic use cases are friends and following, where both related resources would be users.)

## Migration Workflow

Getting your models and tables synced up is a bit tricky. Pay close attention to the following workflow, especially the rake tasks.

```
# create a new rails app
rails new my_app -d postgresql
cd my_app

# create the database
rake db:create

# REPEAT THESE TASKS FOR EVERY CHANGE TO YOUR DATABASE
# <<< BEGIN WORKFLOW LOOP >>>

# -- IF YOU NEED A NEW MODEL --
# auto-generate a new model (AND automatically creates a new migration)
rails g model Pet name:string
rails g model Owner name:string

# --- OTHERWISE ---

# if you only need to change fields in an *existing* model,
# you can just generate a new migration
rails g migration AddAgeToOwner age:integer

# never try to create a migration file yourself through the file system! it's really hard to get the name right!

# -- EITHER WAY --
### whether we're creating a new model or updating an existing one, we can manually edit our models and migrations in sublime
# update associations in model --> this affects model interface
# update foreign keys in migrations --> this affects database tables

# generate schema for database tables
rake db:migrate

# <<< END LOOP >>>

# finally, we need some data to play with
# for now, we'll seed it manually, from the rails console...
rails c
> Pet.create(name: "Wowzer")
> Pet.create(name: "Rufus")

# but later we will run a seed task
rake db:seed
```

## Helpful Hints

When you're **creating associations** in Rails ActiveRecord (or most any ORM, for that matter):

  * Define the relationships in your models (the blueprint for your objects)
    * Don't forget to define all sides of the relationship (e.g. `has_many` and `belongs_to`)
  * Remember to put the foreign key for a relationship in your migration
    * If you're not sure which side of the relationship has the foreign key, just use this simple rule: the model with `belongs_to` must include a foreign key.

## Less Common Associations

These are for your references and are not used nearly as often as `has_many` and `has_many through`.

  * <a href="http://guides.rubyonrails.org/association_basics.html#the-has-one-association" target="_blank">has_one</a>
  * <a href="http://guides.rubyonrails.org/association_basics.html#the-has-one-through-association" target="_blank">has_one through</a>
  * <a href="http://guides.rubyonrails.org/association_basics.html#has-and-belongs-to-many-association-reference" target="_blank">has_and_belongs_to_many</a>

## Useful Docs

* <a href="http://guides.rubyonrails.org/association_basics.html" target="_blank">Associations Rails Guide</a>
* <a href="http://edgeguides.rubyonrails.org/active_record_migrations.html" target="_blank">Migrations Rails Guide</a>
