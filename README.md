## concurrent-log-handler

This package provides an additional log handler for Python's standard logging
package (PEP 282). This handler will write log events to a log file which is
rotated when the log file reaches a certain size.  Multiple processes can
safely write to the same log file concurrently. Rotated logs can be gzipped
if desired. Both Windows and POSIX systems are supported.  An optional threaded 
queue logging handler is provided to perform logging in the background.

This is a fork of Lowell Alleman's ConcurrentLogHandler 0.9.1 which fixes
a hanging/deadlocking problem. See this:

https://bugs.launchpad.net/python-concurrent-log-handler/+bug/1265150

Summary of other changes:

* Renamed package to `concurrent_log_handler`
* Provide `use_gzip` option to compress rotated logs
* Support for Windows
* Uses file locking to ensure exclusive write access
  Note: file locking is advisory, not a hard lock against external processes
* More secure generation of random numbers for temporary filenames
* Change the name of the lockfile to have .__ in front of it.
* Provide a QueueListener / QueueHandler implementation for 
  handling log events in a background thread. Optional: requires Python 3. 
* Allow setting owner and mode permissions of rollover file on Unix
* Depends on `portalocker` package, which (on Windows only) depends on `PyWin32`

## Links

* [concurrent-log-handler on Github](https://github.com/Preston-Landers/concurrent-log-handler)
* [concurrent-log-handler on the Python Package Index (PyPI)](https://pypi.org/project/concurrent-log-handler/)

## Instructions and Usage

### Installation

You can download and install the package with `pip` using the following command:

    pip install concurrent-log-handler

This will also install the portalocker module, which on Windows in turn depends on pywin32.

If installing from source, use the following command:

    python setup.py install

To build a Python "wheel" for distribution, use the following:

    python setup.py clean --all bdist_wheel
    # Copy the .whl file from under the "dist" folder

### Important Requirements

Concurrent Log Handler (CLH) is designed to allow multiple processes to write to the same
logfile in a concurrent manner. It is important that each process involved MUST follow
these requirements:

* You can't serialize a handler instance and reuse it in another process. This means you cannot, for
  example, pass a CLH handler instance from parent process to child process using
  the `multiprocessing` package in spawn mode (or similar techniques that use serialized objects).
  Each child process must initialize its own CLH instance.

* When using the `multiprocessing` module in "spawn" (non-fork) mode, each child process must create
  its OWN instance of the handler (`ConcurrentRotatingFileHandler`). The child target function
  should call code that initializes a new CLH instance.

   * This requirement does not apply to threads within a given process. Different threads within a
     process can use the same CLH instance. Thread locking is handled automatically.

   * This also does not apply to `fork()` based child processes such as gunicorn --preload. 
     Child processes of a fork() call should be able to inherit the CLH object instance.

   * This limitation exists because the CLH object can't be serialized, passed over a network or
     pipe, and reconstituted at the other end.

* It is important that every process or thread writing to a given logfile must all use the same
  settings, especially related to file rotation. Also do not attempt to mix different handler
  classes writing to the same file, e.g. do not also use a `RotatingFileHandler` on the same file.

* Special attention may need to be paid when the log file being written to resides on a network
  shared drive or a cloud synced folder (Dropbox, Google Drive, etc.). Whether the multiprocess 
  advisory lock technique (via portalocker) works in these folders may depend on the details of 
  your configuration.

  Note that a `lock_file_directory` setting (kwarg) now exists (as of v0.9.21) which lets you
  place the lockfile at a different location from the main logfile. This might solve problems
  related to trying to lock files in network shares or cloud folders (Dropbox, Google Drive, etc.)
  However, if multiple hosts are writing to the same shared logfile, they must also have access
  to the same lock file.

  Alternatively, you may be able to set your cloud sync software to ignore all `.lock` files.

* A separate handler instance is needed for each individual log file. For instance, if your app
  writes to two different log files you will need to set up two CLH instances per process.

### Simple Example

Here is a simple direct usage example:

```python
from logging import getLogger, INFO
from concurrent_log_handler import ConcurrentRotatingFileHandler
import os

log = getLogger(__name__)
# Use an absolute path to prevent file rotation trouble.
logfile = os.path.abspath("mylogfile.log")
# Rotate log after reaching 512K, keep 5 old copies.
rotateHandler = ConcurrentRotatingFileHandler(logfile, "a", 512*1024, 5)
log.addHandler(rotateHandler)
log.setLevel(INFO)

log.info("Here is a very exciting log message, just for you")
```

See also the file [src/example.py](src/example.py) for a configuration and usage example.
This shows both the standard non-threaded non-async usage, and the use of the `asyncio`
background logging feature. Under that option, when your program makes a logging statement, 
it is added to a background queue and may not be written immediately and synchronously. 

### Configuration

To use this module from a logging config file, use a handler entry like this:

```ini
[handler_hand01]
class=handlers.ConcurrentRotatingFileHandler
level=NOTSET
formatter=form01
args=("rotating.log", "a")
kwargs={'backupCount': 5, 'maxBytes': 1048576, 'use_gzip': True}
```
    
That sets the files to be rotated at about 10 MB, and to keep the last 5 rotations.
It also turns on gzip compression for rotated files.

Please note that Python 3.7 and higher accepts keyword arguments (kwargs) in a logging 
config file, but earlier versions of Python only accept positional args.

Note: you must have an `import concurrent_log_handler` before you call fileConfig(). For
more information see http://docs.python.org/lib/logging-config-fileformat.html

### Recommended Settings

For best performance, avoid setting the `backupCount` (number of rollover files to keep) too
high. What counts as "too high" is situational, but a good rule of thumb might be to keep
around a maximum of 20 rollover files. If necessary, increase the `maxBytes` so that each
file can hold more. Too many rollover files can slow down the rollover process due to the
mass file renames, and the rollover occurs while the file lock is held for the main logfile.

How big to allow each file to grow (`maxBytes`) is up to your needs, but generally a value of 
10 MB (1048576) to 100 MB (1048576) is reasonable.

Gzip compression is turned off by default. If enabled it will reduce the storage needed for rotated
files, at the cost of some minimal CPU overhead. Use of the background logging queue shown below 
can help offload the cost of logging to another thread.  

Sometimes you may need to place the lock file at a different location from the main log
file. A `lock_file_directory` setting (kwarg) now exists (as of v0.9.21) which lets you
place the lockfile at a different location. This can often solve problems related to trying
to lock files in cloud folders (Dropbox, Google Drive, OneDrive, etc.) However, in
order for this to work, each process writing to the log must have access to the same
lock file location, even if they are running on different hosts.

### Line Endings

By default, the logfile will have line endings appropriate to the platform. On Windows
the line endings will be CRLF ('\r\n') and on Unix/Mac they will be LF ('\n'). 

It is possible to force another line ending format by using the newline and terminator
arguments.

The following would force Windows-style CRLF line endings on Unix:

    kwargs={'newline': '', 'terminator': '\r\n'}

The following would force Unix-style LF line endings on Windows:

    kwargs={'newline': '', 'terminator': '\n'}

### Background logging queue

To use the background logging queue, you must call this code at some point in your
app after it sets up logging configuration. Please read the doc string in the
file `concurrent_log_handler/queue.py` for more details. This requires Python 3.
See also [src/example.py](src/example.py).

```python
from concurrent_log_handler.queue import setup_logging_queues

# convert all configured loggers to use a background thread
setup_logging_queues()
```

This module is designed to function well in a multi-threaded or multi-processes 
concurrent environment. However, all writers to a given log file should be using
the same class and the *same settings* at the same time, otherwise unexpected
behavior may result during file rotation. 

This may mean that if you change the logging settings at any point you may need to 
restart your app service so that all processes are using the same settings at the same time.


## Other Usage Details

The `ConcurrentRotatingFileHandler` class is a drop-in replacement for
Python's standard log handler `RotatingFileHandler`. This module uses file
locking so that multiple processes can concurrently log to a single file without
dropping or clobbering log events. This module provides a file rotation scheme
like with `RotatingFileHandler`.  Extra care is taken to ensure that logs
can be safely rotated before the rotation process is started. (This module works
around the file rename issue with `RotatingFileHandler` on Windows, where a
rotation failure means that all subsequent log events are dropped).

This module attempts to preserve log records at all cost. This means that log
files will grow larger than the specified maximum (rotation) size. So if disk
space is tight, you may want to stick with `RotatingFileHandler`, which will
strictly adhere to the maximum file size.

Important:

If you have multiple instances of a script (or multiple scripts) all running at
the same time and writing to the same log file, then *all* of the scripts should
be using `ConcurrentRotatingFileHandler`. You should not attempt to mix
and match `RotatingFileHandler` and `ConcurrentRotatingFileHandler`.
The file locking is advisory only - it is respected by other Concurrent Log Handler
instances, but does not protect against outside processes (or different Python logging 
file handlers) from writing to a log file in use.

## Changelog

See [CHANGELOG.md](CHANGELOG.md)

## Contributors

The original version was written by Lowell Alleman.

Other contributors are listed in [CONTRIBUTORS.md](CONTRIBUTORS.md).

## License

See the [LICENSE file](LICENSE)
