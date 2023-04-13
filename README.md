# Embedded Linux System Development

This is a collection of tutorials to be compiled with
[Material for MkDocs](https://squidfunk.github.io/mkdocs-material/),
an extension of [MkDocs](https://www.mkdocs.org/).


## Requirements

A basic installation of *Material for MkDocs* within a generic Python environment is enough:

```console
$ python -m pip install -r requirements.txt
$ python -m mkdocs build
$ python -c "p='./site/index.html'; import os, webbrowser; webbrowser.open(f'file://{os.path.realpath(p)}');"
```
