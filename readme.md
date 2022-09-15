### 1. Setup Mysql

-> grab sakila zip from mysql

### 2. Primay and foreign key

primary - unique identifier for our records / good for indexing as well

foreign key - to setup relationships

unsigned-> positive

### 3. Foreign key constraints

Blog db

2 tables :

posts /comments

```sql
CREATE TABLE `posts` (
  `id` int(11) unsigned NOT NULL AUTO_INCREMENT,
  `title` varchar(255) NOT NULL DEFAULT '',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

```sql
CREATE TABLE `comments` (
  `id` int(11) unsigned NOT NULL AUTO_INCREMENT,
  `body` text NOT NULL,
  `post_id` int(11) unsigned DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

we have connected those 2 tables :

```
select * from comments where post_id = 1;
```

we can assign comments to post that does not exist - it is called orphant record

solution is foreign key

```sql
SET FOREIGN_KEY_CHECKS=0 // # if problem with setting fk
alter table comments add foreign key(post_id) references posts(id)
```

we can drop foreign key

```sql
alter table comments
drop foreign key comments_ibfk_1
```

**cascade down**

-> delete all comments related to that post that was deleted

```sql
alter table comments
add foreign key (post_id) references posts(id) on delete cascade
```

but on update the same

```
alter table comments
add foreign key (post_id) references posts(id) on delete cascade on update cascade
```

now we can update post -> and it will be updated inside comments

inventory has got

id | film_id | store_id

1 1 1

store has

store_id | manager_staff_id | address_id

so film of id of 1 is available in store with id of 1

rental table

rental_id | date | inventory id | customer_id | return date | staff_id

### 4. Laravel and Fk constraints

-m is for migration

```php
php artisan make:model Post -m
php artisan make:model Comment -m
php artisan make:migration create_comment_table
```

go to create create_post_table

```
    public function up()
    {
        Schema::create('posts', function (Blueprint $table) {
            $table->bigIncrements('id');
            $table->string('title');
            $table->text('body');
            $table->timestamps();
            $table->timestamps('published_at')->nullable();
        });
    }
```

then run in command:

```
php artisan migrate
```

Go to create_comments_table:

from laravel 5.8

migration - primary key to bigInteger

```php
$table->bigIncrements('id');
	$table->usignedBigInteger('post_id');
	$table->usignedBigInteger('user_id');
            $table->text('body');
            $table->timestamps();
$table->foreign('post_id')
  	->references('id')
  	->on('posts')
    ->onDelete('cascade');


```

```
php aristan migrate:fresh
```

drop table and run once again all schema

```
php artisan migrate:fresh  ->
```

this is for recap :

```php
factory(App\User::class)->create();
```

set up foreign key in migraiton

---

### 5. Difference between mysql join types

od tego

give me everyting from store table

-   join table (address in that example)
-   on condition that store table id = address.id

```sql
select *
from store
join address
on store.`address_id` = address.`address_id`;
```

**differences**

-   join default (inner join) - match criteria from **both** tables

-   left join (outer by default) - **favoured left** side

    > if there is no match give results from left table

-   right join - more results - **favoured **right side

    > if no match - still give results from right table -- we will

---

### 6. Group results with aggregate functions

show me all customer as total number of rentals

we are going to use LEFT join because we want :

Even if there is no rental we **still want to have my customers**

```sql
select
	customer.customer_id, customer.first_name, customer.last_name, rental.rental_id
FROM customer
LEFT join rental
on rental.customer_id = customer.`customer_id`
```

output is :

> we will have a lot of rows repetition / for example for customer_id 1

we want to group similar results into 1

give total of rentals grouped by customer_id ;

on the end we declare how we group " "

```sql
select
	customer.customer_id, customer.first_name, customer.last_name, count(rental.rental_id)
FROM customer
LEFT join rental
on rental.customer_id = customer.`customer_id`
group by customer.customer_id

```

now it returns : count()

we can give that count alias - **rentals_checked_out**

```sql
select
	customer.customer_id, customer.first_name, customer.last_name, count(rental.rental_id) rentals_checked_out
FROM customer
LEFT join rental
on rental.customer_id = customer.`customer_id`
group by customer.customer_id

```

we need to run aggregate function to rental

because we could not smush the table without counting

because sql will not know what to do with rental_id

### 7.Multiple Joins in one query

we have previous sql cod

-> we want to include address of customer nearest store

Basically we are going to work with 3 tables :

**customer** table -> **stores** table -> **address** -> where is location

we want to use subquery (new query)

**mysql is trying to smash all results into 1 row ** - aggregation

we need to add it as unique in the end to **group by** - pass address

-   we add store_address in the end as alias

```sql
select
	customer.customer_id,
	customer.first_name, customer.last_name, count(rental.rental_id) rentals_checked_out,
	address.address store_address
FROM customer
LEFT join rental
on rental.customer_id = customer.`customer_id`
LEFT join address
on address.address_id = (
	SELECT address_id from store where store.store_id = customer.store_id
)

group by customer.customer_id, address.address


```

subcqueries come with speed cost

to make it faster:

```sql
select
	customer.customer_id,
	customer.first_name, customer.last_name, count(rental.rental_id) rentals_checked_out,
	address.address store_address
FROM customer
LEFT join rental
on rental.customer_id = customer.`customer_id`
LEFT join store
on store.store_id = customer.`store_id`
LEFT join address
ON address.address_id = store.address_id

group by customer.customer_id, address.address


```

so here :

1 ) join rental - referring to rental column + customer (which we had)

2.  join store - referring to store + customser

3.  join address - ferrring to addrse + store column

---

### 8. Filtering aggragated data

**what is the most popular rentals in our store - most profitable - need sum**

-   we work with all rentals
-   we need informatino about payment (is joined with rentals)
-   we do not know what rental name is - we need to grab from invetory table
-   invetory table is a look up table joinning film_id

we joining rental + payment + invetory + film (4 tables

```sql
select * from rental
join payment
on payment.rental_id = rental.rental_id
join inventory
on inventory.`inventory_id` = rental.inventory_id
join film
on film.film_id = inventory.film_id
```

we need to simplify -> need title

sum of particular record: amount

name as sales

```sql
select title, sum(amount) sales from rental
join payment
on payment.rental_id = rental.rental_id
join inventory
on inventory.`inventory_id` = rental.inventory_id
join film
on film.film_id = inventory.film_id
group by title
order by sales desc
```

-> get number of records -> naming as rentals

```sql
select title, sum(amount) sales,  count(*) rentals from rental
join payment
on payment.rental_id = rental.rental_id
join inventory
on inventory.`inventory_id` = rental.inventory_id
join film
on film.film_id = inventory.film_id
group by title
order by sales desc
```

-> show only those which earn more than 200

where > reference actual column we are selecting from

sum(count) is not a column

rentals is an aggregate

**having keyword** - act on rows after they were grouped

```sql
select title, sum(amount) sales,  count(*) rentals from rental
join payment
on payment.rental_id = rental.rental_id
join inventory
on inventory.`inventory_id` = rental.inventory_id
join film
on film.film_id = inventory.film_id
group by title
having sales > 200
order by sales desc
```

---

## Laravel - Eloquent Relationship

### One-to-one

user has 1:

-   address
-   profile
-   one set of experience

```bash
php artisan make:model Profile -m
```

create method inside User class

```php
    public function profile()
    {
        return $this->hasOne(Profile::class);
    }
```

tinker

```php
$user = User::first();
```

for debugging we can run ->

db facade - with query and bindings : ]

```bash
DB::listen(function($sql){var_dump($sql->sql, $sql->bindings); });
$user = User::first();
```

> string(29) "select \* from `users` limit 1"
> array(0) {
> }

```php
DB::enableQueryLog();
$user = User::first();
DB::getQueryLog();
```

```
=> [
    [
      "query" => "select * from `users` limit 1",
      "bindings" => [],
      "time" => 0.56,
    ],
  ]

```

```bash
php artisan make:model Experience -m
```

User@experience

```php
return $this->hasOne(Experience::class);
```

```bash
>>> $user->experience->points
=> 500

```

### One-To-many

many posts

```bash
php artisan make:model Post -mf
```

PostFatory:

```php
   public function definition()
    {
        return [
            //
            'user_id' => User::factory()->create()->id,
            'title' => $this->faker->sentence,
            'body' => $this->faker->sentence
        ];
    }
```

User@posts

```php
    public function posts()
    {
        return $this->hasMany(Post::class);
    }
```

in tinker

```php
Post::factory()->create()
User::find(2)->posts
```

### Many-to-Many

Post can have many comments

```php
class Post extends Model
{
    public function comments()
    {
        return $this->hasMany(Comment::class);
    }
}

class Comment extends Model
{
    public function post()
    {
        return $this->belongsTo(Post::class);
    }

}
```

**we need 3 different table**

Posts / tags / post_tag (**pivot table - naming : single version in alphabetical order**)

```
php artisan make:migration create_tags_table --create=tags
php artisan make:migration create_post_tag_table --create=post_tag
```

Tag belong to many Posts

Posts belong to many tags

```php
 public function posts()
    {
        return $this->belongsToMany(Post::class);
    }
```

```
$post = Post::find(2);
$post->tags->pluck('name');
$post = Post::find(1);
$post->tags()->attach([1,2]);

```

**has many through**

blog with political affiliation

sql

```sql
 select * from users where affiliation_id = 1;
 select id from users where affiliation_id = 1; # 5,6
 select * from posts where user_id in (5,6)
```

Go to Affilation model

-   fetch all post who are affilated with

> hasMany posts through User table

```php
namespace App;

use Illuminate\Database\Eloquent\Model;

class Affiliation extends Model
{

    public function posts()
    {
        return $this->hasManyThrough(Post::class, User::class);
    }
}

```

```php
$rightWing = App\Affiliation::whereName('conservative')->first();
$rightWing->posts;
```

### Polymorphic

```bash
php artisan make:model Video -m -f
php artisan make:model Series -m -f
```

Videos -> belongs to Series

Series -> has many Videos

```php
   public function videos()
    {
        return $this->hasMany(Video::class);
    }

public function series()
    {
        return $this->belongsTo(Series::class);
    }
```

Video can belong to series / sometimes to Collection

in video migration

```php
        Schema::create('videos', function (Blueprint $table) {
            $table->id();
            $table->morphs('watchable');
```

update Series & Collection

```php
class Series extends Model
{
    use HasFactory;
    public function videos()
    {
        return $this->morphMany(Video::class, 'watchable');
    }
}

```

then in Video

```php
class Video extends Model
{
    use HasFactory;
    public function watchable()
    {
        return $this->morphTo();
    }
```
