## Design Patterns - Visitor

# Welcome 🚀

First of all, welcome! This is the first post in a series of posts about design patterns. Today we will talk about the visitor pattern, a simple abstraction that will help you decouple your domain objects from application objects and keep your code clean.

But what are design patterns?

Design patterns are merely abstractions of a set of ideas. They have a name, a type, and a general implementation - slightly different depending on each context. 

Abstracting these _ideas_ in a vocabulary and giving them proper names, allows us to refer to them and communicate these complex ideas with ease.

There are several design patterns used in object-oriented programming languages that help us to solve common problems while designing and writing software and are a powerful tool in every software developer toolbox.

Not all patterns apply to all languages and all paradigms. Sometimes a pattern will not make any sense in a dynamically typed language, but it will make a lot of sense in a statically typed language.

These _ideas_, or _recipes_ can be applied to different contexts and suffer variations, but use this as a guide. 

# Visitor Pattern 🧑‍🚀👩‍🚀

The visitor pattern is a behavioral pattern that uses a technique called double dispatch to simplify your software object hierarchies and keep implementation details of processes, separate from your domain logic.

From [Wikipedia](https://en.wikipedia.org/wiki/Double_dispatch), Double Dispatch is a technique to pass messages between methods, with varying arguments and types.

Let's imagine a scenario, for a more concrete view:

1. We have a collection of different object types - let's say a bookstore. Our bookstore sells different types of items: `books`, `magazines`, `articles` and `t-shirts`.
2. Each object has a different set of attributes and they may or may not be related, or composed.
3. We want to integrate our bookstore with a large e-commerce platform, but that requires us to convert our data structures, from their internal Python representation to a JSON representation to be sent to our partner e-commerce.

Sounds pretty easy, right?

Let's get started with would be a barebones implementation of our cool bookstore in `Python` (this example uses `dataclasses` because I don't want to complicate it with framework details):

```python
# models.py
from dataclasses import dataclass
from typing import List


@dataclass
class Author:
    name: str
    lives_in: str

@dataclass
class Publisher:
    name: str
    company_number: str

@dataclass
class Book:
    title: str
    author: List[Author]
    genre: str

@dataclass
class Magazine:
    title: str
    publisher: Publisher
    issue: int

@dataclass
class Article:
    title: str
    authors: List[Author]
    abstract: str
    full_text: str
```

To solve our problem, we need to be able to convert our `models` for our API integration to send this information to our e-commerce partner. There are some ways of achieving this, but the visitor pattern is a way of achieving this with very little modification to our business objects and avoiding coupling our algorithm to `convert my data to json` separate from the domain objects.

## Simple idea #1

The easiest thing we can do to solve our problem is to implement a generic `to_json` method to each of our classes and be done with it. Like so:

```python
# models.py
# ... 

@dataclass
class Author:
    name: str
    lives_in: str

    def to_json(self):
        return {"name": self.name, "lives_in": self.lives_in}

@dataclass
class Publisher:
    name: str
    company_number: str

    def to_json(self):
        return {"name": self.name, "company_number": self.company_number}

# ... rest of the file
```

While this take is not a bad idea, it suffers from the following problems:

1. Our business model has to know about integration formats. Our first e-commerce partner accepts a specific payload and a format (JSON). But what if we need to support multiple partners?
2. Our business model know is covered with implementation details of things that are not crucial to these objects. They should be able to exist, without these details.

Ok, let's refactor this approach in a second object! It will be clearer and decoupled!

## Separate converter object

In this approach, we take the details of each conversion and stick it into a new converter object.

```python
class ECommerceExporter:

    def _book_to_json(self, item):
        authors = [self.to_json(author) for author in item.authors]
        return {"name": self.name, "authors": authors, "genre": item.genre}

    def _author_to_json(self, item):
        return {"name": item.name, "lives_in": item.lives_in}

    def to_json(self, item):
        if isinstance(item, Book):
            return self._book_to_json(item)
        if isinstance(item, Author):
            return self._author_to_json(item)
        # ... all the other objects
```

While the following implementation is somewhat naive (you can use a `dict` like structure to simplify the `to_json` method) - this is hard to maintain. And this will only work for this specific use case. If you have a different `e-commerce` partner, will you need to repeat the `to_json` code again and again. For large object hierarchies, this will be difficult.

We can DRY (_do not repeat yourself_) a bit and completely separate the `algorithm` (JSON conversion) from our objects, allowing us even to create different algorithms and keep supporting these.

## Visitor Pattern to the rescue

So, what is this all about? The idea behind the `Visitor Pattern` is to implement a technique called `Double Dispatching`, decoupling the identification of a type and the method it calls from *another object*. Weird, but effective.

The first thing we need to do for this to work is to create a basic, generic `Visitor`. This object will determine the required interface for all algorithms or processes to operate on. 

```python
class BookstoreVisitor:

    def visit_book(self, item):
        raise NotImplementedError
    def visit_author(self, item):
        raise NotImplementedError
    def visit_magazine(self, item):
        raise NotImplementedError
```

We want that all `Bookstore` visitors to follow the above pattern. Each visitor decides what to do when it visits one object and that it's only responsibility. 

Our converted would be the following:

```python
class ECommerceVisitor:

    def visit_book(self, item):
        authors = [self.visit_author(author) for author in item.authors]
        return {"name": item.name, "authors": authors, "genre": item.genre}

    def visit_author(self, item):
        return {"name": item.name, "lives_in": item.lives_in}
```

Ok - so far so good. But how this connects with our business model objects? We need to make a slight change in them, for them to support _any visitor_. It's general application logic that can be reused for any purpose, it's not tied to a specific feature or requirement and easy to maintain.

Let's implement our `accept` method that receives a visitor. This is the catch. The accept method is the `double dispatch`, where we determine implementation based on the input arguments (the visitor itself) and the type of the object it's calling the visitor.

```python
# models.py
from dataclasses import dataclass
from typing import List

@dataclass
class Author:
    name: str
    lives_in: str

    def accept(self, visitor):
        return visitor.visit_author(self)

@dataclass
class Book:
    title: str
    author: List[Author]
    genre: str

    def accept(self, visitor):
        return visitor.visit_book(self)
```

With this implementation, we could create many types of visitors to operate on collections of different objects and perform different work within each.

If a new requirement comes, asking us to export this to another format - let's say `XML`, all we need to do is create a separate `XMLVisitor`.

## Tie all together 👍

Now, this is all good and fine, but how do I use this? Let's imagine that our application has a cronjob that sends all the data we have to our e-commerce partner.

```python
import json
def get_data():
    author1 = Author(name="george", lives_in="Floripa")
    author2 = Author(name="Dude #1", lives_in"Anywhere")
    book1 = Book(name="Design Patterns are Cool", authors=[author1, author2], genre="Software Development")
    article1 = Article(name="study about stuff", authors=[author2], abstract="xyz", full_text="this is my new cool article")
    return [book1, article1]

def integrate_with_ecommerce(data):
    visitor = ECommerceVisitor()
    result = [item.accept(visitor) for item in data]
    response = requests.post("https://large-ecommerce.xyz/create", data=json.dumps(result))
```

This is very useful to specialize behavior in complex large object-oriented hierarchies. Imagine that in the following case, it was one URL per object type. It would still be doable, with a new visitor and delegating to the visitor the call to each URL.

## Recap 🔁

1. Design patterns are abstractions. They have names so we can communicate these abstractions easily with others (_hey man, let's just implement another visitor and that ticket product wants for tomorrow is done!_).
2. The visitor pattern allows us to separate algorithmic / process implementations from our precious and critical business objects.
3. Double dispatch is weird - `item` calls a method using a visitor as a parameter and that visitor is called from within the object, passing self as an argument. Two method calls to one another.

## Resources 📚

There were used several resources in writing this:

* [Design Patterns: Elements of Reusable Object-Oriented Software](https://www.amazon.com/Design-Patterns-Object-Oriented-Addison-Wesley-Professional-ebook/dp/B000SEIBB8)
* [Refactoring Guru](https://refactoring.guru/design-patterns/visitor)

