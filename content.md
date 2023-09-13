# Getting started with scaffolds

## Walkthrough video

<div class="bg-red-100 py-1 px-5" markdown="1">
**Please note**, the video is from a previous iteration of the project, so there are some differences:

- The project was previously called `ad2-getting-started`, but it has been renamed to `getting-started-with-scaffolds`
- I am using Gitpod as my cloud editor, so the interface looks a bit different.
- Anything contained in the project "README" is now contained in this Lesson
- I use `bin/server` to start my live app preview, _you_ should use `bin/dev`
- I use `rails db:migrate`, _you_ should use `rake db:migrate`
- You should drop the `.html.erb` when rendering a view template:
    `render({ :template => "photo_templates/index" })` 
    _instead of_ 
    `render({ :template => "photo_templates/index.html.erb" })`
</div>

Did you read the differences above? Good! Then [here is a walkthrough video for this project.](https://share.descript.com/view/nTBab1wF2JO)

The lesson below contains some additional details, but the video should mostly guide you through the project.

## Getting started

There is no target for this app, this is just a playground to repeat and levelup what we've learned so far using industry standard (i.e. Professional) code.

This project does include a few basic automated tests, so click on this button to get started:

LTI{Load Getting Started with Scaffolds assignment}(https://grades.firstdraft.com/launch)[S9ymPy6WCsn18gLbByVbZQ7k]{vfdtzJb5bLYqYwuqgeRKpc5d}(5)[Getting Started with Scaffolds Project]

Launch your live app preview with `bin/dev`, and you will be greeted by a brand new, blank Rails app. Let's get started filling it in.

## `draft:resource`

This bash prompt command represents the culmination of what we've learned thus far:

```
rails generate draft:resource movie title:string description:text released:boolean
```

If you try and run it now, you will get an error. That's because _we_ wrote the `draft:resource` generator and it doesn't come with Rails out of the box!

We wrote it in order to make it easier to learn in the early stages. But, moving forward, you should avoid using the `draft:resource` generator. We're going to learn more powerful generators that come with Rails. _So this is the last time where I'm going to generate one of these._

Let's head over to our `Gemfile`, which is located in the root folder, and pull in the `draft_generator` gem:

```ruby{8}
# Gemfile

source "https://rubygems.org"
git_source(:github) { |repo| "https://github.com/#{repo}.git" }

ruby "3.2.1"

gem "draft_generators"
# ...
```

Now, at the bash prompt run:

```
bundle install
``` 

(Or just `bundle` for short.) 

This command will go get all of the gems from our `Gemfile` and install them. 

Now try the bash prompt command again:

```
rails generate draft:resource movie title:string description:text released:boolean
```

That should work, and output:

```
    create  app/controllers/movies_controller.rb
    invoke  active_record
    create    db/migrate/20230913004129_create_movies.rb
    create    app/models/movie.rb
      exist  app/views/movies
    create  app/views/movies/index.html.erb
    create  app/views/movies/show.html.erb
      route  RESTful routes
    insert  config/routes.rb
```

As usual, we need to `rake db:migrate` to run the pending migrations.

Start your live app preview with `bin/dev` and visit `/movies`. This is the `movies#index` action page, where there is a form to add things to the table in our database.

Try to add one. What happens? 

An error we haven't encountered before:

```
ActionController::InvalidAuthenticityToken at /insert_movie
Can't verify CSRF token authenticity.
```

<div class="bg-red-100 py-1 px-5" markdown="1">
Note, in the walkthrough video, I get an error message in the browser. You won't see this error due to some Rails 7 updates. But nevertheless follow along with the next section to add the authenticity token to your POST request!
</div>

## HTTP POST authenticity

Let's have a look at that form in the `app/views/movies/index.html.erb` view template:

```erb{3}
<!-- app/views/movies/index.html.erb -->

    <form action="/insert_movie" method="post">
    
    <!-- ... -->

    </form>
```

Here and in the `config/routes.rb` file, we are using a POST request, since we are modifying the database with this `create` action.

However, up to this point, we have not been using POST entirely correctly. We left a security hole! Happily, Rails and tens of thousands of companies that use Rails, have encountered these security holes and performance issues and built solutions for them. It requires a little bit of extra work to use these solutions built into Rails. Previously, we disabled them all to make it simpler to get something up and running. But now we've re-enabled all of those default protections. 

**Here's what's going on here:** When I submitted the form, it did a POST to `"/insert_movie"`. Now that we are using POST for our update and create actions to _modify our database_, Rails is going to expect that POST to come along with a password. That guarantees that the form was created by us and not by a malicious third party.

If we just blindly accepted that HTTP request, somebody could put this form on their own website, but have it point to _our_ website (e.g., by replacing the `<form>` `action` attribute with: `action="https://our-app-domain.com/insert_moview`). Then they would be able to modify our database!

That's called **cross-site request forgery**. 

What's the solution? In the form, we're going to put a hidden input that has a "password", and Rails is going to randomly generate a password every time we draw this form:

```erb{4}
<!-- app/views/movies/index.html.erb -->

    <form action="/insert_movie" method="post">
      <input type="hidden" name="authenticity_token" value="<%= form_authenticity_token %>">
    
    <!-- ... -->

    </form>
```

We added a new form `<input>` with some attributes. First, we set the `type="hidden"`, so the input is not displayed to the user. Then we gave the input a `name="authenticity_token"`. Finally, we filled it in with a `value`, which is a rendered Ruby method `<%= form_authenticity_token %>`.

The method `form_authenticity_token` is known as a **helper method** and it comes with Rails out of the box. This method uses cryptography to generate and return a long string token that acts as our "password" for Rails to check and make sure the request is from _us_ and not an outside source. The string is scrambled going into and out of Rails, so even if someone did an "Inspect" on the browser view of our page, the token would not be useful for getting into our database. (`authenticity_token` is the name in the `params` hash that Rails will automatically look for to extract the token, so we had no choice but to name it so.)

With this one-line addition to the form code, try and re-submit a new movie on the `/movies` page. 

Boom. Error message begone. All of a sudden we have **CSRF** protection. One of the many things that we would never have thought of building until we got hacked. Rails just has this out of the box so that you can't even _build_ forms without solving this problem first!

Again, we did this all with the **helper method** `form_authenticity_token` that comes with Rails. We actually already saw some other Rails helper methods, for instance, remember `time_ago_in_words`? There's a whole bunch of these helper methods that we need to learn to level up our codebase.

## RESTful routes

Look at our routes:

```ruby
# config/routes.rb

# ...
  # CREATE
  post("/insert_movie", { :controller => "movies", :action => "create" })
          
  # READ
  get("/movies", { :controller => "movies", :action => "index" })

  get("/movies/:path_id", { :controller => "movies", :action => "show" })
  
  # UPDATE
  post("/modify_movie/:path_id", { :controller => "movies", :action => "update" })
  
  # DELETE
  get("/delete_movie/:path_id", { :controller => "movies", :action => "destroy" })
# ...
```

We started with `/movies` for the `index` page, and for the details of an individual movie (`show`), we did `/movies/ID`, with the dynamic route segment `"/movies/:path_id"`. Then we added `"/insert_movie"` as a way to trigger an action that `create`s it.

All of these were originally HTTP GET requests (`get` in Ruby on Rails), and we only later evolved the `create` path to an HTTP POST request (`post` in Ruby on Rails). We named these paths in a way that made it really obvious to us what the intention was. We specifically _chose_ names that revealed our intentions for what action was supposed to trigger on these URL paths.

In the real world, professional developers don't do this! 

The way we name our URLs shouldn't have to include what we're trying to do with the resource. The path should only identify the resource that we're trying to work with (here, `movies`). And then the **HTTP verb** (`get`, `post`, etc.) is supposed to tell us what operation to perform on it.

`get` (GET) indicates reading. So the route:

```ruby
get("/movies", { :controller => "movies", :action => "index" })
```

is good, because our path doesn't say `read_movie`, or something like that, it just says `movie`, which is the resource (the table in our database) we want to read from. `get` tells us our intention. 

This is a topic called REST or REST API or [RESTful API](https://restfulapi.net/resource-naming/): if your routes are RESTful, then they're adhering to the naming convention that most other web developers use. 

That means, *we don't use which CRUD function is being performed in the URL itself, rather the request method should be used to indicate which CRUD function is happening.*

In practice, it's fairly simple. All of our routes are going to start with slash and then the plural version of the table name. So our `# READ` routes are good:

```ruby
# READ
get("/movies", { :controller => "movies", :action => "index" })

get("/movies/:path_id", { :controller => "movies", :action => "show" })
```

But, we need to change the `# CREATE` route, so it doesn't say `"/insert_movie"`:

```ruby
# CREATE
post("/movies", { :controller => "movies", :action => "create" })
```

Now, even though our movie index page with the `index` action and our insert movie page with the `create` action share the same URL path (`/movies`), Rails won't get confused, because they are using different HTTP verbs to specify what type of request is being made (GET / `get` vs. POST / `post`).

We also need to change the `# UPDATE` route: 

```ruby
# UPDATE
patch("/movies/:path_id", { :controller => "movies", :action => "update" })
```

We changed the route name to start with our resource of interest (`"/movies"`), and we changed the **HTTP verb** to a new method `patch` (for an HTTP PATCH request).

And again, even though our movie details page with the `show` action and our update movie page with the `update` action share the same URL path the difference in HTTP verbs will prevent any confusion (GET / `get` vs. PATCH / `patch`).

Similarly, to delete a movie, we're going to use the HTTP verb DELETE (`delete`):

```ruby
# DELETE
delete("/movies/:path_id", { :controller => "movies", :action => "destroy" })
```

Despite all of the similar URL paths in our routes (`"/movies/ID"`), none of them will get mixed up because they have different verbs!

We do need to update the URLs now anywhere else they appear in our app. For instance, in the `app/views/movies/index.html.erb` page form, we need to change this:

```erb
<form action="/insert_movie" method="post">
```

to this:

```erb
<form action="/movies" method="post">
```

And we don't need to worry about confusion, because Rails will look for the route with the matching `method` (here, that's POST, or `post`).

But what if we manually type in `/movies` to the address bar? Which operation will be performed. Well, the browser can _only_ perform GET requests! So that method will be used and the index of movies will be shown.

## No `.at` for ActiveRecord

On the `/movies` index page, try to click on the "Show details" link for the movie you added.

We get an error message `undefined method 'at' for #<ActiveRecord:Relation...`, pointing to this line in our `MoviesController`:

```ruby{10}
# app/controllers/movies_controller.rb

class MoviesController < ApplicationController
  # ...
  def show
    the_id = params.fetch("path_id")
    
    matching_movies = Movie.where({ :id => the_id })
    
    @the_movie = matching_movies.at(0)
    
    render({ :template => "movies/show.html.erb" })
  end
  # ...
```

Previously, I told you all to just think of an `ActiveRecord:Relation` like an array, and to use the familiar array methods (like `.at`). But, it's not actually an array! Sorry! In the real world, `.at` is not defined for `ActiveRecord:Relation`, we just added it to make it more familiar for us beginners. From now on, when you have an `ActiveRecord:Relation`, and you're trying to get a single element out of it, you could do square bracket syntax:

```ruby
@the_movie = matching_movies[0]
```

or you could do `.first`:

```ruby
@the_movie = matching_movies.first
```

These methods are shared between arrays and `ActiveRecord:Relation`s. Square brackets are very common on the internet (and in other programming languages), so I recommend getting used to that. 

Try the "Show details" link again after you make this change. Working? Great! Be sure to make this change everywhere else you find a `.at(0)` in your controller.

## DELETE request

Now, on the details page for a movie, try to click "Delete movie". What's the error telling us?

```
No route matches [GET] "/delete_movie/1"
```

Right, we changed that route to be RESTful using the `delete` method, so in our `show.html.erb` view template, we need to change the link:

```erb{3}
<!-- app/views/movies/show.html.erb -->

        <a href="/delete_movie/<%= @the_movie.id %>">
          Delete movie
        </a>
```

to this:

```erb
<a href="/movies/<%= @the_movie.id %>"> 
```

Make that change and try to "Delete movie" again. Nothing happens. Because this is _still a GET request_. And if you do `get("/movies/:id"...)`, our routes indicate that this should call the `show` action. So now we have a conflict between the two. How do we fix it?

We need to make this `<a>` link tag submit a DELETE request instead of a GET request. Sadly, we can't just add an attribute to the `<a>` tag `method="delete"`. HTML only has POST and GET, because the people who wrote HTML are different than the committee who wrote HTTP, and they apparently didn't share the memo.

Fortunately though, Rails includes some fanciness that will let us fake this pretty easily. Add this attribute:

```erb{1:(40-66)}
<a href="/movies/<%= @the_movie.id %>" data-turbo-method="delete"> 
```

<div class="bg-red-100 py-1 px-5" markdown="1">
Note, in the walkthrough video I used `data-method`, but in our Rails 7 update, this attribute is renamed `data-turbo-method`.
</div>

Now go back and try the "Delete movie" link once more. It worked! (Unless you forgot to change `.at(0)` to `[0]` in the controller.)

With Rails, we can just add this attribute onto a link, and Rails will add some stuff in the background that makes it appear like a DELETE request to our server.

## PATCH request

Now let's fix the "Edit movie" form that we use to update a database entry. This form is at the bottom of the `show` page:

```erb
<!-- app/views/movies/show.html.erb -->

    <form action="/modify_movie/<%= @the_movie.id %>"  method="post" >
```

Right away, we know we can switch the `action` to point to the correct URL:

```erb{1:20-26}
    <form action="/movies/<%= @the_movie.id %>"  method="post" >
```

And now we're going to have the same conflict issue that we did with the delete method. We wish we could just change the `method` to `"patch"`, but that's not an option in plain HTML. We also can't just use `data-method="patch"`, similar to how we did it with the delete link.

Well, let's start by adding the authenticity token to our `post` request so it would actually work:

```erb{2}
    <form action="/movies/<%= @the_movie.id %>"  method="post" >
        <input type="hidden" name="authenticity_token" value="<%= form_authenticity_token %>">
```

Now we need to add the following:

```erb{4}
    <form action="/movies/<%= @the_movie.id %>"  method="post" >
        <input type="hidden" name="authenticity_token" value="<%= form_authenticity_token %>">

        <input type="hidden" name="_method" value="patch">
```

Another hack! Once again, this bit of code is just some Rails fanciness to trick the HTTP server into making a PATCH rather than a POST request. We _have_ to name the new hidden input `"_method"` because Rails will be looking for that key in the `params` hash, and if it sees that key, then it will know to work its magic based on the value (here, `"patch"`).

<div class="bg-red-100 py-1 px-5" markdown="1">
**There's no need to memorize all of these oddities**, because soon we'll use some advanced helper methods to generate all of this automatically.
</div>

## Dynamic path naming

In our `config/routes.rb` file, we were always using `:path_id` for the dynamic route segment indicating a movie's ID column. We made this name up to make it clear to us that this is a parameter that is going to appear in the address bar, e.g., at `/movies/1`. Then, we know in the `params` hash when we see `path_id`, it is coming from the URL path as opposed to a query string. 

In the real world, developers just call it `:id`, so all of our routes `"/movies/:path_id"` should be changed to `"/movies/:id"`.

If you switch the dynamic path segments in the `config/routes.rb` from this:

```ruby
get("/movies/:path_id", { :controller => "movies", :action => "show" })
```

to this:

```ruby
get("/movies/:id", { :controller => "movies", :action => "show" })
```

Then you need to remember to change the `params` key in the `app/controllers/movies_controller.rb` file for that action (in the example, `show`). From this:

```ruby
    the_id = params.fetch("path_id")
```

to this:

```ruby{1:(27-31)}
    the_id = params.fetch("id")
```

And we need to make this change everywhere (`path_id` to `id`) in our codebase. 

You can try selecting the offending characters and [creating multiple cursors](https://code.visualstudio.com/docs/editor/codebasics#_multiple-selections-multicursor) to change them. Or you can try a [global find and replace](https://learn.microsoft.com/en-us/visualstudio/ide/finding-and-replacing-text?view=vs-2022#find-and-replace-control). But if you do any major find-and-replace operations, be sure to `git commit` before so you can rewind if you mess things up.

With that, our routes are done. They are industrial grade. The next thing to do is build separate pages for create and update actions instead of having them on the show and index pages.

## The Goal: Scaffolds

Why are we doing all of this seemingly arbitrary work? Switching naming conventions and making our routes RESTful? 

Let's look at the command we're going to start using rather than `draft:resource`:

```
rails generate scaffold book title:string description:text released:boolean
```

It's just another Rails generator. There's a name for a model (`book`) and columns (`title`, etc.). But the difference is the name of the generator we choose: `scaffold`. 

When we run this generator, look what happens:

```
      invoke  active_record
      create    db/migrate/20230913023116_create_books.rb
      create    app/models/book.rb
      invoke  resource_route
       route    resources :books
      invoke  scaffold_controller
      create    app/controllers/books_controller.rb
      invoke    erb
      create      app/views/books
      create      app/views/books/index.html.erb
      create      app/views/books/edit.html.erb
      create      app/views/books/show.html.erb
      create      app/views/books/new.html.erb
      create      app/views/books/_form.html.erb
      create      app/views/books/_book.html.erb
      invoke    resource_route
      invoke    jbuilder
      create      app/views/books/index.json.jbuilder
      create      app/views/books/show.json.jbuilder
      create      app/views/books/_book.json.jbuilder
```

Woah! Unlike the `draft:resource` generator, I didn't have to add a gem to get this to run. It's just built in to Rails. And look at all of the stuff it does!

As before, `rake db:migrate`, visit `/books` in your live app preview, click around, and examine the code that was generated (`config/routes.rb`, `app/controllers/books_controller.rb`, etc).

Phew! There's a lot of mysterious new stuff. We're going to demystify it all over the next few weeks, step-by-step.

One thing to note, where are our routes? It looks like only _one line_ got added to the `config/routes.rb` file:

```ruby
  resources :books
```

But `/books` showed us the standard CRUD interface. And we can check the other routes by navigating to `/rails/info` in our live app preview.

All of the book routes exist! 

With `scaffold`, if you give it the name of the database table that you want to build a frontend interface around, it writes all of the routes and it writes them RESTfully.

Imagine if that's how you were introduced to Rails on day one? Pretty confusing for a newcomer; but we've been building up to this and we're almost there!

Let's also take a quick look at some of the controller actions:

```ruby
# app/controller/books_controller.rb

class BooksController < ApplicationController
  before_action :set_book, only: %i[ show edit update destroy ]

  # GET /books or /books.json
  def index
    @books = Book.all
  end

  # GET /books/1 or /books/1.json
  def show
  end

  # GET /books/new
  def new
    @book = Book.new
  end

  # GET /books/1/edit
  def edit
  end
# ...
```

Where's all of the code? Well, once we adopt the conventional names for our actions, our view templates, our variables, and everything else, all of that boiler plate code can be removed. Rails will figure out what we mean, because of the _conventional naming_.

For example, the `show` template is just called `show.html.erb`, so we don't need to have the line `render({ :template => "books/show" })`. We're already in the `BooksController` and the template is named after the action, pretty easy for Rails to figure out, yeah?

<aside markdown="1">
You don't _have to_ use conventional names, but if you stray from convention then you would need to add some lines fo code. Like, for the `show` action, if you named your view template `details_page.html.erb`, then you would need to point that out to Rails by adding `render({ :template => "books/details_page" })`.
</aside>

Here's an example of where `scaffold` added code that we haven't seen: 

```ruby{11-12}
# app/controller/books_controller.rb

class BooksController < ApplicationController
  # ...
  # POST /books or /books.json
  def create
    @book = Book.new(book_params)

    respond_to do |format|
      if @book.save
        format.html { redirect_to book_url(@book), notice: "Book was successfully created." }
        format.json { render :show, status: :created, location: @book }
      else
        format.html { render :new, status: :unprocessable_entity }
        format.json { render json: @book.errors, status: :unprocessable_entity }
      end
    end
  end
  ...
```

The generator includes responses for multiple formats besides HTML (e.g., above for JSON)!

It might seem like what we're doing is arbitrary. We're changing the names of routes that were already working and we're actually making the names less understandable. But, the benefit is that eventually we'll get to use all of these shortcuts to avoid typos and bugs by typing less code. 

## Reusing forms

On the `index` or `show` pages for `/books`, there's no form to add a book or edit a book. Instead, there's a link that brings you to that form. 

Let's move the `/movies` form on the `index` page to its own page as well.

Starting with `config/routes.rb`, add the new route we want:

```ruby{7}
# config/routes.rb

Rails.application.routes.draw do
  resources :books

  # Routes for the Movie resource:

  get("/movies/new", { :controller => "movies", :action => "new" }))

  # CREATE
  post("/movies", { :controller => "movies", :action => "create" })     
  # ...
```

<aside markdown="1">
Be sure your new `get("/movies/new"...)` route is _above_ the `get("/movies/:id"...)` route that is already in `config/routes.rb`. If it was _below_ this route, then there would be a conflict and anytime we tried to visit `/movies/new` Rails would treat **new** in the path as an `:id`, which wouldn't work.
</aside>

Now we need to add the (conventionally named) `new` action to our controller:

```ruby{4-6}
# app/controllers/movies_controller.rb

class MoviesController < ApplicationController
  def new
    render template: "movies/new"
  end
  # ...
```

To continue with our RCAV, let's go to the live app and navigate to `/movies/new`. A `Missing template...` error! Let's create that as `app/views/movies/new.html.erb`, and cut the POST form out of the `index.html.erb` page and paste it into the `new.html.erb` page:

```erb
<!-- app/views/movies/new.html.erb -->

<h2>
  Add a new movie
</h2>

<form action="/movies" method="post">
  <input type="hidden" name="authenticity_token" value="<%= form_authenticity_token %>">

  <!-- ... -->

  <button>
    Create movie
  </button>
</form>
```

Refresh `/movies/new` and test the form. Working? Great!

Now, on your own, do the same with the editing form on the `show` page. Start with a new route:

```ruby
get "/movies/:id/edit", controller: "movies", action: "edit"
```

<aside markdown="1">
Why are we using `get` on the new `edit` action? Because all this route is doing is GETting the `edit` page for us, not making any database changes. But on the `edit` form, we have our `patch` method to update the database record.
</aside>

Then add the `edit` action, view template, and cut-paste the edit form into the new view.

When you're done, try the new `/movies/[ID]/edit` route for one of the movies. We get an error `undefined method 'id' for nil:NilClass`. And we can see on the error page, that the problem is with the instance variable `@the_movie`. That's `nil` on our new page, because it doesn't exist in our `edit` action back in the controller!

Back in the controller, we'll need to:

```ruby
# app/controllers/movies_controller.rb

class MoviesController < ApplicationController
  # ...
  def edit
    @the_movie = Movie.where(id: params.fetch(:id))[0]
    render template: "movies/edit"
  end
  # ...
```

<aside markdown="1">
Why did we write `params.fetch(:id)`? Isn't the ID a string in the `params` hash? Well `params` is not actually a hash, it's a subclass of hash that Rails created. It can use _both_ symbols and strings. Most of the time you're going to see symbols.
</aside>

And now the edit form should work. Try it out to be sure.

## Form errors

Let's take an aside, and talk about the user experience of filling out a form. What if somebody submits a movie with a blank title? Typically we don't want to allow that and we want to notify the user so they can correct it. 

Let's look at what happens with the `/books` scaffold when we try adding a book with a blank title. Right now, it lets us do that if we use the form at `/books/new`. We need to add **validations** to prevent this from happening! We can do that in the `Book` model:

```ruby{4-5}
# app/models/book.rb

class Book < ApplicationRecord
  validates :title, presence: true
  validates :description, presence: true
end
```

With this, the `Book` model will not allow a `.save` to be called on it _unless_ there's a value for those columns.

Now look at the UI when we mess up the form. It should give you a list of everything that went wrong and how to correct it. Nice!

How can we implement something like this on our `/movies/new` form?

First, let's add the validations to `Movie`:

```ruby{4-5}
# app/models/movie.rb

class Movie < ApplicationRecord
  validates :title, presence: true
  validates :description, presence: true
end
```

Now, let's have a look at the `create` action in the `MoviesController`:

```ruby
# app/controllers/movies_controller.rb

class MoviesController < ApplicationController
  # ...
  def create
    the_movie = Movie.new
    the_movie.title = params.fetch("query_title")
    the_movie.description = params.fetch("query_description")
    the_movie.released = params.fetch("query_released", false)

    if the_movie.valid?
      the_movie.save
      redirect_to("/movies", { :notice => "Movie created successfully." })
    else
      redirect_to("/movies", { :alert => the_movie.errors.full_messages.to_sentence })
    end
  end
  # ...
```

We instantiated a `.new` record, take columns from the form with the query string and `params` hash, and assign them to the columns in our `the_movie` database record. Then we ask if `the_movie` is `.valid?`, which returns true or false based on whether all of the validation rules in the `Movie` model pass or not.

If it _is_ valid, we `.save` and redirect. If it is _not_ valid, we just redirect. The only difference with the redirection is that we have different `:notice` and `:alert` messages. 

But we aren't doing anything with that `:alert` message yet, so we don't get any notification on our `Movie` model that something went wrong if a validation doesn't pass. And we're just redirecting back to `/movies` rather than staying on the form. Use the `/movies/new` form to add a movie with and without a title and see how it works.

Now might be a good time to add a link to the new movie form on the index page:

```erb{9}
<!-- app/views/movies/index.html.erb -->

<div>
  <div>
    <h1>
      List of all movies
    </h1>

  <a href="/movies/new">Add a new movie</a>
  </div>
</div>
```

Let's improve the UI on the form.

We can add the notice and alert messages to the application layout, so that they render on every page:

```erb{5-6}
<!-- app/views/layouts/application.html.erb -->

<!-- ... -->
  <body>
    <%= notice %>
    <%= alert %>

    <%= yield %>
  </body>
</html>
```

Now the messages will appear on top of the page, which is rendered at the location of `<%= yield %>`. 

Try to enter a movie with a blank title at `/movies/new` and you will see the `alert` message, which is being generated in the controller from `the_movie.errors.full_messages.to_sentence`. 

Great! But let's also keep the user on the form when they mess something up. In the `MoviesController` file, change this:

```ruby
redirect_to("/movies", { :alert => the_movie.errors.full_messages.to_sentence })
```

to this:

```ruby{1:(21-25)}
redirect_to("/movies/new", { :alert => the_movie.errors.full_messages.to_sentence })
```

Now, when the user tries to input a blank movie, they remain on the form and are given an error message. But all of their progress on the form has been wiped out! What if they wrote a long description? That would be gone and they would need to retype it. Not great.

When `the_movie.valid?` is false, and I follow the `else` branch of the control flow in my controller `create` action, instead of wiping out all of the values, I would like to prepopulate the form with the data that has already been entered. We could do this with `cookies`, but let's see another (maybe more straightforward) way.

When `the_movie.valid?` returns false, and we go to the `else` statement, you can see we are still using `the_movie` object to get the `.errors`. So `the_movie` object is still available to us in that branch. And prior to that `.valid?` method, we _already stored values_ in the attributes of this new object that we just instantiated.

That means, `the_movie` has everything we need to redraw the form with the entered data. However, when we `redirect_to` a new page, it is the same as the user typing the name into their address bar. Which means we lose the access to anything from this `create` action. It begins a whole new RCAV starting from `/movies/new`, going to the `new` action and rendering the template without any instance variables defined in that `new` action.

```ruby
# app/controllers/movies_controller.rb

# ...
  def new
    render template: "movies/new"
  end
# ...
```

Okay, then rather than `redirect_to`, let's instead make `the_movie` an instance variable (`@the_movie`) and `render`:

```ruby{14}
# app/controllers/movies_controller.rb

# ...
  def create
    @the_movie = Movie.new
    @the_movie.title = params.fetch("query_title")
    @the_movie.description = params.fetch("query_description")
    @the_movie.released = params.fetch("query_released", false)

    if @the_movie.valid?
      @the_movie.save
      redirect_to("/movies", { :notice => "Movie created successfully." })
    else
      render template: "movies/with_errors"
    end
  end
# ...
```

Now we create that new view template where we have access to the `@the_movie` object:

```erb
<!-- app/views/movies/with_errors.html.erb -->

<%= @the_movies.inspect %>
<%= @the_movies.errors.full_messages.to_sentence %>
```

And now, if we enter a movie with a blank title at `/movies/new`, we will be taken to our page with the `ActiveRecord` object and the errors. And we can see that the object does contain any information that the user filled out!

Let's copy-paste the form from our other `new.html.erb` template into this `with_errors.html.erb` view template, and prepopulate the values with our instance variable:

```erb
<!-- app/views/movies/with_errors.html.erb -->

<h2>
  Add a new movie
</h2>

<form action="/movies" method="post">
  <input type="hidden" name="authenticity_token" value="<%= form_authenticity_token %>">

  <div>
    <label for="title_box">
      Title
    </label>

    <input type="text" id="title_box" name="query_title" value="<%= @the_movie.title %>">
  </div>

  <div>
    <label for="description_box">
      Description
    </label>

    <textarea id="description_box" name="query_description" rows="3"><%= @the_movie.description %></textarea>
  </div>

  <div>
    <input type="checkbox" id="released_box" name="query_released" value="1">

    <label for="released_box">Released</label>
  </div>

  <button>
    Create movie
  </button>
</form>
```

(The HTML `<textarea>` element does not have a `value` attribute, so we had to prepopulate the field itself.)

We didn't add the `alert:` to our `render template: "movies/with_errors"` line, so we aren't getting any message if we mess things up, but at least the form is staying prepopulated now. Let's add some messages though:

```erb
<!-- app/views/movies/with_errors.html.erb -->

<h2>
  Add a new movie
</h2>

<% @the_movie.errors.full_messages.each do |msg| %>
  <li><%= msg %></li>
<% end %>

<form action="/movies" method="post">
```

Test out the form. Do you see error messages when you leave something blank? Does the form stay prepoulated? What about when you succeed? Are you redirected to the index page with your movie added to the table? Yes to all? Great!

## Reusing `new` view template

In the `MoviesController#create` action, we changed the "sad" branch, where `@the_movie` is _not_ valid:

```ruby
  # ...
  def create
    # ...
    if @the_movie.valid?
      @the_movie.save
      redirect_to("/movies", { :notice => "Movie created successfully." })
    else
      render template: "movies/with_errors"
    end
  # ...
```

Because we're rendering a template rather than redirecting, we have access to `@the_movie` in the `with_errors.html.erb` template. It was _not_ valid for saving, but that doesn't mean the variable was deleted or further modified. It just never entered the database. Any of the values that we assigned in the `create` action (e.g., `@the_movie.title = params.fetch("query_title")`) are still available to prepopulate the form in the template.

<aside markdown="1">
Playing with a `Movie` object in the `rails console` is a good way to understand what is happening:

```ruby
pry(main)> m = Movie.new 
pry(main)> m.title = "Some title" 
pry(main)> m.save 
```

That should cause an error, then you can inspect the error messages with:

```ruby
pry(main)> m.errors.full_messages
```

And if you print `m` or do an `m.inspect`, you will see that the `title` is still filled in despite the save error.
</aside>

This new template, `with_errors`, that we are using to `create` a new object is similar to what we've done in the past with edit forms. In the case of edit or `update` actions, we looked up an existing object in our database using the ID, and then we prepopulated the whole form using value attributes taken from that object.

Similarly, now we're using the object that we _attempted_ to save, to prepopulate the `with_errors` form. Interesting similarity. Can we exploit that?

Indeed! Copy-paste the contents of `app/views/movies/with_errors.html.erb` to `app/views/movies/new.html.erb`. Also, change the render statement in `MoviesController#create` on the "sad" branch to point to this template:

```ruby{8:(25-35)}
  # ...
  def create
    # ...
    if @the_movie.valid?
      @the_movie.save
      redirect_to("/movies", { :notice => "Movie created successfully." })
    else
      render template: "movies/new"
    end
  # ...
```

Try the new movie form again. Still working like before? Good, then we can delete the `with_errors.html.erb` file.

However, there's one issue. What if I refresh the `/movies/new` page by typing it into the address bar (i.e., by triggering the `MoviesController#new` action)?

We get an error `undefined method 'errors' for nil`. That's because `@the_movie` is not defined in our `MoviesController#new` action that gets triggered when a user visits `/movies/new`!

We can solve this with a little hack:

```ruby{3}
  # ...
  def new
    @the_movie = Movie.new
    render template: "movies/new"
  end
  # ...
```

We create the variable that the template wants, by just putting a blank brand new movie instance in it. We don't try to `.save` or call `.valid`, which means the validations don't get run. 

When we render this template using the `MoviesController#new` action _or_ using the `MoviesController#create` action, the instance variable `@the_movie` is available. When it is the empty `.new` instance, there are no errors to display and no values to prepoluate with, so those things in our form are just ignored.

You can't always use this hack. If the two forms diverge significantly, just make two templates and do the two different RCAVs to keep things simple. But, for many cases, this works well. 

The last thing we want to do here is to do the same thing for the `update` action: render a form with nice error messages and prepopulated fields from the object (which in this case already exists in our database). 

Can you do that on your own? 
