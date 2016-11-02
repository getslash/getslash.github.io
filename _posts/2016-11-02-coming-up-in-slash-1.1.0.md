---
layout: post
title:  "Coming up in Slash 1.1.0"
date: "2016-11-02"
categories: slash
---

Slash 1.1.0 is nearing completion and going into beta. Once we have confidence that the new version has no unexpected regressions, we'll release it to the wild. In the meantime, we thought you might be interested to learn about the notable changes coming in this version, so we included them below:

### Fixture Improvements

Slash now supports the `slash.exclude` decorator to exclude specific parametrizations of specific tests:

```python
import slash

SUPPORTED_SIZES = [10, 15, 20, 25]

@slash.exclude('car.size', [10, 20])
def test_car(car):
    ...

@slash.parametrize('size', SUPPORTED_SIZES)
@slash.fixture
def car(size): # <-- will be skipped for sizes 10 and 20
    ...
```

An additional feature is `this.test_start` and `this.test_end` -- two callbacks that fixtures can use to be notified when tests start or end while they're active. This is especially useful for widely-scoped fixtures


### Ability to Control Test Sorting During Collection

We've added the `tests_loaded` hook to enable you to control how collected tests are preprocessed. This is especially useful for controlling execution order:

```python
@slash.hooks.tests_loaded.register
def tests_loaded(tests):
    for index, test in enumerate(reversed(tests)):
        test.__slash__.set_sort_key(index)
```

### Suite File Filters

Suite files can now contain filters on specific items via a comment beginning with `filter:`, e.g:

```
/path/to/test.py # filter: x and not y``
```

### Logging Improvements

Logs emitted before logging is configured are now buffered and emitted into the session log to ease debugging.

Slash also now supports an optional "error log" -- an additional path to a log file that will only record added errors or failures. This eases the investigation of failures given verbose tests/sessions.

### Misc. Changes

* Tests skipped with `@slash.skipped` now don't perform setup or fixture expansion, saving a lot of time during execution
* Added `slash list-plugins` command to list active/installed plugins in the current environment/config
* When recording exceptions through `handling_exceptions`, Slash now properly reports the stack frames above the call site, through frame mocking
* Many other small bugfixes and improvements



The full changelog can be found at [the usual place](http://slash.readthedocs.io/en/latest/changelog.html). Feel free to try the beta at our `develop` branch on GitHub, and as usual feedback/issues/pull requests are very welcome!

Happy Slashing!
