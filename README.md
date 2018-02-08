# fbs tutorial
This tutorial is meant for Windows, Mac and Ubuntu. You need
[Python 3.5](https://www.python.org/downloads/release/python-354/).
(Later versions may work as well, but are not officially supported.)

## Setup
[Download](https://github.com/mherrmann/fbs-tutorial/archive/master.zip)
the Zip file of this repository and extract it. Then, open a command prompt
and `cd` into it:

    cd .../path/to/fbs-tutorial-master

Create a virtual environment:

    python3 -m venv venv

Activate the virtual environment:

    # On Mac/Linux:
    source venv/bin/activate
    # On Windows:
    call venv\scripts\activate.bat

The remainder of the tutorial assumes that the virtual environment is active.

Install the required libraries (most notably, `fbs` and `PyQt5`):

    pip install -r requirements.txt

## Run the app
This repository contains a sample application. To run it from source, execute
the following command:

    python -m fbs run

This shows a (admittedly not very exciting) window:

![Screenshot of sample app](screenshots/app.png)

## Freezing your app
Running the app from source requires Python to be set up. We don't want your
users to have to do this. Instead, we want to create a standalone form of your
app that runs on your users' computers. In the context of Python applications,
this process is called "freezing".

Use the following command to turn your app's source code into a standalone
executable:

    python -m fbs freeze

This creates the folder `target/Tutorial`. You can copy this directory to any
other computer (with the same OS as yours) and run your app there! Isn't that
awesome?

## Creating an installer
Desktop applications are normally distributed by means of an installer.
On Windows, this would be an executable called `TutorialSetup.exe`.
On Mac, mountable disk images such as `Tutorial.dmg` are commonly used.
fbs lets you generate both of these files.

### Windows installer
To create an installer on Windows, please first install
[NSIS](http://nsis.sourceforge.net/Main_Page) and add its directory to your
`PATH` environment variable. Then, you can run the following command:

    python -m fbs installer

This creates an installer at `target/TutorialSetup.exe`. It lets your users pick
the installation directory and adds your app to the Start Menu. It also creates
an entry in Windows' list of installed programs. Your users can use this to
uninstall your app. The following screenshots show these steps in action.

<img src="screenshots/installer-windows-1.png" height="160"> <img src="screenshots/installer-windows-2.png" height="160"> <img src="screenshots/installer-windows-3.png" height="160"> <img src="screenshots/installer-windows-4.png" height="160">

<img src="screenshots/uninstaller-windows-1.png" height="160"> <img src="screenshots/uninstaller-windows-2.png" height="160"> <img src="screenshots/uninstaller-windows-3.png" height="160">

### Mac installer
Creating an installer on Mac is done with the same command as on Windows:

    python -m fbs installer

This creates the file `target/Tutorial.dmg` for distribution to your users.
Upon opening it, the following volume is displayed:

![Screenshot of installer on Mac](screenshots/installer-mac.png)

To install your app, your users simply drag its icon to the _Applications_
folder (also shown in the volume).

## Source code of the sample app
The source code for the sample app is in
[`src/main/python`](src/main/python/tutorial). It contains a
[`main.py` script](src/main/python/tutorial/main.py), which serves as the entry
point for the application:

```python
from tutorial.application_context import AppContext

import sys

if __name__ == '__main__':
    appctxt = AppContext()
    exit_code = appctxt.run()
    sys.exit(exit_code)
```

The script instantiates and then runs an _application context_. This is defined
in [`application_context.py`](src/main/python/tutorial/application_context.py):

```python
from fbs_runtime.application_context import ApplicationContext, \
    cached_property
from PyQt5.QtWidgets import QApplication, QMainWindow

class AppContext(ApplicationContext):
    def run(self):
        self.main_window.show()
        return self.app.exec_()
    @cached_property
    def main_window(self):
        result = QMainWindow()
        result.setWindowTitle('Hello World!')
        result.resize(250, 150)
        return result
```

Your apps should follow the same structure:

 * Create a subclass of `fbs_runtime.application_context.ApplicationContext`.
 * Define a `run()` method that ends with `return self.app.exec_()`.
 * Use `@cached_property` to define the objects of your app.
 * In your `main` script, instantiate the application context, invoke its
   `run()` method and pass the return value to `sys.exit(...)`.

This  may seem complicated at first. But it has several advantages: First, it
lets `fbs` define useful default behaviour (such as setting the
[app icon](src/main/icons) or letting you access resource files bundled with
your app). Also, as your application becomes more complex, you will find that
an application context is extremely useful for "wiring together" the various
Python objects that make up your app. The next section demonstrates both of
these advantages.

## A more complicated example
Take a look at
[`application_context_2.py`](src/main/python/tutorial/application_context_2.py).
It defines a new `@cached_property`:

```python
class AppContext(ApplicationContext):
    ...
    @cached_property
    def image(self):
        return QPixmap(self.get_resource('success.jpg'))
```

A `@cached_property` is simply a Python `@property` whose value is cached.
Here's how it is used:

```python
class AppContext(ApplicationContext):
    ...
    def main_window(self):
        ...
        image_container.setPixmap(self.image)
```

The first time `self.image` is accessed, the `return QPixmap(...)` code from 
above is executed. After that, the value is cached and returned without
executing the code again.

`@cached_property` is extremely useful for instantiating and connecting the
Python objects that make up your application. Define a `@cached_property` for
each component (a window, a database connection, etc.). If it requires other
objects, access them as properties, like `self.image` above. The fact that all
parts of your application live in one place (the application context) makes it
extremely easy to manage them and see what is used where.

To see the new example in action, change the line

```python
from tutorial.application_context import AppContext
```

in your copy of [`main.py`](src/main/python/tutorial/main.py) to

```python
from tutorial.application_context_2 import AppContext
```

Then, run `python -m fbs run` again. You will be rewarded ;-)

### Resources
Another feature of our new example was the call
`self.get_resource('success.jpg')`. It loads an image that lives in the folder
[`src/main/resources`](src/main/resources/base).
But what if the user is running the compiled form of your app? In that case,
there is no `src/...`, because the directory structure is completely different.

The answer is that `get_resource(...)` is clever enough to determine whether it
is running from source, or from the compiled form of your app. To ensure that
the image is in fact distributed alongside your application, `fbs` copies all
files from `src/main/resources` into the `target/Tutorial` folder. So, if you
have data files that you want to include (such as images, `.qss` style sheets -
Qt's equivalent of `.css` files - etc.) place them in `src/main/resources`.

### Different OSs
Often, you will want to use different versions of a resource file depending on
the operating system. A typical example of this are `.qss` files where you
modify your app's style to match the current OS.

The solution for this is that `get_resource(...)` first looks for a
platform-specific version of the given file. Depending on the current OS, it
searches the following locations:

 * `src/main/resources/windows`
 * `src/main/resources/mac`
 * `src/main/resources/linux`

If it can't find the file in any of these folders, it falls back to
`src/main/resources/base`.

## Working with an IDE

So you'd like to launch the app using your IDE and use your IDE's debuging
support?

Just use the `launch.py` script as a launcher for the application.

Example run configuration for JetBrains PyCharm:

![Screenshot of PyCharm run configuration](screenshots/pycharm-run-configuration.png)

Note: make sure that the working directory of the script is the root
directory of this repo!


## Up next...
As of February 2018, this tutorial is a work in progress. Still to come:

 * Creating an installer for Ubuntu (Linux)
 * Codesigning so your users don't get ugly "app is untrusted" messages
 * Automatic updates

Feel free to share the link to this tutorial! If you are not yet on fbs's
mailing list and want to be notified when the tutorial is expanded,
[sign up here](http://eepurl.com/ddgpnf).