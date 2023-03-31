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