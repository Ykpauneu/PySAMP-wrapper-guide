# How to create your own wrapper for the SA:MP plugin

## Foreword

To begin with, PySAMP is an evolving library that provides all the features for writing a game mode for SA:MP. But if you want to use plugins, there may be some problems with it, not all plugins are wrapped for Python. In this guide you will learn how to write a wrapper for any SA:MP plugin

### Preparing

Before you start, **install PySAMP and the SA:MP server**. You can find all the necessary links on our [Discord](https://discord.gg/z7tZ3mN6T4) server. For this guide, I will be creating a wrapper for the [ColAndreas](https://github.com/Pottus/ColAndreas) plugin.

## Installation

1. You need to install the plugin itself, so open the "Releases" tab.

![image](https://i.postimg.cc/tJQhPdKg/image.png)

2. Install the `*dll` file and move it to the plugins folder.

![image](https://i.postimg.cc/J4RXLhmS/image.png)
![image](https://i.postimg.cc/4yfxrmt5/image.png)

3. Edit the `server.cfg` file.

![image](https://i.postimg.cc/28qD3QzW/image.png)

## Getting started

Great, you have installed the plugin. Now you need to write a wrapper.

First, create a folder (I'll call it `pycolandreas`) in the server folder. In this folder create the file `__init__.py`, so it should be `*/pycolandreas/__init__.py`.

Now open the file `__init__.py` that we created earlier, in the next paragraph we will begin to break down the basics.

Open the python (`*/python/__init__.py`) folder and type the following.

```python
# */python/__init__.py
from pysamp import on_gamemode_init

@on_gamemode_init
def on_init():
    print("Server started")
```

This code will appear every time you start the server.

```
> samp-server.exe
> ...
> Server started!
```

### Imports

Before we start, we import the necessary libraries.

```python
# */pycolandreas/__init__.py
from pysamp import call_native_function, register_callback
```

Most likely your code editor will report that there is no such library (**PySAMP is not a pip package**), but this is normal.

### Natives

We will be wrapping functions using `call_native_function()`. Let's wrap the very first function, which is `CA_Init()`. The function should also be renamed to `snake_case`.

```python
# */pycolandreas/__init__.py
...

def col_andreas_init(): # Or ca_init()
    return call_native_function("CA_Init") # Plugin function
```

Now back to the server, change the `__init__.py` file.

```python
# */python/__init__.py
...
from pycolandreas import col_andreas_init

@on_gamemode_init
def on_init():
    print("Server started")
    col_andreas_init()
```

You may have made a simple mistake that you will often stumble upon.

```diff
- return CallNativeFunction(name, *arguments)
- SystemError: <built-in function CallNativeFunction> returned NULL without setting an error
```

It is very easy to avoid it - **the function that causes the error should be called in `@on_gamemode_init`**.

Now let's move on to wrapping the next function, we won't wrap all the functions. Next function is `CA_CreateObject()`. The good thing about this function is that it accepts arguments, which will allow us to deal with them as well.

```c
// */include/colandreas.inc

native CA_CreateObject(modelid, Float:x, Float:y, Float:z, Float:rx, Float:ry, Float:rz, bool:add = false);
```

As I said earlier, the function takes arguments. A minimal knowledge of Python is enough to wrap this function. I also advise you to always save the annotations. The names of the arguments must also be changed.

```python
# */pycolandreas/__init__.py
...

def col_andreas_create_object(
    model_id: int,
    x: float,
    y: float,
    z: float,
    rotation_x: float,
    rotation_y: float,
    rotation_z: float,
    add: bool = False
    ):
    return call_native_function(
        "CA_CreateObject",
        model_id,
        x,
        y,
        z,
        rotation_x,
        rotation_y,
        rotation_z,
        add
    )

```

You can also call this function in `*/python/__init__.py`, don't forget to import it from `pycolandreas`.

Read the documentation of the plugin (if it exists of course) to avoid ridiculous errors or other behavior of the function. Colandreas docs says: **CA_CreateObject - ONLY CREATES THE COLLISION, NOT THE IN-GAME OBJECT**.

This function also returns the ID (`index`) of the created object, this will come in handy later when we work with OOP, now we just wrap the `CA_DeleteObject()` function and pass `index` from `CA_CreateObject()`.

```python
# */pycolandreas/__init__.py
...

def col_andreas_delete_object(index: int):
    return call_native_function("CA_DeleteObject", index)
```

Now we can delete our object, let's try.

```python
from pycolandreas import (
    col_andreas_init,
    col_andreas_create_object,
    col_andreas_delete_object
)

@on_gamemode_init
def on_init():
    print("Server started")
    col_andreas_init()
    index = col_andreas_create_object()
    col_andreas_delete_object(index)
```

As you can see it's pretty simple.

You will often encounter different types of data, so without explaining them all, here is a simple table.

| Pawn    | Python      |
|---------|-------------|
| `Float`   | `float`       |
| `Integer` | `int`         |
| `Boolean` | `bool`        |
| `[]`      | `str or list` |

Examples will be just below.

```
Float: x => x: float
Integer: x => x: int
Boolean: x => x: bool
type[] => x: str
types[] => x: list
```

Notice the difference between `str` and `list`. When you write your own wrapper, you can easily notice where str and list are used.

### OOP

Now you will learn how to work with OOP. In fact, there is nothing complicated, we have already done a wrapper for two functions, the same functions we are going to use now. First, let's create a file, call it `caobject.py` (`*/pycolandreas/caobject.py`) to make it clearer to you.

Now create a class, name it whatever you like, I'll call it `CAObject()` to avoid confusion with `pysamp.object.Object`.


```python
# */pycolandreas/caobject.py
from pycolandreas import (
    col_andreas_create_object,
    col_andreas_delete_object
)

class CAObject():
    def __init__(self, index):
        self.index = index

```
# NOT FINISHED

### Pass by refenrce

Some functions take an argument with a `&` (`&Float: x`, etc.) sign. I will now tell you how to work with them. PySAMP does not currently have any support for pass-by-refernce.

Many modules have functions whose value we can "save" to avoid pass-by-refernce. To make it as clear as possible, look at the [PySAMP-streamer](https://github.com/pysamp/PySAMP-streamer) wrapper.

```python
class DynamicActor:
    def __init__(
        self, id, x=None, y=None, z=None, rotation=None, health=None
    ) -> None:
        self.id = id
        self._x = x
        self._y = y
        self._z = z
        self._rotation = rotation
        self._health = health

    @classmethod
    def create(
        cls,
        model_id: int,
        x: float,
        y: float,
        z: float,
        rotation: float,
        invulnerable: bool = True,
        health: float = 100.0,
        world_id: int = -1,
        interior_id: int = -1,
        player_id: int = -1,
        stream_distance: float = 200.0,
        area_id: int = -1,
        priority: int = 0,
    ) -> "DynamicActor":
        return cls(
            create_dynamic_actor(
                model_id,
                x,
                y,
                z,
                rotation,
                invulnerable,
                health,
                world_id,
                interior_id,
                player_id,
                stream_distance,
                area_id,
                priority,
            ),
            x,
            y,
            z,
            rotation,
            health,
        )
```

Here, in addition to returning the id of the created actor to the class, we also return `x`, `y`, `z`, `rotation`, `health`. At the moment this is one of the easiest, if not the only way to avoid pass-by-reference.
