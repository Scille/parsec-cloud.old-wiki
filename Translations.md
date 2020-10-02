# Translations

## Introduction

Parsec is currently available in English and French. Other languages should not be hard to implement, provided that someone sufficiently fluent in any language is up to the task.

Its translation format being the well known and used .PO format, translations can be done using any software supporting this format (like poedit).

## Implementation

Since Parsec uses Qt, it should seem logical that we should also use Qt's translation framework. However, due to technical limitations, we switched to the gettext API.

Qt forms, once generated (`python setup.py generate_pyqt_forms`) are scanned, and translations strings are extracted (using `python setup.py extract_translations`) into PO format. PO format files, once translated, are compiled (as .mo) and stored into the Qt's resources system. We later monkeypatch Qt default translator to use our own translations at exec time.

## Add a string to translate

In the source code, we use the `translate` function in the `parsec.core.gui.lang` module to mark strings are translatable.

```python
from parsec.core.gui.lang import translate as _

l = QLabel(_("This string will be translatable."))
```

## Translation codes

We use translations code instead of the whole English string in the source code to improve the text consistency.

```python
# No
b = QPushButton(_("Cancel"))
# Yes
b = QPushButton(_("ACTION_CANCEL"))
```

Usually, codes are broken down into two categories: ACTION and TEXT. An action is something the user can click on, a text is a description. We also try to include some context on where the code is used, to make it easier to translate it later on:
```python
b.setText(_("ACTION_BOOTSTRAP_ORGANIZATION_VALIDATE"))
# or 
line_edit.setPlaceHolder(_("TEXT_BOOTSTRAP_ORGANIZATION_URL_PLACEHOLDER"))
```
To use variables in the translation string :
```python
b.setText(_("ACTION_BOOTSTRAP_ORGANIZATION_VALIDATE_organizationid-othervariable").format(organizationid=organization_id, othervariable=other_variable))
```
## Example

In widgets :
```python
from parsec.core.gui.lang import translate as _

number1 = 1
number2 = 2
my_label.setText("TEXT_MY_LABEL_number1-number2").format(
                number1=number1,
                number2=number2,
            )
```
Execute

`python setup.py extract_translations`

The po files should contain the newly created field :

```
msgid "TEXT_MY_LABEL_number1-number2"
msgstr ""
```

Replace the `msgstr ""` in po files by the message you want:
```
msgstr "<b>number 1:</b> {number1}\n<b>number 2:</b> {number2}"
```

When finished, execute `python setup.py extract_translations` to correctly reformat the .po file, then `python setup.py generate_pyqt` to include the translations in the GUI.

## Resources

[Microsoft International Terminology](https://www.microsoft.com/en-us/language)