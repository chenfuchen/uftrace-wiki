#basic usage
The uftrace tool consists of several subcommands.  We'll see how to use them with simple examples in this document.

## Getting started
The first subcommand to look at is the `live`.  It's a default subcommand and will be used if you don't give other subcommand when running uftrace (So it's same to run "uftrace xxx" and "uftrace live xxx").  It's basically same as running `record` and then `replay` subcommands in a row.  That means it'd show the output after a program (given on the command line) finished.  Below is the familiar "hello world" program.

    $ cat hello.c
    #include <stdio.h>
    
    int main(void) {
      printf("Hello world\n");
      return 0;
    }

To analyze this program with uftrace, you need to enable the compiler instrumentation.  The gcc provides `-pg` and `-finstrument-functions` options for this.  We prefer to use `-pg` option as it's more light-weight in terms of compiler optimization, but there were some cases it didn't work well.  Currently function arguments are only accessible if it's compiled with the `-pg` option.

    $ gcc -o hello -pg hello.c

Now you can run the hello program with uftrace like below:

    $ uftrace hello
    Hello world
    # DURATION    TID     FUNCTION
       1.337 us [26639] | __monstartup();
       0.897 us [26639] | __cxa_atexit();
                [26639] | main() {
       6.696 us [26639] |   puts();
       7.582 us [26639] | } /* main */

The first line is, as we expect, the output from the hello program.  The following lines are from the uftrace and shows function execution time, process/thread id and function names.  As with function graph tracer in the Linux kernel, the functions are shown as a source code of C programming language.  You can see that the "puts" functions was called during "main" instead of "printf".  It's an optimization of the compiler (gcc) to replace it when the format string has no conversion specifier (e.g. %d) and ends with a new line character.

Note that it showed the "puts" which is a library function you didn't write (the same goes to "__monstartup" and "__cxa_atexit") as well as your own functions in the program.  Of course it doesn't show the internals of the "puts" but it's good to see how long does the "puts" take.  It uses a technique called "PLT hooking" which redirects functions called from a dynamically-linked program.  While it's very powerful (so it's enabled by default), it comes with its overhead too.  So if you don't want to see them and/or don't want to take the overhead, you can use `--no-libcall` option for `uftrace record` to disable it.  As `live` subcommand can take all options the `record` takes, you can run like this:

    $ uftrace --no-libcall hello
    Hello world
    # DURATION    TID     FUNCTION
       7.534 us [26714] | main();

Now you can see only the "main" function - you wrote it only, right? :)   The "Hello world" in the output shows that it actually calls the "puts" function but it didn't show up in the output from uftrace.  Also, you might notice that the "main" function is shown on a single line now.  This is because uftrace shows leaf functions which don't call other functions (from the uftrace's perspective) in the compact format.

## Recording trace data
The uftrace needs to collect trace data in order to analyze the program execution.  Many subcommand in uftrace requires the data to run.  The `record` subcommand saves the trace data in the "uftrace.data" directory by default, and other subcommands use it.  You can use `-d` or `--data` option to use different name.

As you already know, you can give the program name on the command line.

    $ uftrace record pwd

However it'll show the following error message and exit.

    ftrace: /home/namhyung/project/uftrace/cmd-record.c:1271:check_binary
     ERROR: Cannot trace 'pwd': No such file
            Note that ftrace doesn't search $PATH for you.
            If you really want to trace executables in the $PATH,
            please give it the absolute pathname (like /usr/bin/pwd).

This is because it doesn't search directories in the PATH environment variable for you.  In order to run with uftrace, the program needs to be built with the compiler instrumentation.  But programs in the PATH are usually not.  So you need to give the full path to run if you really want to run it with the uftrace.  Let's do this:

    $ uftrace record `which pwd`
    ftrace: /home/namhyung/project/uftrace/cmd-record.c:1307:check_binary
     ERROR: Can't find 'mcount' symbol in the '/usr/bin/pwd'.
            It seems not to be compiled with -pg or -finstrument-functions flag
            which generates traceable code.  Please check your binary file.

It still shows the error message and exits.  This is because the program (pwd) was not built with the compiler instrumentation so there's nothing uftrace can trace.  However you might remember that it can also trace library functions.  If you want to trace calls to library functions even though the program itself was not built to be traced, you can use `--force` option:

    $ uftrace record --force `which pwd`
    /home/namhyung/project/uftrace

The output would be a series of library calls which look like a very simple version of ltrace.  You can use `replay` subcommand to see the recorded program execution.

    $ uftrace replay
    # DURATION    TID     FUNCTION
       1.716 us [26891] | getenv();
       0.994 us [26891] | strrchr();
      60.438 us [26891] | setlocale();
       2.244 us [26891] | bindtextdomain();
       1.152 us [26891] | textdomain();
       0.792 us [26891] | __cxa_atexit();
       1.449 us [26891] | getopt_long();
       4.977 us [26891] | getcwd();
      15.407 us [26891] | puts();
       1.249 us [26891] | free();
       0.985 us [26891] | __fpending();
       0.786 us [26891] | fileno();
       0.800 us [26891] | __freading();
       0.223 us [26891] | __freading();
       3.897 us [26891] | fflush();
       2.635 us [26891] | fclose();
       0.180 us [26891] | __fpending();
       0.166 us [26891] | fileno();
       0.166 us [26891] | __freading();
       0.137 us [26891] | __freading();
       0.283 us [26891] | fflush();
       0.617 us [26891] | fclose();

Basically uftrace will save trace of every single function call (and return).  But it's huge and sometimes impossible to do it for long-running and/or heavy-weight programs.  So uftrace provides a couple of filtering options to control the trace data.  Although some of the filtering can work at later processing (like replay), it'd be better to reduce the amount of data at record time.

One is function-level filters and works on the name of functions.  You can use `-F` or `--filter` option to specify a function to trace.  With this option, all functions called during the function will be recorded and *NO* functions called outside of the function will be recorded.  Also there're `-N` or `--notrace` option to do it in an opposite way.  All functions called during the function will *NOT* be recorded, and all functions called outside of the function will be recorded.  You can also use these options together and more than once.  In the hello world example, you can use it to see function called under "main" only.

    $ uftrace -F main hello
    Hello world
    # DURATION    TID     FUNCTION
                [27132] | main() {
       6.522 us [27132] |   puts();
       8.744 us [27132] | } /* main */

The next is function-depth filter.  You can use `-D` or `--depth` to limit function call depth to be recorded.  Below is apply depth-2 filter on the uftrace itself when replay the hello world program.  The uftrace-pg is built with the compiler instrumentation and the trace data is in hello.data.  The "--" in the command line is to tell the option parser that it's the end of the option.  For simplicity, I omitted the output of replaying hello world:

    $ uftrace -D 2 -- uftrace-pg replay -d hello.data
     ... output from (uftrace-pg replay) ...
    # DURATION    TID     FUNCTION
       2.630 us [27091] | __cxa_atexit();
                [27091] | main() {
      46.440 us [27091] |   argp_parse();
       4.925 us [27091] |   setup_color();
       3.124 us [27091] |   setup_signal();
       2.185 us [27091] |   start_pager();
     444.797 us [27091] |   command_replay();
       0.127 us [27091] |   wait_for_pager();
     513.493 us [27091] | } /* main */

The last type of filter is a time-based one.  It will show functions running longer than the specified time.  Usually short-running functions are out of interest when analyzing program execution so it's useful to remove those function at once.  You can use `-t` or `--time-filter` option like below:

    $ uftrace -t 100us  uftrace-pg replay -d hello.data
     ... output from (uftrace-pg replay) ...
    # DURATION    TID     FUNCTION
                [27154] | main() {
                [27154] |   command_replay() {
                [27154] |     open_data_file() {
     134.230 us [27154] |       read_ftrace_info();
                [27154] |       read_task_txt_file() {
                [27154] |         create_session() {
     321.755 us [27154] |           read_map_file();
                [27154] |           load_symtabs() {
     120.350 us [27154] |             load_symbol_file();
     148.726 us [27154] |           } /* load_symtabs */
     478.026 us [27154] |         } /* create_session */
     535.897 us [27154] |       } /* read_task_txt_file */
     715.152 us [27154] |     } /* open_data_file */
     963.311 us [27154] |   } /* command_replay */
       1.080 ms [27154] | } /* main */

Above show functions run longer than 100 us when replaying the hello world program.  You can see that reading map file and symbol file takes most of time.

In addition, it can also access function arguments and return value.  You can use the `-A` or `--argument` option to access the arguments and likewise, `-R` or `--return` option for return value.  (Currently) it needs to pass function name and argument/return value specifier(s).

    $ uftrace -A puts@arg1/s -R main@retval hello
    Hello world
    # DURATION    TID     FUNCTION
       2.619 us [27961] | __monstartup();
       2.014 us [27961] | __cxa_atexit();
                [27961] | main() {
       8.256 us [27961] |   puts("Hello world");
       9.996 us [27961] | } = 0; /* main */

The first argument of "puts" function is the string so it needs to add "/s" format specifier at the end.  By default integer type is assumed so retval has no format specifier.  For more information please refer the manual page.