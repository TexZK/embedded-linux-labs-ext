# *git commit* hash expansion

Within the [*crosstool-NG* tutorial](../qemu/toolchain.md) we encountered a *git* commit `7622b490`.
That code is a shorthand for the full *commit hash number*: `7622b490a359f6cc6b212859b99d32020a8542e7`.

While the creation of a shorthand simply keeps the first letters (retaining hash unicity), the expansion requires some look-up to be performed.


## Local repository

In case you have a local *git* repository, you can use the `rev-parse` command within it:

```console
$ cd "$HOME/embedded-linux-qemu-labs/toolchain/crosstool-ng/"
$ git rev-parse 7622b490
7622b490a359f6cc6b212859b99d32020a8542e7
```


## GitHub

In case the repository is hosted by [*GitHub*](https://github.com/), you can retrieve the full hash from the web pages.

Open the main page of the specific commit:

* [https://github.com/crosstool-ng/crosstool-ng/commits/**7622b490**](https://github.com/crosstool-ng/crosstool-ng/commits/7622b490)

By clicking on the shorthand of that entry, its URL gets expanded to the full hash code:

* [https://github.com/crosstool-ng/crosstool-ng/commit/**7622b490a359f6cc6b212859b99d32020a8542e7**](https://github.com/crosstool-ng/crosstool-ng/commit/7622b490a359f6cc6b212859b99d32020a8542e7)
