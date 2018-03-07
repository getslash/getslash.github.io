---
layout: post
title:  "Slash 1.5 Released"
date: "2018-03-08"
categories: slash
---

We are pleased to announce the 1.5.0 release of the Slash testing framework! This release is a huge
one, and has been in the making for quite some time now. 

Release 1.5.0 has seen some major changes in areas relating to usability, both for users and for
framework developers. It also puts emphasis on integration with other tools and services (like
Backslash), making the integration process easier and more productive.

Below are some of the changes that made it to this release.

## Runtime Improvements

### Parametrization labels

One of the biggest problems with parameterization in Slash has been the lack of ability to *label*
certain parameters. Take the following code as an example:

```python

@slash.parametrize('param', [None, SmallObject(), BigObject()])
def test_something(param):
    ...
```

The above would generate non-meaningful variation names in Slash 1.4
(e.g. `test_something(param=param0)`), making it very hard to understand what actually ran after the
fact. The problem is compounded when reporting to services such as Backslash, in which it becomes
critical to be able to know the meaning of the different variations, not only by the value they use,
but by the "name" of any specific variation.

Slash 1.5 solves this issue by introducing the ability to label parameters:

```python
@slash.parametrize('param', [
    None // slash.param('none'),
    SmallObject() // slash.param('small'),
    BigObject() // slash.param('big'),
])
def test_something(param):
    pass
```

This will generate more meaningful result names, like `test_something(param=big)`, which will help
filtering it during the reporting phase.

If you find the postfix syntax above not to your liking, you can also use `slash.param('label',
'value')` instead.

### Rerunning Previous Sessions

In addition to the previous functionality of resuming failed sessions through `slash resume`, Slash
learned the `slash rerun` command, allowing you to rerun a previous session - not only failed or
not-run tests.

### Repeats in Suite Files

Suite files can now have a ``repeat: X`` marker to make the test run multiple times (Thanks
@pierreluctg!):

```
# suite_file.txt
path/to/test_something.py # repeat: 10
```


### Warning Control

A new API for controlling warning capture, `slash.ignore_warnings`, has been added. It allows you to
ignore warnings of specific category/origin, preventing Slash from capturing and reporting them:

```python
slash.ignore_warnings(category=MySpecificWarningClass, filename='/specific/spammy/file.py')
```

This is useful if your framework uses old or unmaintained third-party libraries which emit lots of warnings.

### Selective Debugging

Slash now supports the `--pdb-filter` command-line parameter. This flag allows you to selectively
activate pdb for specific exceptions thrown during a session. It supports the same flexible syntax
as `-k`:

```
$ slash run ./tests --pdb --pdb-filter 'TimeoutException and not CommandTimeoutException`
```

The above will enter PDB only for exceptions containing `TimeoutException` but not for
`CommandTimeoutException` errors.

## Logging Improvements

### Log Compression

Setting the new `log.compression.enabled` configuration path to `True` makes Slash compress logs
on-the-fly (by default using the *brotli* compression scheme). This is useful for projects emitting
large log files. 

To improve compression rates, logs use a relatively large compression window, which could delay the output of
recent log entries during runs. If this is a problem for you, we also support
`log.compression.use_rotating_raw_file` to have a separate log file containing recent entries.

### Core Log Level

Slash 1.5 separates the logging configuration used by tests and tools from the logging configuration
used by the Slash framework itself. While in some projects it makes sense to read logs coming from
both the testing project and Slash itself, in most cases developers aren't interested in debugging
Slash itself. This is why we separated the log level used by Slash to a separate configuration
variable, `log.core_log_level` - which is set to *WARNING* by default. This allows you to use
`TRACE` logs in your framework without having it spammed with Slash trace logs.

### Debugging Local Variables

By setting `log.traceback_variables` to `True`, traceback variable values will now be written to the
debug log upon failures/errors. This also include members of `self`, which is useful to debug
utilities and framework components.

### The Highlights Log

Slash 1.4 introduced the error log, which was a useful way to get only "important" logs about your
session. In 1.5 we decided to generalize that facility and renamed it to `highlights` log. Its path
is controlled using the `log.highlights_subpath`, and by default it will contain all errors emitted
during the session, but also additional logs marked with "highlight":

```python
slash.logger.info('This is an important event', extra={'highlight': True})
```

### Log Formatting using Timestamps

You can now use `timestamp` when formatting log file paths. This is a datetime object presenting the
creation time of the log file, so you can easily have test log directories containing the current
time by using:

```python
config.root.log.subpath = '{timestamp:%Y%m%d-%H%M%S}.log'
```

## Plugin Names

In Slash 1.4 and before, multi-worded plugin names caused annoyances in two contexts - command-line
activation and configuration management. Let's say you had a plugin named `my_plugin`:

```python
class MyPlugin(PluginInterface):
    def get_name(self):
        return 'my_plugin'
    
    def get_default_config(self):
        return {'some': 'config'}
```

This plugin's configuration would be loaded into `plugin_config.my_plugin` as expected, but
activating it would be cumbersome since it would require to pass `--with-my_plugin` -- which is
ugly. Changing its name to dash causes an issue with the configuration which is not easy to work
around.

Slash 1.5 changes the recommended practice to use spaces (hence `"my plugin"`). When plugins use
that notation, their configuration paths are now normalized using underscores, and activation uses
dashes as one would expect.

## Miscellaneous

* When using the built-in notifications plugin, a notification will be sent whenever a session
  enters a debugger. This is useful for fire-and-forget test runs.
* The notifications plugin now has a new configuration flag - `--notify-only-on-failure`, making it
  notify only when tests fail. The emails it sends are now also formatted based on the session
  result (e.g. changing the logo at the top of the message)
* `PluginInterface.get_config` has been deprecated and renamed to `PluginInterface.get_default_config`
* `before_interactive_shell`is a new hook allowing you to control the namespace used in interactive
  tests
* `slash list tests` now accepts the ``--warnings-as-errors`` flag, making it treat warnings it encounters as errors
* `-X` can now be used to turn off stop-on-error behavior. Useful if you have it on by default
  through a configuration file
* Location of the state files used by `slash resume` can now be configured through
  `run.resume_state_path`. This is useful in cases where you cannot use *sqlite* files in your
  homedir (e.g. if your home directory is mounted through NFS)
* Error objects now have their respective `exc_info` attribute containing the exception info for the current info (if available). This deprecates the use of the `locals`/`globals` attributes on traceback frames.
* During the execution of `error_added` hooks, traceback frame objects now have `python_frame`,
  containing the original Pythonic frame that yielded them. Those are cleared soon after the hook is
  called.
* `use.X` is now a shortcut for `use('x')` for fixture annotations
* The new `interruption_added` hook is fired whenever a test or a session is interrupted (through
  either a keyboard interrupt or a SIGTERM signal)
* `assert_raises` now raises `ExpectedExceptionNotCaught` if exception wasn't caught also allowing
  inspection of the expected exception object
* Native Python warnings are now emitted for WARNING-level Logbook logs
* `session.results.current` is now always the same as `context.result`
* `traceback_level` configuration parameter has been renamed to `console_traceback_level`, since it
  effectively controls only the console output
* Many, many other small bug fixes and improvements

That's about it! Hope you find the new release useful, and as usual feel free to leave us feedback
or report issues/improvement suggestions!
