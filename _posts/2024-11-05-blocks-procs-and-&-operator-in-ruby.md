---
title: "Blocks, Procs, and & operator in Ruby"
date: 2024-11-05 23:10:00 +0530
categories: [Ruby]
tags: [ruby]
render_with_liquid: false
---

Blocks, procs, and **&** operator are something that's used by Ruby developers daily. While used along with enumerators, we see cool statements like `["ant", "bat"].map(&:upcase)`. Let's peek behind the abstractions and try to understand what happens behind.

## Block

Block is a chunk of code enclosed in curly braces or `do...end` that you can pass to a method.

```ruby
def say_something
  yield if block_given?
end

say_something do
  puts "hello"
end                  # >> hello

say_something { puts "hello" } # >> hello

say_something # => nil
```

The block passed into the method is executed when the method uses the `yield` keyword. You might have guessed that `block_given?` method checks if a block was passed during method call. `yield` raises an error if no block is passed.

Some points to note are:
- A method can only accept one block.
- `yield` can be used multiple times in the method.

## Proc

Blocks cannot be stored in a variable directly because they are not objects. One way to store it is by converting it to a `Proc` object.

```ruby
my_proc = Proc.new { |x| puts "hello #{x}" }
my_proc.call(5) # >> hello 5

# A more concise way to create a new proc object is

my_proc = proc { |x| puts "hello #{x}"}
my_proc.call(5) # >> hello 5
```

We use the `call` method on `my_proc` to execute its code block. Any arguments passed in to the `call` method are passed as arguments to the block.

Procs are like any other Ruby objects and can be passed on to methods as arguments. There are no restrictions on the number of procs that can be passed to a method.

```ruby
def say_something(english_hello, hindi_hello)
  english_hello.call
  hindi_hello.call
end

english_hello = proc { puts "hello" }
hindi_hello = proc { puts "namaste" }

say_something(english_hello, hindi_hello)

# OUTPUT
# >> hello
# >> namaste
```

## Conversion between Procs and Blocks

### Block to Proc

Another way to use a block in a method is to convert the incoming block to a proc and store it in a variable to use later. This is exactly what **&** operator does. Let's see how.

```ruby
def say_something(&speak)
  speak.call
end

say_something { puts "hello" } # >> hello
```
**&** operator converts the block passed into the `say_something` method into a proc and stores it in the `speak` variable.

> Note: `block.call` used to create a Proc object, but since Ruby 2.6 it has been optimized to not create a Proc object thus enhancing the performance.
{: .prompt-info }

### Proc to Block

Sometimes we may have a proc object with use that we would like to pass into a method that accepts a block. The same **&** operator comes to the rescue here. Let's see how.

```ruby
def say_something
  yield if block_given?
end

my_proc = proc { puts "hello" }

say_something(&my_proc) # >> hello
```

**&** operator converts the `Proc` object `my_proc` into a block which is accepted by the `say_something` method.

Now let's see an example where both conversions take place

```ruby
def say_something(&speak)
  speak.call
end

my_proc = proc { puts "hello" }

say_something(&my_proc) # >> hello
```

Here, a `Proc` object `my_proc` is converted to a block and passed to `say_something` method, which accepts the block, converts it to proc and store it in the `speak` variable.

One point to note is that **&** only works within the context of a method call or method definition.

`each` and `map` are enumerators that accept a block. Let's try to use a **&** and a proc object to replicate the functionality.

```ruby
numbers = [1, 2, 3]

# Double the numbers in the array
numbers.map { |x| x * 2 } # => [2, 4, 6]

doubler = proc { |x| x * 2 }
numbers.map(&doubler) # => [2, 4, 6]
```

## to_proc

We understood that **&** converts Proc to Block and vice versa. But that doesn't explain how this statement works.

```ruby
["ant", "bat"].map(&:upcase) # => ["ANT", "BAT"]
```

Here we are calling **&** operator on a `Symbol` object, but how can a `Symbol` object be converted to a block ?. To understand this, we need to dig a bit deeper and understand what **&** does under the hood.

Before converting to a block, **&** operator calls `to_proc` method on the object, in our case we have a `Symbol` object. `Symbol` class defines a `to_proc` method in it. Here's a sample implementation of the `to_proc` method in the `Symbol` class to get an idea of what it does on the background.

> Note: The original implementation of this is in C, this is a sample re-implementation for understanding purposes.
{: .prompt-info }

```ruby
class Symbol
  # STUFF...
  def to_proc
    proc { |obj, *args, **kwargs, &block| obj.public_send(self, *args, **kwargs, &block) } # self here represents the Symbol object (:upcase)
  end
end
```

So, in `["ant", "bat"].map(&:upcase)`, **&** operator calls the `to_proc` method defined in `Symbol` class which returns a Proc object which is then converted into a block for passing into the `map` method which accepts a block. `map` passes each of the strings from the array into the Block which calls the `public_send` method on the string with `:upcase` as the argument, which dynamically invokes the `upcase` method of the string. `map` just passes the element to the block, so `args`, `kwargs`, and `block` are not present for this example.

Now you might think, if **&** calls the `to_proc` method, then it must be defined in the Proc object too. Yes, we have the `to_proc` method defined for Proc objects as well. In the case of Procs `to_proc` method returns itself as it is already a proc and there is no need for a conversion.

```ruby
class Proc
  # STUFF...
  def to_proc
    self
  end
end
```

There is one more class in Ruby that has a `to_proc` method defined on it. The `Method` class. Let's see **&** operator in action with a `Method` object.

```ruby
def append_hello(string)
  "hello #{string}!"
end

method_object = method(:append_hello) # Creates a Method object from the method

["ant", "bat"].map(&method_object) # => ["hello ant!", "hello bat!"]
```

Sample implementation of `to_proc` method in the `Method` class.

```ruby
class Method
  # STUFF...
  def to_proc
    proc { |*args, **kwargs, &block| self.call(*args, **kwargs, &block) }
  end
end
```

If you want to support **&** operator on your custom Ruby class, then all you have to do is to define a `to_proc` method that gives back a Proc object that is relevant. You can also modify existing Ruby classes to support **&** operator by defining a `to_proc` method.

Let's modify the `String` class in Ruby to support statements like `["ant", "bat"].map(&"upcase")`. Currently, the `String` class doesn't define a `to_proc` method in it, let's reopen the class and define one.

```ruby
class String
  def to_proc
    to_sym.to_proc
  end
end

["ant", "bat"].map(&"upcase") # => ["ANT", "BAT"]

method_name = "capitalize"

["ant", "bat"].map(&method_name) # => ["Ant", "Bat"]
```

Another gotcha is that the **&** doesn't only support literals and variables, but it can also be used with expressions that return an object that supports `to_proc` method.

```ruby
["ant", "bat"].map(&("up" + "case").to_sym) # => ["ANT", "BAT"]
```

## Conclusion

In this post, we've explored the concepts of blocks, procs, and the **&** operator in Ruby, explaining how they work and interact with one another. Blocks are an integral part of Ruby and offer a flexible way to pass chunks of code to methods. However, blocks are not objects, and thus, we rely on Procs to treat them as first-class objects.

The **&** operator is the bridge between blocks and procs, converting one into the other when necessary. We also learned how **&** works with Symbol, Proc, and Method objects by invoking the `to_proc` method, enabling powerful and concise enumerations like `array.map(&:upcase)`.

The `to_proc` method provides an elegant mechanism for transforming objects into callable procs, and with this understanding, you can now explore extending this functionality to your custom classes. By providing a `to_proc` method in your own classes, you can leverage the & operator in creative and useful ways, further embracing Ruby’s expressive syntax.

This exploration opens up new possibilities for you to write more concise and flexible Ruby code, especially when dealing with enumerable methods and functional patterns.

## Sources
- [Programming Ruby 3.3: The Pragmatic Programmers Guide](https://pragprog.com/titles/ruby5/programming-ruby-3-3-5th-edition/)
