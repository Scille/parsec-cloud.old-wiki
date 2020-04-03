#  Requirements

* QtCreator to edit forms and rc files
* Poedit (a text editor also works) to edit translation files
* A Parsec environment

# Useful commands

* `python setup.py extract_translations` to extract the translation strings from the sources and the forms
* `python setup.py generate_pyqt` to convert the forms to .py files, compile the translations, and create the resources binary

# Files

* .ui are form files, they can be edited with QtCreator (or QDesigner). It's a WYSIWYG for Qt interfaces
* .po are translation files for a specific language, they can be edited with Poedit or any text editor
* .pro files are a Qt project. In this case we only use them to include all Qt forms so that translations can be extracted from them.
* .qrc files are Qt resources file. They include icons, logos, generated translations, CSS, ... All files we want to package with the app. They can be edited with QtCreator.

# Things to know

* Styles meant for the whole application and not aiming at any particular widget can be put into rc/styles/main.css.
* Particular style aiming at a specific widget should be put into the corresponding .ui file, always on the main widget in the .ui file, and always using ID selector (#widget_name), never class selector.
* Strings that are meant to be shown to the user should always be translated using the translate function in lang.py. A common thing to do is to import this function as _. See [this page](https://github.com/Scille/parsec-cloud/wiki/Translations) for more information on translations.

```
from parsec.core.gui.lang import translate as _

s = _("TEXT_STRING_TO_TRANSLATE")
```
* As much as possible, try different resolutions for the GUI. It usually looks great in your native resolution but can break down on smaller or larger resolutions.
* Try to write test (tests/core/gui). Those are incredibly awful to write but can save your ass. Be mindful that since the GUI is running in parallel with an asynchronous context, a test that will work on your computer may not on the CI if it has been badly written.
* Don't forget to run `generate_pyqt` after updating .ui files.
* If you're new to Qt, don't forget the layouts, or better yet, ask someone with some experience to create the .ui file for you so you don't have to deal with positioning, which can be a real pain.
* Every time you're using a function or an attribute in parsec.core, it should be through the jobs_ctx. Those operations are mostly asynchronous, they can be a little difficult to call and handle correctly but there are a few examples in the code base.
* Widgets can be sub classed directly in the .ui file, replacing what should be a QWidget by a custom widget defined in the code. This may be why some widgets have a specific behavior that you don't understand. Be mindful of the widget type in QtCreator interface.