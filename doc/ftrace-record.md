% FTRACE-RECORD(1) Ftrace User Manuals
% Namhyung Kim <namhyung@gmail.com>
% March, 2015

NAME
====
ftrace-record - Run a command and record its trace data

SYNOPSIS
========
ftrace record [*options*] COMMAND [*command-options*]

DESCRIPTION
===========
This command runs COMMAND and gathers function trace from it, saves it into files under ftrace data directory - without displaying anything.

This data can then be inspected later on, using `ftrace-replay` or `ftrace-report` command.

OPTIONS
=======
-b *SIZE*, \--buffer=*SIZE*
:   Size of internal buffer which trace data will be saved.

-f *FILE*, \--file=*FILE*
:   Specify name of output trace data (directory).  Default is `ftrace.dir`.

-F *FUNC*[,*FUNC*,...], \--filter=*FUNC*[,*FUNC*,...]
:   Set filter to trace selected functions only.  See *FILTERS*.

\--force
:   Allow to trace library source rather than executable itself.  When `ftrace-record` finds no mcount symbol (which is generated by compiler) in the executable it quits with an error message since it there's no need to run the program.  However it's possible one is only interested functions in a library, in this case she can use this option so ftrace can keep running the program.

-L *PATH*, \--library-path=*PATH*
:   Load necessary internal libraries from this path.  This is only for testing purpose.

\--logfile=FILE
:   Save log message to this file instead of stderr.

\--no-plthook
:   Do not record library function invocations.  The ftrace traces library calls by hooking dynamic linker's resolve function in the PLT.  One can disable it with this option.

-N *FUNC*[,*FUNC*,...], \--notrace=*FUNC*[,*FUNC*,...]
:   Set filter not trace selected functions only.  See *FILTERS*.

-D *DEPTH*, \--depth=*DEPTH*
:   Set trace limit in nesting level.

\--nop
:   Do not record any functions.  It's a no-op and only meaningful for performance comparison.

\--time
:   Print running time of children in time(1)-style.

FILTERS
=======
The ftrace support filtering only interested functions.  When ftrace is called it receives two types of function filter; opt-in filter with -F/\--filter option and opt-out filter with -N/\--notrace option.  These filters can be applied either record time or replay time.

The first one is an opt-in filter; By default, it doesn't trace anything and when it executes one of given functions it starts tracing.  Also when it returns from the given funciton, it stops again tracing.

For example, suppose a simple program which calls a(), b() and c() in turn.

    $ cat abc.c
    void c(void) {
        /* do nothing */
    }

    void b(void) {
        c();
    }

    void a(void) {
        b();
    }

    int main(void) {
        a();
        return 0;
    }

    $ gcc -o abc abc.c

Normally ftrace will trace all the functions from `main()` to `c()`.

    $ ftrace ./abc
    # DURATION    TID     FUNCTION
     138.494 us [ 1234] | __cxa_atexit();
                [ 1234] | main() {
                [ 1234] |   a() {
                [ 1234] |     b() {
       3.880 us [ 1234] |       c();
       5.475 us [ 1234] |     } /* b */
       6.448 us [ 1234] |   } /* a */
       8.631 us [ 1234] | } /* main */

But when `-F b` filter option is used, it'll not trace `main()` and `a()` but only `b()` and `c()`.

    $ ftrace record -F b ./abc
    $ ftrace replay
    # DURATION    TID     FUNCTION
                [ 1234] |     b() {
       3.880 us [ 1234] |       c();
       5.475 us [ 1234] |     } /* b */

The second type is an opt-out filter; By default, it trace everything and when it executes one of given functions it stops tracing.  Also when it returns from the given funciton, it starts tracing again.

In the above example, you can omit the function b() and its children with -N option.

    $ ftrace record -N b ./abc
    $ ftrace replay
    # DURATION    TID     FUNCTION
     138.494 us [ 1234] | __cxa_atexit();
                [ 1234] | main() {
       6.448 us [ 1234] |   a();
       8.631 us [ 1234] | } /* main */

In addition, you can limit the print nesting level with -D option.

    $ ftrace record -D 3 ./abc
    $ ftrace replay
    # DURATION    TID     FUNCTION
     138.494 us [ 1234] | __cxa_atexit();
                [ 1234] | main() {
                [ 1234] |   a() {
       5.475 us [ 1234] |     b();
       6.448 us [ 1234] |   } /* a */
       8.631 us [ 1234] | } /* main */

In the above example, it prints functions up to 3 depth, so leaf function c() was omitted.  Note that the -D option works with -F option.

SEE ALSO
========
`ftrace`(1), `ftrace-replay`(1), `ftrace-report`(1)