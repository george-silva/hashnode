## Design Patterns - Strategy

Hello, my friends! This is the second post in the  [series](https://dev.metageo.xyz/series/design-patterns) about _Design Patterns_ in `Python`.

So, first things first: what are design patterns? Design patterns are abstractions. The abstractions or concepts encompass a particular implementation that solves a known, frequent problem while developing software.

Compiling these abstractions and giving them proper names, makes it easier for us, as developers to communicate, express intent, and convey complex interactions between our code with a single name. Knowing these patterns will allow you to express yourself better with your fellow developers 💪.

Remember: design patterns are _recipes_. They are general and should serve as a guide.

Today we are going to talk about the Strategy Pattern.

## Strategy Pattern 

Like we did in our  [first post of the series](https://dev.metageo.xyz/design-patterns-visitor), let's start with a scenario. A concrete scenario helps us visualize how these patterns are useful.

I will reuse our first scenario - the `bookstore`. Our first problem was that we had to convert our objects - pure python objects to JSON representations, so we could integration our `Awesome Bookstore` to a big e-commerce player.

Let's recall our POPO (_Plain Old Python Objects_) - modeled as `dataclasses` for brevity.

```python
# models.py
from dataclasses import dataclass
from typing import List

@dataclass
class Customer:
    name: str
    status: str  # can be `normal` or `VIP
    # for brevity we are considering address to be the state where the customer lives
    # possible values: state-1, state-2 and state-3
    address: str

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
    weight: int # in grams
    # this of course is a simplification because prices are usually handled in Decimal    
    price: float  
```

Our problem today is that we need to calculate the final price for our customers, once they finalize the checkout in our store.

Depending on their status (customer status) - we want to give them discounts and apply different shipping fees.

Consider the following rules:

1. Customers can be of two types: `normal` and `VIP. `VIP` customers are indeed very important for us because they buy a lot from us. This type of customer gets 50% off in their shipping fees
2. Shipping fees depend on the state and weight of the book for any kind of customer. The prices for shipping per state and kilograms are listed below:

```python
# this per 1000 grams
STATE_SHIPPING_PRICES = {
    "state-1": 10,
    "state-2": 12,
    "state-3": 25
}
```

## Simple idea 1

To kick things off, we can do it the simplest way possible. Basically, check if the customer is VIP and calculate their shipping price differently. Should be straightforward.

Imagine we have some code, that given a `cart`, returns the total price of the shopping cart.

```python

def get_cart():
    # dummy function that retrieves a static shopping cart with some books
    george = Customer(name="George", status="normal", address="state-1")
    daniel = Author(name="Daniel Higginbotham", lives_in="USA")
    russ = Author(name="Russ Olsen", lives_in="USA")
    book1 = Book(
        name="Clojure for the Brave and True",
        author=[daniel],
        genre="Computer Science",
        weight=200,
        price=19.99)
    book2 = Book(
        name="Getting Clojure",
        author=[russ],
        genre="Computer Science",
        weight=200,
        price=19.99)
   return {"books": [book1, book2], "customer": george}
```

Using the function above, we can construct a list of books to be purchased and who is our customer. In this case - the customer, me, is a non-VIP customer living in `state-1`.

This means that the shipping fee is 10 `moneys` per kilogram. If I'm purchasing 2 books of 200g each, it means that we have 400g total weight. Doing quick math here we should charge me 4 `moneys` in shipping fees.

Let's implement that in the easiest way possible:

```python
def calculate_total(books, customer):

    total_book_price = sum([book.price for book in books])
    total_weight = sum([book.weight for book in books])
    total_shipping_price = total_weight * STATE_SHIPPING_PRICES[customer.address]
    
    if customer.is_vip:
        total_shipping_price = total_shipping_price * 0.5  # 50% discount
    return total_book_price + total_shipping_price
```

Python is really expressive, so it seems easy, huh?

The problem starts when there are a lot of variations for price calculations, which makes this hard to calculate and maintain. See the `if` clause? There is where our headache begins, when the product department decides to implement discounts at book prices for other types of customers, or if they introduce a new type of `super-vip` customer, that can get 100% discount at shipping prices. 

All of this starts to get messy _fast_.

That's where the strategy pattern is useful - implementing variations of the same _algorithm_ with different objects and contexts (keep in mind that Python has _ functions as first-class citiziens_, meaning you can pass functions around, simplifying how to implement the strategy pattern - I'll show both ways).

## Refactoring this into a strategy

A strategy is an object that has a single `execute` method and performs an algorithm - a series of steps, that might return or not return anything.

The only real constraint is that the parameters needed by this algorithm are common among different objects. 

So - let's imagine that we first refactor our code to split the logic between calculating totals and shipping fees in two functions:

```python
def calculate_shipping_total(books, customer):
    total_weight = sum([book.weight for book in books])
    total_shipping_price = total_weight * STATE_SHIPPING_PRICES[customer.address]
    
    if customer.is_vip:
        total_shipping_price = total_shipping_price * 0.5  # 50% discount
    return total_shipping_price

def calculate_total(books, customer):

    total_book_price = sum([book.price for book in books])
    total_weight = sum([book.weight for book in books])
    total_shipping_price = calculate_shipping_total(books, customer)
    return total_book_price + total_shipping_price
```

As you can see, a common _signature_ between the functions starts to emerge. To accurately calculate prices we need both the list of books being purchased and the customer, right?

In order to make this more flexible - let's implement a strategy, instead of having a hardcoded algorithm, like the one shown in `calculate_shipping_total`.

```python
class ShippingStrategy:
    """
    Object responsible to calculate the total price of shipping,
    given a customer and a list of books and their weights.
    """
    def execute(self, books, customer):
        raise NotImplementedError
```

This is our first strategy. The skeleton. Let's say that we have a default strategy, with no discounts and our discounted one.


```python
class DefaultShippingStrategy:
    """
    Object responsible to calculate the total price of shipping,
    given a customer and a list of books and their weights.
    """
    def execute(self, books, customer):
    
        total_weight = sum([book.weight for book in books])
        total_shipping_price = total_weight * STATE_SHIPPING_PRICES[customer.address]
        return total_shipping_price

class DiscountedShippingStrategy:
    def execute(self, books, customer):
    
        total_weight = sum([book.weight for book in books])
        total_shipping_price = total_weight * STATE_SHIPPING_PRICES[customer.address]
        return total_shipping_price * 0.5
```

## Tie it all together

Ok, cool, we have two strategies. But how can we effectively use it? We need a way to figure out which strategy to apply dynamically. And that is basically it.

```python
def which_strategy(customer):
    if customer.is_vip:
        return DiscountedShippingStrategy()
    return DefaultShippingStrategy()

def calculate_total(books, customer):
    shipping_strategy = which_strategy(customer)
    total_book_price = sum([book.price for book in books])
    total_weight = sum([book.weight for book in books])
    total_shipping_price = shipping_strategy.execute(books, customer)
    return total_book_price + total_shipping_price
```

See that instead of using multiple, complicated `if` statements in `calculate_total` function, we delegated all the shipping calculation responsibility to other objects, that are self-contained and execute only one thing.

The catch here is to abstract our different algorithms in different objects and using a method or a function to determine the strategy for us, given a set of parameters. 

Separating the **execution** of the strategy from its choice allows us to have methods with single responsibilities, that are easier to test, easier to extend and maintain.

With `Strategy` we can have several different forms of calculating shipping - without creating a monster function that has all the possible shipping variations you will find around.

## Functional bonus round

In Python, we have functions as _first class citizens_. That means we can pass around functions to functions. That means we don't need an extra object and an extra concept to wrap our shipping fee logic.

Instead, we can basically return functions from other functions and directly use them as needed. The same constraint applies: their signatures should be the same.

```python
def discounted_strategy(books, customer):
    # implement our code
    # ...
    return total_shipping * 0.5

def default_strategy(books, customer);
    # implement our code
    # ...
    return total_shipping
 
def which_strategy(customer):
    # this returns different functions based on our customers input
    if customer.is_vip:
        return discounted_strategy
    return default_strategy

def calculate_total(books, customer):
    shipping_strategy = which_strategy(customer)
    total_book_price = sum([book.price for book in books])
    total_weight = sum([book.weight for book in books])
    # directly execute the function return to us from `which_strategy`
    total_shipping_price = shipping_strategy(books, customer)
    return total_book_price + total_shipping_price
```

## Recap

1. Design patterns are abstractions and guidelines. Use them to communicate complex ideas with your fellow teammates effectively.
2. The Strategy Pattern allows us to execute different algorithms to solve a problem during runtime, based on a strategy selection method
3. Strategy decouples finding the method from its execution
4. The example gave in this post is _extremly_ simple and we might not even need a strategy pattern for this use case. Use your best sense to determine when these rules and choices frequently change or you want a clear separation of concern.

## Resources

* [Design Patterns: Elements of Reusable Object-Oriented Software](https://www.amazon.com/Design-Patterns-Object-Oriented-Addison-Wesley-Professional-ebook/dp/B000SEIBB8)
* [Strategy Pattern](https://refactoring.guru/design-patterns/strategy)