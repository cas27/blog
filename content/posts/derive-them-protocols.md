+++ 
draft = false
date = 2019-01-06T11:41:07-08:00
title = "Derive Them Protocols"
slug = "deriving-protocols-in-elixir" 
tags = ["elixir"]
categories = ["elixir"]
thumbnail = "images/tn.png"
description = ""
+++

Using derive to implement a protocol is a quick way to give functionality to your modules.

Let's say you had a protocol that rounds values for you.  Like such:

```elixir
defprotocol Math do
  def round(v, p)
end

defimpl Math, for: Float do
  defdelegate round(v, p), to: Float
end
```

Which allows us to do something like this:

```elixir
iex(1)> Math.round(1.556, 2)  
1.56
```

Now say we wanted to round all the float values in our custom data structures.  Instead of having to implement the logic over and over would be nice if we could do it simply once.

Here is an example data structure.

```elixir
defmodule Data do
  defstruct x: 1.556, y: 2.556
end
```

If we tried to use our protocol as it stands now on the Data structure we'd get:

```elixir
iex(1)> Math.round(%Data{}, 2)
** (Protocol.UndefinedError) protocol Math not implemented for %Data{x: 1.556, y: 2.556}. This protocol is implemented for: Float
    (super_happy_fun_land) lib/math.ex:1: Math.impl_for!/1
    (super_happy_fun_land) lib/math.ex:2: Math.round/2
```

Let's implement the Math protocol for any such data structure in our system to take advantage of.

```elixir
defimpl Math, for: Any do
  defmacro __deriving__(module, _struct, _opts) do
    quote do
      defimpl Math, for: unquote(module) do
        def round(data, p) do
          rounded_map =
            data
            |> Map.from_struct()
            |> Enum.reduce(%{}, fn {k, v}, acc ->
              Map.merge(acc, %{k => round_float(v, p)})
            end)

          struct(unquote(module), rounded_map)
        end

        defp round_float(v, p) when is_float(v), do: Math.round(v, p)
        defp round_float(v, _), do: v
      end
    end
  end
end
```

We've implemented the Math protocol for Any and then defined the __deriving__ macro which will allow us to use this in our modules.

Then we can implement the round function for any module that takes advantage of the derive.

Finally let's add this to our Data module and give it a try.

```elixir
defmodule Data do
  @derive [Math]
  defstruct x: 1.556, y: 2.556
end

iex(1)> Math.round(%Data{}, 2)
%Data{x: 1.56, y: 2.56}
```

As you can see using derive is a nice tool if you have lots of data structures that all need to implement a protocol in a simple fashion and saves you the time from having to repeat yourself over and over.

