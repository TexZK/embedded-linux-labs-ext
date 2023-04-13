# `menuconfig`

The `menuconfig` is a common tool to assist the creation and editing of configuration files, usually named `.config`.

It can:

* change options dynamically, depending on predicate evaluation;
* search for specific options via a handy search tool;
* check for option dependencies;
* import and export configuration files.

It's typically accessed by entering the root folder of the source code, where a `Makefile` is capable of this tool (usually via `Kconfig`):

```console
$ cd "/source/code/path"
$ make menuconfig
```


## Keyboard

++left++ and ++right++ move the cursor within the lower horizontal action list.<br/>
++enter++ applies the highlighted action entry.

++up++ and ++down++ move the cursor within the vertical main list.<br/>
++space++ changes the highlighted main entry.

++backspace++ acts as *backspace* within input text fields.<br/>
If the key alone doesn't work, try with ++ctrl+backspace++.

++esc++-++esc++ exits the current dialog.

++slash++ opens the search tool.

++y++/++n++/++m++ changes the highlighted option to: *enabled* (`y`), *disabled* (`n`), *module* (`m`).

Press the highlighted letter of menu entries to select them, or to activate the highlighed action.


## Actions

`<Select>` selects the highlighted main entry.

`<Exit>` exits the current dialog.

`<Help>` shows information of the highlighted main entry:<br/>
If on a configuration option, it shows its **symbol** and **value** (very useful!)

`<Save>` and `<Load>` are used to export/import a configuration file.


## Search tool

The search tool is called by typing ++slash++ anywhere in the interface.

This brings up a dialog where you can type the text to be searched, like a configuration symbol (e.g. `TARGET_ALIAS`).

You can go to a listed match directly by typing its entry number (e.g. ++1++ for `(1)`), pushing a **new** dialog.

After any changes, `Exit` to return to the search match list, where you can confrm that the value assigned is the expected one.

`Exit` again to return to the page before the search happened.
