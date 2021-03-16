---
title: How to make your Python code maintainable with the open/close principle
description: >-
  How to avoid code spaghetti? How to refactor our if-else hell to something
  easier to read, painless to extend, and fun to work with.
date: '2020-09-07T16:51:41.545Z'
categories: []
keywords: []
slug: >-
  how-to-make-your-python-code-maintainable-with-the-open-close-principle
---

How to avoid code spaghetti? How to refactor our if-else hell to something easier to read, painless to extend, and fun to work with.

The second principle of SOLID says

> “Software entities (classes, modules, functions, etc.) should be open for extension, but closed for modification.”

Let’s say we have to write code responsible for generating offers for our clients based on the order they made. There are a few conditions that have to be considered. Different types of clients need different logic. In very basic implementation this may look something like that:

{% highlight python %}

def generate_offer(order: Order) -> Offer:  
    offer = Offer(order.client)  
  
    for product in order.get_ordered_products():  
        product_price = product.price  
  
        if order.client.is_regular:  
            # imagine more complex logic here  
            if product.quantity > 10:  
                discount = 0.9  # complicated logic  
                product_price = product_price * discount  
            elif product.quantity > 5:  
                product_price = product_price * 0.95  
        elif order.client.is_vip:  
            discount = 0.9  # complicated logic  
            product_price = product_price * discount  
        else:  
            if product.quantity > 15:  
                # imagine more complex logic here  
                product_price = product_price * 0.05  
  
        offer.addItem(product, product_price)  
  
    return offer
{% endhighlight %}

This is a fairly simple example but will give us what we need. In real projects, we may have much more complicated logic and conditions could change based on a weekday or the number of products in the warehouse. Now imagine that this happened and we actually have not 20 but 8 thousand lines of code. How would you refactor code like that?

### **What should we look for?**

Very often code can be split into 3 parts. We can look for parts that answer one of those 3 questions:

![](/assets/1__WOpSKX5h2e8kPhu4aru34A.png)


**WHAT** changes

This is a stable part of our code. It shouldn’t change much in the future. In this case, calculating the offer looks like something really steady.

{% highlight python %}

    # this code doesn't change  
    offer = Offer(order.client)  
  
    for product in order.get_ordered_products():  
  
        offer.addItem(product, product_price)  
  
    return offer
{% endhighlight %}

**HOW** it changes

In this part, we will know how the discount is calculated. All complex operations will be handled here.

{% highlight python %}

if product.quantity > 10:  
    discount = 0.9  # complicated logic  
    product_price = product_price * discount  
elif product.quantity > 5:  
    product_price = product_price * 0.95
{% endhighlight %}

**WHY** it changes

How to make a decision about choosing **HOW**? Seems like the main reason to change our discount policy depends on the client type.

{% highlight python %}

if order.client.is_regular:  
    # stuff             
elif order.client.is_vip:  
    # stuff  
else:  
    # stuff
{% endhighlight %}

**Object-Oriented Approach**
{% highlight python %}

# HOW  
class DiscountPolicy(ABC):  
    @abstractmethod  
    def calculate(self, product: Product) -> Price:  
        pass

class VIPClientDiscountPolicy(DiscountPolicy):  
    def calculate(self, product: Product) -> Price:  
        # complex logic  
        return

class RegularClientDiscountPolicy(DiscountPolicy):  
    def calculate(self, product: Product) -> Price:  
        # complex logic  
        return  
  
  
class NewClientDiscountPolicy(DiscountPolicy):  
    def calculate(self, product: Product) -> Price:  
        # complex logic  
        return  
 
  
# WHY  

class BidderFactory:  
    def create(self, client, *args) -> Bidder:  
        if client.is_vip:  
            return Bidder(VIPClientDiscountPolicy())

        if client.is_regular:  
            return Bidder(RegularClientDiscountPolicy())

        return Bidder(NewClientDiscountPolicy())  
  
  
# WHAT is stable. It shouldn't change in the feature.  
class Bidder:  
    def __init__(self, discount_policy: DiscountPolicy):  
        self.discount_policy = discount_policy  
  
    def generate_offer(self, order: Order) -> Offer:  
        offer = Offer(client)  
  
        for product in order.get_ordered_products():  
            offer.add_item(product, self.discount_policy.calculate(product))  
  
        return offer
{% endhighlight %}

Now our code is really close for modification and open for extension. We can easily add new _DiscountPolicy_ thanks to the [strategy pattern](https://en.wikipedia.org/wiki/Strategy_pattern) with a different, more complex implementation that answers the question of **how** price should be calculated.

By using a factory pattern we can easily decide **why** or when our logic should be different.

This leaves us with the “**what?”** question. Very simple _Bidder_ class that exposes the behavior with meaningful names.

Notice that previously isolated building blocks can be now tested separately or could be even created by different teams. The hardest thing to come up with is a common interface, resistant to changes.

The same design can be accomplished without OOP. In this example, we have a function responsible for providing another function that calculates discounts for our clients.

{% highlight python %}

def vip_client_discount_calculator(product):  
    return

def new_client_discount_calculator(product):  
    return

def regular_client_discount_calculator(product):  
    return

def discount_calculator_factory(client):  
    if client.is_vip:  
        return vip_client_discount_calculator  
    return new_client_discount_calculator  
  
  
def generate_offer(  
    order: Order,  
    discount_calculator=new_client_discount_calculator  
) -> Offer:  
    offer = Offer(client)  
  
    for product in order.get_ordered_products():  
        offer.add_item(product, discount_calculator(product))  
  
    return offer  
  
  
generate_offer(order, discount_calculator_factory(client))
{% endhighlight %}
