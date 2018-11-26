+++ 
draft = false
date = 2018-11-25T13:58:02-08:00
title = "How to Behave and Why"
slug = "how-to-behave-and-why" 
tags = ["elixir"]
categories = ["elixir"]
thumbnail = "images/tn.png"
description = ""
+++

Using [behaviours](https://hexdocs.pm/elixir/master/Behaviour.html) in elixir is a great way to define how a module act. They are also used a lot when doing polymorphism, but there are a few best practices that are not apparent when taking this approach. Lets go over them.

Lets implement a simple behaviour to give us some context.

```elixir
defmodule Calculator do
  @callback add(number(), number()) :: number()
end

defmodule Calc.Base10 do
  @behaviour Calculator

  @impl true
  def add(x, y), do: x + y
end
```

We've created a simple behaviour and implemented its only callback; the add function.  Now what if we want to create a default implementation for this function? This is where we can get ourselves into a bit of trouble. Lets do that first the naive way.

```elixir
defmodule Calculator do
  @callback add(number(), number()) :: number()

  defmacro __using__(_opts) do
    quote do
      @behaviour Calculator

      def add(x, y), do: x + y
    end
  end

end

defmodule Calc.Base10 do
  use Calculator
end

Calc.Base10.add(1, 2) #=> 3
```

Implementing it this way comes with a few problems. Namely slower compile times and harder to trace bugs. Lets take a look at the latter.

```
iex> Calc.Base10.add(1, "2")    
** (ArithmeticError) bad argument in arithmetic expression
    iex:21: Calc.Base10.add/2
```

The problem with this is the error location. Imagine you have a complex module and this error message is telling you there is a problem on line 21.  Tracking that error down is going to be harder than it has to be.  Let's take a look at a way we can avoid this.


```elixir
defmodule Calculator do
  @callback add(number(), number()) :: number()

  defmacro __using__(_opts) do
    quote do
      @behaviour Calculator

      def add(x, y), do: Calculator.add(x, y)
    end
  end

  def add(x, y), do: x + y
end

defmodule Calc.Base10 do
  use Calculator
end
```

Now that we've delegated the function to the Calculator module let's see take a look at that error message again.

```
iex> Calc.Base10.add(1, "2")
** (ArithmeticError) bad argument in arithmetic expression
    iex:34: Calculator.add/2
```

Now we can go right to where this function is implemented and start debugging. For a none contrived example you can see that [EEx.Engine](https://github.com/elixir-lang/elixir/blob/v1.7.4/lib/eex/lib/eex/engine.ex) uses this same pattern.

Behaviours might not always be the right choice when it comes to mixing your code.  Sometimes just an `import Calculator, only: [add: 2]` will do the trick or even passing in a function like `def add(x, y, implementation \\ Calculator.add/2)`. Before you complicate your application consider what's going to be the easiest to debug and reason about when it comes to behaviours.
