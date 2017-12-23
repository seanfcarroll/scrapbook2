# Write more explicit code layers !

> I'll demonstrate a Ruby on Rails example here, but the rules really apply to
> any programming language and any web framework.

Imagine you have a Rails controller that is creating a Book record

```ruby
# app/controllers/book_controller.rb
class BookController
  def create
    # ...
    @book = Book.create(params)
    # ...
  end
end
```

Tell me from looking at this code what data will be saved?

Well obviously it's a `Book` model so it will be some data related to
book right ?

But a `book` may mean different things in different applications:

* are we saving comprehensive book details (like in library)?
* are we saving some book price ? (like in book store?)
* maybe we are `book`ing  a ticket to concert

Ok lets have a look at the book model in Ruby on Rails.

```ruby
# app/models/book.rb
class Book < ActiveRecord::Base
  belongs_to :author
end
```

Hmm, I see there is an author, but I still cannot say what data are
saved related to book. We could run the `rails c` and inspect the `Book` model definition:


```ruby
Book.attribute_names
```

...but the only way to determine this directly from code is to look at **database schema** /
migrations definition or to look at **web interface**

Ok no big deal let's just have a look at `app/views/books/create.html.erb`

> If you never worked with Ruby on Rails, this is where the HTML code is stored. So
> we are trying to look at the HTML form definition and data that the
> form is sending.

Oh! I've forgot to tell you this RoR application is just an JSON API
and the Frontend code interface definition is not rendered with this
Rails app, but rather separate codebase NodeJS single page app (SPA).

> Yes Rails can render the single page frontend. But some companies
> likes to split into FE and BE teams working on separate codebase & separate technologies so that 
> everyone can implement their own favorite solutions. Teams can then
> deploy to different servers and the FE SPA is just communicating with
> BE APP via JSON API


So you need to go to other Github repository and check what the Frontend
is sending from there. And if you are not familiar with JS frameworks
then good luck.

Ok maybe your team is not split in such way or you are sending data from
HTML form that is rendered from `app/views`. But now you discovered that
that endpoint is actually an API that is called by your company partner,
or the app is a microservice.

**The point is that in these scenarios the Rails application is a separate
layer, an isolated bubble. There is a brick wall between your BE app
and something else. You cannot just throw garbage of values wrapped in a plastic bag
called `params` and push it directly to [ActiveRecord](http://guides.rubyonrails.org/active_record_basics.html) to save in DB. There needs to be an Explicit Interface defined!**

So, did you spotted a security smell in my code?  And yes the solution for
this security smell will solve the explicit interface issue: I need to restrict permitted
parameters for this resource otherwise someone may set undesired values
(E.g. `book['author']['admin']=true`) :

```ruby
# app/controllers/book_controller.rb
class BookController < ApplicationController
  def create
    # ...
    @book = Book.create(book_params)
    # ...
  end

  def book_params
     params.require(:book).permit(:title, :author_id, :price)
  end
end
```

This way we are allowing only to set `book.title`, `book.author_id` and
`book.price` and from this we can determine what the Book model is
really about.

Cool, we have an existing solution for this. But imagine you are
building complex search solution that is too big to be in a controller.

You can search `term` and apply several filters like:

* search `term`:  e.g. "ruby" as in "Programming Ruby"
* `publisher`:  e.g.: The Pragmatic Bookshelf => 'pragprog'
* `format`: `paper`, `hard` or `ebook`
* `favorite`: `true/false`
* ...and many more

e.g:

`GET localhost:3000/books?term=ruby&publisher=pragprog&favorite=true&format=ebook`

So you decide to use Service object (or Procedure Module):


```ruby
# app/controllers/book_controller.rb
class BookController < ApplicationController

  def index
    # ...
    @results = BookSearcher.find(params)
    # ...
  end
end
```


Similar to previous example, tell me from this code what the `BookSearcher`
is searching for ? You don't know do you ? Why because that
responsibility is passed to Searcher. Let's investigate that.


> We will use `ActiveRecord` example but the  BookSearcher search can be on ElasticSearch, DynamoDB, ... That's why we don't convert it to [Rails Query Object](http://www.eq8.eu/blogs/38-rails-activerecord-relation-arel-composition-and-query-objects)
> ...


```ruby
# app/services/book_searcher.rb
module BookSearcher
  extend self

  def find(params)
    scope = Book.all
    scope = scope.where("books.title LIKE ?", "%#{params['terms']}%") if params['terms']
    scope = scope.where('? = ANY (book_format)', params['format']) if params['format']

    if params['publisher'] && publisher = PublisherSearch.find(params)
      # do some search on publisher keyword and add condition to `scope`
    end

    # ...

    scope
  end
end

# app/services/book_searcher.rb
module BookSearcher
  extend self

  def find(params)
    publisher_keyword = params[:publisher]
    # ...it dosn't really matter what it does with `publisher_keyword`
  end
end
```


So the way how this works is that `BooksController` will just pass all
the `params` to `BookSearcher.find(params)` depending on the values of
`params` we construct various SQL conditions directly on `Book` model
`ActiveRecord::Relation` scope
but when we need to do some search on `Publisher` keyword we just pass
the `params` to `PublisherSearch` which then takes value of
`params[:publisher]` to do some generic search used is some other place.

Let's just assume that `PublisherSearch` is used in some other controller
and we just want to reuse it.


> Wait a minute, `BookSearcher` is not a service object !
>
> Recently Avdi Grimm published article [Enough with service objects](https://avdi.codes/service-objects/) where he argues that Object existence needs
> needs to be justified with receiver of message.
>
> I have problem with unjustified objects  as well (and therefore service objects) but in terms
> of unnecessary state holding that I've explained 
in my article [Lesson learned after trying functional programming](http://www.eq8.eu/blogs/46-lesson-learned-after-trying-functional-programming-as-a-ruby-developer)
>
> Normally I would just write Service object example as developers are
> more familiar with them and I want to prove different point
> but it's time to start separating `procedures` (transaction script)
> and `objects` as different thing.


Now let me rewrite the code differently


```ruby
# app/controllers/book_controller.rb
class BookController < ApplicationController

  def index
    # ...
    @results = BookSearcher.find({
      term: params[:term].blank?` ? nil : params[:term],
      format: params[:format],
      publisher_keyword: params[:publisher]
      # ...
    })
    # ...
  end
end
```

Now before we continue to Service solution, tell me by just reading
this controller code what the `BookSearcher` could be searching for ?

```ruby
# app/services/book_searcher.rb
module BookSearcher
  extend self

  def find(term:, format:, publisher_keyword: )
    scope = Book.all
    scope = scope.where("books.title LIKE ?", "%#{term}%") if term
    scope = scope.where('? = ANY (book_format)', format) if format

    if publisher_keyword && publisher = PublisherSearch.find(publisher_keyword: publisher_keyword)
      # do some search on publisher keyword and add condition to `scope`
    end

    # ...

    scope
  end
end

# app/services/book_searcher.rb
module BookSearcher
  extend self

  def find(publisher_keyword:)
    # ...it dosn't really matter what it does with `publisher_keyword`
  end
end
```

It may not seems much but the difference is huge !

You don't pass `params` wildly to rest of your application, you clearly
define what you will pass to rest of your code.

By being more explicit we clearly distinguish
or controller to be a layer that holds responsibility of forwarding
well defined values.

### Service example

Let say there is a good reason to satisfy `BookSearcher` to be a Service Object.
One scenario could be that you created platform wher multiple companies
can host their online book store.

This way the service object is initialized based on `current_store`
(current client) and search options `filters` serves as modifiers:


```ruby
# app/controllers/book_controller.rb
class BookController < ApplicationController

  def index
    # ...
    current_store = Store.find_by(session[:store_id])

    BookSearcher
      .new(store: current_store)
      .tap do |s|
        s.term   = params[:term]   unless params[:term].blank?
        s.format = params[:format]
        # ...
    })
    # ...
  end
end
```

As you can see we didn't loose the explicitness, we still can determine
what the search logic will be from here.

```ruby
# app/services/book_searcher.rb
class BookSearcher
  attr_reader :store
  attr_accessor :term:, :publisher_keyword
  attr_writter :format

  def initialize(store)
    @store
  end

  def find
    scope = store.books
    scope = scope.where("books.title LIKE ?", "%#{term}%") if term
    scope = scope.where('? = ANY (book_format)', format) if format
    # ...
    scope
  end

  private
    def format
      store.supported_format?(format: @format) if @format
    end
end
```

### Request model

At this point you may say what we've lost the agility of the code. If we
want to introduce a new search parameter all we needed to do in our
"pass the `params` to service" example was just to add the logic in the
service. This way we need to update the Controller and any other
method/object dealing with this value.

Some may say that this is silly, all we need to do is to write a test on
how our service will behave when we pass it `params` in different forms.

Well all this is true.

Ever heard of **Request models**  also called **Request Objects** ?

They are type of objects that are just passed through your Controller
directly to Service object/s or Procedure module/s and you can test
different scenarios with them.

The catch is however that they need to be well defined and plain `params` is not it! (it's too dynamic)

> Same is with constructing `OpenStruct.new` objects passed to other
> services

The problem with `params` is that it represents anything passed via
browser or JSON API. But limit it to only the stuff your app really
need e.g. `params.require(:book).permit(:title, :author_id, :price)`
and you got yourself a Request Model !

But there is another gotcha you just don't write tests on your Service
object when passing different Request Models, you also write tests on
different possibilities of Request Models itself !

I've described [Request Objects](http://www.eq8.eu/blogs/22-different-ways-how-to-do-contextual-rails-validations) bit further in [this article](http://www.eq8.eu/blogs/22-different-ways-how-to-do-contextual-rails-validations) on topic of contextual validations in Rails. I'll write entire article on them someday in future therefore I'll
not go into too much details but just to show you what I mean:


```ruby
# app/controllers/book_controller.rb
class BookController < ApplicationController

  def index
    request_object BookSearchRequest.new(params)
    @result = BookSearcher.find(request_object)
  end
end
```

```ruby
# app/services/book_searcher.rb
module BookSearcher
  extend self

  def find(request_object)
    scope = Book.all
    scope = scope.where("books.title LIKE ?", "%#{request_object.term}%") if request_object.term
    scope = scope.where('? = ANY (book_format)', request_object.format) if request_object.format

    if publisher = PublisherSearch.find(request_object)
      # ...
    end

    # ...

    scope
  end
end
```


```ruby
# app/request_models/book_searcher_request.rb
class BookSearchRequest
  attr_reader :params

  def initialize(params)
    @params = params
  end

  def term
    params[:term] unless params[:term].blank?
  end

  # ....

  def publisher
    PublisherRequest.new(params)
  end
end

# app/request_models/book_searcher_request.rb
class PublisherSearchRequest
  attr_reader :params

  def initialize(params)
    @params = params
  end

  def publisher_keyword
    params[:publisher]
  end
end

#app/service/publisher_search
class PublisherSearch
  extend :self

  def find(request_models)
    publisher_keyword = request_models.publisher_keyword
    # ....
  end
end
```

Plus  you need to write tests on when you pass given request model to
various Procedure modules and cross-test various request models so that
they are contract and any change to them is explicit to every aspect of
application where they are used.

This way no longer the Controller is explicit definition of your
endpoint but Request model.

Let me just highlight that Request Models are really rare in my code. I
need to deal with really complex request (e.g. bulk update) to implement
them. I rather prefer explicit controller passing of arguments as
dynamic nature of Request Models comes with price of overcomplication.