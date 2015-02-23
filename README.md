# execstream

## Overview

Libexecstream is a C++ library that allows you to run a child process and have its input, output and error
avaliable as standard C++ streams. 

Like this:

```c++
#include <exec-stream.h>
#include <string>
...
try {
    exec_stream_t [es](reference.html#constructor1)( "perl", "" ); <span class="comment">// run perl without any arguments </span>
    es.[in](reference.html#in)() << "print \"hello world\";"; <span class="comment">// and make it print "hello world" </span>
    es.[close_in](reference.html#close_in)();                        <span class="comment">// after the input was closed </span>
    std::string hello, world;
    es.[out](reference.html#out)() >> hello; <span class="comment">// read the first word of output </span>
    es.[out](reference.html#out)() >> world; <span class="comment">// read the second word </span>
}catch( std::exception const &amp; e ) {
    std::cerr << "error: "  <<  e.what()  <<  "\n";
}
```

Features:

*   Works on Linux and Windows
*   Uses threads
*   Does not depend on any other non-standard library
*   Distributed as source code only, requires you to compile and link one file into your program
*   BSD-style license

Another example:

```c++
#include <exec-stream.h>
...
exec_stream_t [es](reference.html#constructor0);
try {
    // run command to print network configuration, depending on the operating system
    #ifdef _WIN32
        es.[start](reference.html#start1)( "ipconfig", "/all" );
    #else
        es.[start](reference.html#start1)( "ifconfig", "-a" );
    #endif

    std::string s;
    while( std::getline( es.[out](reference.html#out)(), s ).good() ) {
        // do something with 's'
    }
}catch( std::exception const &amp; e ) {
    std::cerr << "error: "  <<  e.what()  <<  "\n";
}
```

## Installation

Libexecstream is provided in source code form only. In order to use it, you need to compile and link
one file, `exec-stream.cpp`, into your program.

On Linux, libexecstream was tested on Red Hat 9 with gcc compiler. Versions of gcc prior to 3.0 will not work.
Make sure that `exec-stream.h` is found somewhere on the include path, 
compile `exec-stream.cpp` as usual, link your program with `-lpthread`.
GCC must be configured with `--enable-threads`, which is by default on most Linux distributions.

On Windows, libexecstream was tested on XP and 95 flavors with VC++ 7 compiler. VC++ 6 will not work. 
Make sure that `exec-stream.h` is found somewhere on the include path, 
compile `exec-stream.cpp` as usual, link you program with multi-threaded runtime.

Example makefiles for Windows and Linux (used to build the testsute) are provided in the `test` directory
of the source distribution.

The `exec-stream.cpp` file includes several platform-dependent 
implementation files. Selection of platform-specific implementation is done at compile time: when `_WIN32`
macro is defined (usually by windows compiler) win32 implementation is included, when that macro is not defined,
posix implementation is included.

Header file `exec-stream.h` defines interface of the library and uses only standard C++. 
It does not include any platform-specific header files.

For more examples see the file `test/exec-stream-test.cpp` in the source distribution.  The interface provided by the library is documented in the Reference.

## Reference

Libexecstream provides one class, exec_stream_t, which has the following members:

## class error_t : public std::exception

Exceptions thrown from exec_stream_t members are derived from error_t. error_t is derived from std::exception and has no additional
public members besides constructors. Exceptions may be thrown from any exec_stream_t member function except destructor and accessors: in(), out() and err().
Writing to in() and reading out() and err() will also throw exceptions when errors occur.

## exec_stream_t()

Constructs exec_stream_t in the default state. You may change timeouts, buffer limits and text or binary modes
of the streams before `start`ing child process (see `set_buffer_limit`, 
`set_binary_mode`, `set_text_mode`, 
`set_wait_timeout`). In the default state, amount of data buffered for writing to child's stdin,
and amount of data read in advance from child's stdout and stderr is unlimited. On windows, all streams are in the text mode.

## exec_stream_t( std::string const &amp; program, std::string const &amp; arguments )

Constructs exec_stream_t in the default state, then starts program with arguments. Arguments containing space should be
included in double quotation marks, and double quote in such arguments should be escaped with backslash.

## template&lt; class iterator &gt; exec_stream_t( std::string const &amp; program, iterator args_begin, iterator args_end )

Constructs exec_stream_t in the default state, then starts program with arguments specified by the range args_begin, args_end. 
args_begin should be an input iterator that when dereferenced gives value assignable to std::string. 
Spaces and double quotes in arguments need not to be escaped.

## ~exec_stream_t()

Writes (with `timeout`) all pending data to child stdin, 
closes streams and waits (with `timeout`) for child process to stop.

## std::ostream &amp; in()

Returns output stream for writing to child's stdin.

## std::istream &amp; out()

Returns input stream for reading child's stdout.

## std::istream &amp; err()

Returns input stream for reading child's stderr.

## bool close_in()

Closes child's standard input after writing (with `timeout`) all pending data to it.

## void start( std::string const &amp; program, std::string const &amp; arguments )

Starts program with arguments. Arguments are space-separated. Arguments containing space should be
included in double quotation marks, and double quote in such arguments should be escaped with backslash.

## template&lt; class iterator &gt; void start( std::string const &amp; program, iterator args_begin, iterator args_end )

Starts program with arguments specified by the range args_begin, args_end. args_begin should be an input iterator that when dereferenced 
gives value assignable to std::string. Spaces and double quotes in arguments need not to be escaped.

## enum stream_kind_t { s_in=1, s_out=2, s_err=4, s_all=s_in|s_out|s_err, s_child=8  }

Used for the first argument to `set_buffer_limit`, `set_wait_timeout`, 
`set_binary_mode`, `set_text_mode` for selecting stream to operate upon.

## void set_buffer_limit( int stream_kind, std::size_t size )

For `out()` and `err()` streams (when `exec_stream_t::s_out` 
or `exec_stream_t::s_err` is set in the stream_kind), sets maximum amount of data to read from child process 
before it will be consumed by reading from `out()` or `err()`.

For `in()` stream (when `exec_stream_t::s_in` is set in the stream_kind)
sets maximum amount of data to store as result of writing to `in()` before it will be consumed by child process.

Setting limit for both input and output streams may cause deadlock in situations when both your program and child process
are writing data to each other without reading it. Such deadlock will cause the `timeout` to expire while
writing to `in()`.

When size argument to set_buffer_limit is 0, buffers are considered unlimited, and will grow unlimited if one side produce data that the other side does not consume.
This is the default state after exec_stream_t creation.

 set_buffer_limit will throw `exception` when called while child process is running.

## typedef unsigned long timeout_t

Type of second argument to `set_wait_timeout` - timeout in milliseconds.

## void set_wait_timeout( int stream_kind, timeout_t milliseconds )

For `out()` and `err()` streams (when `exec_stream_t::s_out` 
or `exec_stream_t::s_err` is set in the stream_kind), sets maximum amount of time to wait for a 
child process to produce data when reading `out()` and `err()` respectively.

For `in()` stream (when `exec_stream_t::s_in` is set in the stream_kind), 
sets maximum amount of time to wait for a child process to consume data that were written to `in()`. 
Note that when [buffer limit](#set_buffer_limit) for in() is not set, writing to in() always writes to buffer and does not wait for child at all.

If that amount of time is exceeded while reading in() or writing to out() and err(), exception is thrown.

When `exec_stream_t::s_child` is set in the stream kind, set_wait_timeout sets the maximum amount of time to wait
for a child process to terminate when `close` is called. If that amount of time is exceeded, close() will return false.

 set_wait_timeout will throw `exception` when called while child process is running.

## void set_text_mode( int stream_kind )

sets stream specified by `stream_kind` to text mode. In text mode, in the data written to child's stdin,
\n are replaced by \r\n; and in the data read from child's stdout and stderr, \r\n are replaced by \n. Text mode is the default on Windows. 
set_text_mode has no effect on Linux.

 set_text_mode will throw `exception` when called while child process is running.

## void set_binary_mode( int stream_kind )

sets stream specified by `stream_kind` to binary mode. All data written or read from streams are passed unchanged.
set_binary_mode has no effect on Linux.

 set_binary_mode will throw `exception` when called while child process is running.

## bool close()

Writes (with `timeout`) all pending data to child stdin, 
closes streams and waits (with `timeout`) for child process to stop.
If timeout expires while waiting for child to stop, returns false. Otherwise, returns true.

## void kill()

Terminates child process, without giving it a chance of proper shutdown.

## int exit_code()

Returns exit code from child process. Exit code usually is available only after close. Exception is thrown if chid process
has not yet terminated. Exit code has indeterminable value after `kill`.
