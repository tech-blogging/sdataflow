a small and simple language within the project [sblog][1].

[1]: https://github.com/haoxun/sblog

# Install

```
pip install sdataflow
```

# Concepts

`sdataflow` provides:

* A small and simple language to define the relation of entities. An `entity` is a logic unit defined by user(i.e. a data processing function), it generates some kind of `outcome` as a respond to some kind of input `outcome`(which might be generated by other Entity). Relations of entities forms a `dataflow`.
* A scheduler automatically runs entities and ships outcome to its destination.

# Language

## Tutorial

Let's start with a simplest case(**one-to-one** relation):

```
A --> B
```
where entity `B` accepts outcome of `A` as its input.

To define a **one-to-more** or **more-to-one** relation:

```
# one-to-more
A --> B
A --> C
A --> D

# more-to-one
B --> A
C --> A
D --> A
```
where in the **one-to-more** case, copies of outcome of `A` could be passed to `B`, `C` and `D`. In the **more-to-one** case, outcomes of `B`, `C` and `D` would be passed to `A`.

And here's the form of **outcome dispatching**, that is, a mechanism of sending different kinds of outcome of an entity to different destinations. For instance, entity `A` generates two kinds of outcome, say `[type1]` and `[type2]`, and pass outcomes of `[type1]` to `B`, outcomes of `[type2]` to `C`:

```
# one way.
A --> [type1]
A --> [type2]
[type1] --> B
[type2] --> C

# another way.
A --[type1]--> B
A --[type2]--> C
```
where identifier embraced in brackets(i.e. `[type1]`) represents the name of outcome. In contrast to the form of outcome dispatching, `A --> B` would simple pass outcome of `A`, with default name `A`(the name of entity generates the outcome), to entity `B`. Essentially, above form(statement contains brackets) overrides the name of outcome, and acts like a filter for outcome dispatching.

Outcome could be used to define **one-to-more**, **more-to-one** relations as well, in the same way discussed above:

```
# one-to-more example.
A --> [type1]
A --> [type2]
[type1] --> B
[type1] --> C
[type2] --> D
[type2] --> E

# more-to-one example.
A --> [type1]
B --> [type1]
[type1] --> C

```

After loading all user defined dataflow, there are basically two steps of analysis will be applied:

1. Build a DAG of dataflow. Break if error happens(i.e. syntax error, cyclic path).
2. Apply topology sort to DAG to get the linear ordering of entity invocation.

## Lexical Rules

```
ARROW          : re.escape('-->')
DOUBLE_HYPHENS : re.escape('--')
BRACKET_LEFT   : re.escape('[')
BRACKET_RIGHT  : re.escape(']')
ID             : r'\w+'
```

The effect of above rules would be equivalent as if passing such rules to Python's `re` module with the flag `UNICODE` being set.

## CFGs

```
start : stats

stats : stats single_stat
      | empty
      
single_stat : entity_to_entity
            | entity_to_outcome
            | outcome_to_entity
            
entity_to_entity : ID general_arrow ID

general_arrow : ARROW
              | DOUBLE_HYPHENS outcome ARROW

outcome : BRACKET_LEFT ID BRACKET_RIGHT
              
entity_to_outcome : ID ARROW outcome

outcome_to_entity : outcome ARROW ID
```

# API

## Form of Callback

As mentioned above, an entity stands for a user defined logic unit. Hence, after defining the relations of entities in the language discussed above, user should defines a set of callbacks, corresponding to each entity in the definition.

User can define two types of callback:

1. A **normal function** returns `None`(i.e. a function with no `return` statement), or an iterable object, of which the element is a (key, value) tuple, with key as the name of outcome and value as user
defined object.
2. A generator yields the element same as (1).

Input argument list of both types of callback could be:

1. An empty list, meaning that such callback accept no data.
2. An one-element list.

Code fragment for illustration:

```python
# normal function returns `None`, with empty argument list.
def func1():
    pass


# normal function return `None`, with one-element argument list.
def func2(items):
    for name_of_outcome, obj in items:
        # do something.


# normal function return elements, with one-element argument list.
def func3(items):
    # ignore `items`
    data = [('some outcome name', i) for i in range(10)]
    return data


# generator yield element, with one-element argument list.
def gen1(items):
    # ignore `items`
    for i in range(10):
        yield 'some outcome name', i

```

Note that the name of outcome is the string embraced in brackets(**not** including the brackets).

## Register And Execute Callback

`dataflow` provides a class `DataflowHandler` to parse `doc`(a string represents the relations of entities), register callbacks and schedule the execution of callbacks.

```
class DataflowHandler
    __init__(self, doc, name_callback_mapping)
        `doc`: unicode or utf-8 encoded binary data.
        `name_callback_mapping`: a dict of (`name`, `callback`) pairs. `name`
        could be unicode or utf-8 encoded binary data. `callback` is a function
        or generator.
    
    run(self)
        Automatically execute all registered callbacks.
```

Example:

```python
from sdataflow import DataflowHandler
from sdataflow.scheduler import create_data_wrapper

doc = ('A --[odd]--> B '
       'A --[even]--> C '
       'B --> D '
       'C --> D ')

def a():
    odd = create_data_wrapper('odd')
    even = create_data_wrapper('even')
    for i in range(1, 10):
        if i % 2 == 0:
            yield even(i)
        else:
            yield odd(i)

def b(items):
    default = create_data_wrapper('B')
    # remove 1.
    for outcome_name, number in items:
        if number == 1:
            continue
        yield default(number)

def c(items):
    default = create_data_wrapper('C')
    # remove 2.
    for outcome_name, number in items:
        if number == 2:
            continue
        yield default(number)

def d(items):
    numbers = {i for _, i in items}
    assert set(range(3, 10)) == numbers

name_callback_mapping = {
    'A': a,
    'B': b,
    'C': c,
    'D': d,
}

# parse `doc`, register `a`, `b`, `c`, `d`.
handler = DataflowHandler(doc, name_callback_mapping)

# execute callbacks.
handler.run()
```

In above example, `A` generates numbers in the range of 1 to 9, of which the odd numbers(1, 3, 5, 7, 9) are sent to `B`, the even numbers(2, 4, 6, 8) are sent to `C`. Then `B` removes number 1 and sends the rest(3, 5, 7, 9) to `D`, while `C` removes number 2 and sends the rest(4, 6, 8) to `D`. Finally, `D` receives outcomes of both `C` and `D`, and make sure that is equal to `set(range(3, 10))`.





