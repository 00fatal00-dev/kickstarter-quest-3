# Coding Guidelines

This document describes GDScript coding style and best practices to organize code base to keep sane when developing mid-to-large projects.

The ideas exposed below take inspiration from good practices from different paradigms and languages, especially from Python and functional programming, as well as the official GDScript documentation.

In order of importance:

1. [GDScript Style Guide](http://docs.godotengine.org/en/latest/getting_started/scripting/gdscript/gdscript_styleguide.html)
1. [Static typing in GDScript](http://docs.godotengine.org/en/latest/getting_started/scripting/gdscript/static_typing.html)
1. [Docs writing guidelines](http://docs.godotengine.org/en/latest/community/contributing/docs_writing_guidelines.html)
1. [Boundaries - A talk by Gary Bernhardt from SCNA 2012](https://www.destroyallsoftware.com/talks/boundaries) & [Functional Core, Imperative Shell](https://www.destroyallsoftware.com/screencasts/catalog/functional-core-imperative-shell)
1. [The Clean Architecture in Python](https://www.youtube.com/watch?v=DJtef410XaM)
1. [Onion Architecture Without the Tears - Brendan Richards](https://www.youtube.com/watch?v=R2pW09tMCnE&t=1095s)
1. [Domain Driven Design Through Onion Architecture](https://www.youtube.com/watch?v=pL9XeNjy_z4)

There isn’t a straightforward way of transposing these ideas into an object-oriented setting such as when working with Godot since it has its way of handling interactions.

To create modular and composable systems, we have to manage boundaries: the places where different game systems interact with one another. Especially the interaction of the game systems with the user.

## Code Writing Style

This section shows our programming style by example.

<!-- TODO: Add a short but complete, real-world example -->

```gdscript
extends Node

"""
A brief description of the class's role and functionality

A longer description, if needed, possibly of multiple paragraphs. Properties
and method names should be in backticks like so: `_process`, `x` etc.

Notes
-----
Specific things that don't fit the class's description above.

Keep lines under 100 characters long
"""
```

Include `class_name` only if necessary: if you need to check for this type in other classes, or to be able to create the node in the create node dialogue.

```gdscript
class_name MyNode
```

Signals go first and don't use parentheses unless they pass function parameters. Use the past tense to name signals. Append `_started` or `_finished` if the signal corresponds to the beginning or the end of an action.

```gdscript
signal moved
signal talk_started(parameter_name)
signal talk_finished
```

Place `onready` variables after signals, because we mostly use them to keep track of child nodes this class accesses. Having them at the top of the file makes it easier to keep track of dependencies.

```gdscript
onready var timer : = $Timer
onready var ysort : = $YSort
```

After that enums, constants, and exported variables, in this order. The enums' names should be in `CamelCase` while the values themselves should be in `ALL_CAPS_SNAKE_CASE`. The reason for this order is that exported variables might depends on previously defined enums and constants.

```gdscript
enum TileTypes { EMPTY=-1, WALL, DOOR }

const MAX_TRIALS : = 3
const TARGET_POSITION : = Vector2(2, 56)

export(int) var number
```

Follow enums with member variables. Their names should use `snake_case`. Define setters and getters when properties alter their behavior instead of using methods to access them. They should start with an `_` to indicate these are private methods, and use the names `_set_variable_name`, `_get_variable_name`.

```gdscript
var animation_length : = 1.5
var tile_size : = 40
var side_length : = 5 setget _set_side_length, _get_side_length
```

Define private and virtual methods, starting with a leading `_`.

```gdscript
func _init() -> void:
  pass

func _process(delta: float) -> void:
  pass
```

Then define public methods. Include type hints for variables and the return type.

You can use a brief docstring, if need be, to describe what the function does and what it returns. To describe the return value in the docstring, start the sentence with `Returns`. Use the present tense and direct voice. See Godot's [documentation writing guidelines](http://docs.godotengine.org/en/latest/community/contributing/docs_writing_guidelines.html) for more information.

```gdscript
func can_move(cell_coordinates: Vector2) -> bool:
  return grid[cell_coordinates] != TileTypes.WALL
```

Use `return` only at the beginning and end of functions. If `return` is at the beginning, you can use it as a defense mechanism in an `if` statement.

**Don't** return in the middle of the method. It makes it harder to track returned values. Here's an example of a **clean** and readable method:

```gdscript
func start_quest(id: String) -> Quest:
  """
  Finds the quest corresponding to the `id` in the database and calls its start
  method.
  Returns the Quest object so other nodes can connect to its signals.
  """
  var quest : = get_quest_from_database(id)
  if not quest:
    return null
  quest.start()
  return quest
```

Another example of a function with **good** return statements:

```gdscript
func _set_elements(elements: int) -> bool:
  """
  Sets up the shadow scale, number of visual elements and instantiates as needed.
  Returns true if the operation succeeds, else false
  """
  if not has_node("SkinViewport") or \
     elements > ELEMENTS_MAX or \
     not has_node("Shadow"):
    return false

  # If the check succeeds, proceed with the changes
  var skin_viewport : = $SkinViewport
  var skin_viewport_staticbody : = $SkinViewport/StaticBody2D
  for node in skin_viewport.get_children():
    nodef i != skin_viewport_staticbbody:
      node.queue_free()

  var interval : = INTERVAL
  var r : = RandomNumberGenerator.new()
  r.randomize()
  for i in range(elements):
    var e : = Element.new()
    e.node_a = "../StaticBody2D"
    e.position = skin_viewport_staticbody.position
    e.position.x += r.randf_range(interval.x, interval.y)
    interval = interval.rotated(PI/2)
    skin_viewport.add_child(e)

  var shadow : = $Shadow
  shadow.scale = SHADOW.scale * (1.0 + elements/6.0)
  return true
```

### Avoid `null` like the plague

**Use `null` only if you're forced to**. Instead, think about alternatives to implement the same functionality with other types.

`None`, `null`, `NULL`, etc. references could be the biggest mistake in the history of computing. Here's an explanation from the man who invented it himself: [Null References: The Billion Dollar Mistake](https://www.infoq.com/presentations/Null-References-The-Billion-Dollar-Mistake-Tony-Hoare).

For programming languages that rely on `null`, such as GDScript, it's impossible to get rid of it completely: a lot of functionality relies on built-in functions that work with and return `null` values.

`null` can behave like any other value in any context, so the compiler can't find errors caused by `null` at compile time. `null` exceptions are only visible at runtime. This makes it more likely to write code that will fail when someone plays the game and it should be avoided like the plague.

You can use other values to initialize variables of certain types. For example, if a function returns a positive `int` number, if it is not able to calculate the desired return value, the function could return `-1` to suggest there was an error.

### Use static types

We use optional static typing with GDscript.

At the time of writing, static GDScript typing doesn't provide any perofrmance boosts or any other compiler features yet. But it does bring better code completion and better error reporting and warnings, which are good improvements over dynamically typed GDScript. In the future, it should bring performance improvements as well.

Be sure to check [Static typing in GDScript](http://docs.godotengine.org/en/latest/getting_started/scripting/gdscript/static_typing.html) to get started with this language feature.

Normally, you define typed variables like this:

```gdscript
var x : Vector2 = some_function_returning_Vector2(param1, param2)
```

But if `some_function_returning_Vector2` is also annotated with a return type, Godot can infer the type for us so we only need to add a colon after the variable's name:

```gdscript
func some_function_returning_Vector2(param1: int, param2: int) -> Vector2:
  # do some work
  return Vector2()

var v : = some_function_returning_Vector2(param1, param2) # The type is Vector2
```

_Note_ how we still use the collon in the assignment: `: =`. It isn't just `=`. Without the colon, the variable's type would be dynamic.

Use `: =` with a space between the colon and the equal sign, **not** `:=`. `: =` is easier to spot compared to `=`, in case someone forgets to use the colon.

**Let Godot infer the type whenever you can**. It's less error prone because the system keeps better track of types than we humanly can. It also pushes us to have proper return values for all the functions and methods that we write.

Since the static type system mostly brings better warnings and it isn't enforced, sometimes we have to help it out. The following snippet will make the problem clear:

```gdscript
var arr : = [1, 'test']
var s : String = arr.pop_back()
var i : int = arr.pop_back()
```

The `Array` type is a container for multiple diffrent types. In the example above, we have both an `int` and a `String` stored in the array. If you only wrote `var s : = arr.pop_back()`, Godot would complain because it doesn't know what type the `pop_back` method returns. You will get the same issue with all built-in methods that return the engine's `Variant` type. Open the code reference with <kbd>F4</kbd> and search for the methods to see that:

```
Variant pop_back()
  Remove the last element of the array.
```

`Variant` is a generic type that can hold any type Godot supports. That's why we have to explicitly write variable types in these cases: `var s : String = arr.pop_back()`.

In these cases, you must be careful as the following is also valid:

```gdscript
var arr : = [1, 'test']
var s : int = arr.pop_back()
var i : String = arr.pop_back()
```

You will not get any error with this code. At runtime, `s` will surprinsingly still contain a `String`, and `i` will contain an int. But a type check like `s is String` or `i is int` will return `false`. That's a weakness of the current type system that we should keep in mind.
