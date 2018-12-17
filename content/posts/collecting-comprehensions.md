+++ 
draft = true
date = 2018-12-16T09:15:32-08:00
title = ""
slug = "" 
tags = ["elixir"]
categories = ["elixir"]
thumbnail = "images/tn.png"
description = ""
+++

Comprehensions are an elegant way to loop through enumerables, and are especially powerful when the enumerable is heavily nested. Paired with the [Collectable](https://hexdocs.pm/elixir/Collectable.html) protocol they are absolutely fantastic.

Let's walk through doing just that.

```
  @data [
    %{
      1 => [
        %{
          1 => %{
            line_items: [
              %{
                id: 1,
                to_state: "NJ",
                unit_price: 101,
                quantity: 2
              },
              %{
                id: 1,
                to_state: "NJ",
                unit_price: 50,
                quantity: 2
              }
            ]
          }
        }
      ]
    },
    %{
      2 => [
        %{
          1 => %{
            line_items: [
              %{
                id: 1,
                to_state: "CA",
                unit_price: 101,
                quantity: 2
              },
              %{
                id: 2,
                to_state: "NJ",
                unit_price: 201,
                quantity: 3
              }
            ]
          }
        }
      ]
    }
  ]
```

Here is the data structure we are going to use.  We've got a list of users, who have lists of receipts, which each have a list of line items. Let's say we want to find number of line items who shipped to "NJ", total amount was > 100, and the total sum for all their amounts. 

Let's write a comprehension that collects all the line items that meet the above criteria

```elixir
    for user <- @data, # grab a single user
        {_user_id, receipts} <- user, # get that users receipts
        receipt <- receipts, # look at a single receipt
        {_receipt_id, %{line_items: line_items}} <- receipt, # get the line items from that receipt
        %{to_state: "NJ", unit_price: price, quantity: qty} when price * qty > 100 <- line_items, # filter
        do: (price * qty) #collect

        #=> [202, 603]
```

In just 6 lines of code we were able to iterate through all the users, all their receipts, all their line items and use a combination of pattern matching and guards to filter for the exact data we wanted.  Well not quite.  We still need to sum all the totals and count the number of line items we found.

We could Enum.reduce this returned data and get the last 2 answers we want but that comes with a couple negatives.  First we had to collect all this data into memory, and then we'd have traverse it all.  This isn't much with our small dataset but if we were working with something large we'd run into some speed and memory issues.  Really all we want to carry in memory are 2 data points, the total number or line items and the sum of them.

Now lets implement the Collectable that would help us avoid these 2 issues.

```
  defmodule Report do
    defstruct count: 0, total_sales: 0

    def update(report, total) do
      %{count: count, total_sales: total_sales} = report
      %{report | count: count + 1, total_sales: total_sales + total}
    end

    defimpl Collectable do
      def into(original) do
        collector_fun = fn
          report, {:cont, total} -> Report.update(report, total)
          report, :done -> report 
          _report, :halt -> :ok
        end
    
        {original, collector_fun}
      end
    end
  end
```

Here we've created a Report data structure to keep track of our count and total sales.  We've implemented the Collectable protocol by handling the 3 enumerable tags (more on these in a bit) it requires.  Finally we added a simple update function that will update the data points that we care about.  All thats left is to tell our comprehension to use this.

```
    for user <- @data, # grab a single user
        {_user_id, receipts} <- user, # get that users receipts
        receipt <- receipts, # look at a single receipt
        {_receipt_id, %{line_items: line_items}} <- receipt, # get the line items from that receipt
        %{to_state: "NJ", unit_price: price, quantity: qty} when price * qty > 100 <- line_items, # filter
        into: %Report{},
        do: (price * qty) #collect

        #=> %Sales.Report{count: 2, total_sales: 805} 
```

It's as simple as that!

### Enumerable Tags

Since enumerables are tagged tuples (:cont, :done, :halt, :suspend) when you implement different enum protocols you'll often need to handle these tags.

For collectable we need to handle :cont, :done and :halt.  For :cont we need to handle the next element being enumerated. :done is when we've reach the end and return our final accumulator.  But what about :halt?

Halt comes in handy when you need to terminate earlier than you would, ie in the middle of a list. Lets look at an example

First we'll create a file to read named `sample`

```
foo
bar
break
baz
```

Now lets create a stream that reads this file line by line and then using a comprehension puts those lines into a bitstring.

```elixir
# Resource
stream =
  Stream.resource(fn -> File.open!("sample") end,
                 fn file ->
                   case IO.read(file, :line) do
                    data when is_binary(data) and data == "break\n" -> {:halt, file}
                    data when is_binary(data) -> {[data], file}
                     _ -> {:halt, file}
                   end
                 end,
                 fn file -> File.close(file) end)

for x <- stream, into: <<>>, do: x
#=> "foo\nbar\n" 
```

As you can see from the output the bitstring implementation of Collectable halted our stream early because of the `:halt` tag and its match on the "break" line in our file.

So we've used a comprehension to loop through a heavily nested data structure and passed the results into our own collectable to make easy work of getting the exact answers we were looking for.  Until next time!
