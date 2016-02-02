A grep-like DSEE Access Log Parser

The Whizbang Directory Server Access Log Parser

We often have to use multiple pipelines to do things like:

Find and print all operation entries (including the request line) that have a certain characteristic, such as a result code or etime.
Find and print all connections that have certain characteristics, such as a specific source IP or BIND DN.
Using pipelines of grep, awk, and xargs for this works functionally, but can result in many passes through the access log. When the access log is large, this can take a very long time. This script attempts to perform such operations with only one pass through the log.

The caveat is that the script must store the contents of an operation or connection in a data structure until the script determines whether to print the contents or not. So, in a worst case scenario, if connection output has been requested, and the script is unable to determine whether or not to print any of the connections in the log until the very end, the perl process may have the entire log stored in memory.

Usage

``` lgrep [-O ] [-S [DD/Mmm/yyyy:]hh:mm:ss] [-E [DD/Mmm/yyyy:]hh:mm:ss] [-vidh] expression [file] -O Output type - one of connection, operation, sort, stat, or echo - defaults to operation -v negative match (unused for sort, stat or echo) -i case-insensitive match (unused for sort, stat or echo) -I interval for stat reporting, default is 60 -S Start datetime for log processing -E End datetime for log processing -d debug script logging -h display this help expression - Expression to match (unused for sort, stat and echo) file - Access log to parse - if missing, we assume STDIN

Notes:

-O sort produces a sorted, delimited output of connections, only after EOF on the
   input access log.  This means that all log input is stored in memory at the
   time printing starts.  It also means that there is no rolling output such as
   that produced by -O connection or -O operation.  So whereas tail -f piped to
   -O connection or -O operation produces output, tail -f access piped to -O sort
   will never produce output.

-E and -S datetime arguments can be abbreviated by omitting the date portion.  In
   this case, the first line of the log will be used to generate the datetime constraints.
   This means if you have a log that starts as 23:50 and you want to grab a time range from
   the following morning, you will need to specify the datetime in full on the command line.

-O stat produces the following statistics:
    date_end - date at the end of the statistics gathering interval
    time_end - timestamp at the end of the statistics gathering interval
    conn_opn - number of connections opened during the statistics gathering interval
    conn_cls - number of connections closed during the statistics gathering interval
    tot_ops - number of operations initiated during the statistics gathering interval
    tot_res - number of results sent during the statistics gathering interval
    tot_bind - number of BIND operations initiated during the statistics gathering interval
    tot_srch - number of SRCH operations initiated during the statistics gathering interval
    tot_add - number of ADD operations initiated during the statistics gathering interval
    tot_mod - number of MOD operations initiated during the statistics gathering interval
    tot_del - number of DEL operations initiated during the statistics gathering interval
    etime_av - average etime of results sent during the statistics gathering interval
    etime_mx - maximum etime of results sent during the statistics gathering interval
    un_idx_s - number of unindexed search results completed during the statistics gathering interval
    nentries - number of entries sent during the statistics gathering interval
Examples:

1) Print all operations from the large log file "access.foo" that have the expression " BIND " in them and also have the expression " err=32 " in them. Typically, this means all BIND operations that failed with err=32.

cat access.foo | lgrep " BIND " | lgrep " err=32 "

Note that while we are making a second pass through the output of the first iteration, an equivalent pipeline of grep, awk, and/or xargs would require many passes through the original, large log file.

2) Find all operations from the log file "access.foo" that have the expression " ADD ", " MOD ", or " DEL " in them, with an LDAP result code other than 0 or 50.

lgrep " ADD | MOD | DEL " access.foo | lgrep -v " err=0 | err=50 "

3) Find all operations from the log file "access.foo" that have the expression " etime=" in them, where is not equal to zero. Typically, this returns all operations with etime of 1 second or greater.

lgrep " etime=[^0]" access.foo

Note that this form can be significantly more efficient in how much memory it uses compared to the -v form.

4) Watch a live access log for wrong password logins.

tail -f access | lgrep " err=49 "

5) Grab log output between two datetimes and echo it

lgrep -O echo -S 15/Nov/2008:19:00:00 -E 15/Nov/2008:19:59:59 access.foo

6) Examine workload statistics in one-hour increments

lgrep -O stat -I 3600 access.foo

```

Bugs

When a start datetime is specified for stats output, the stats intervals should all start at that time and subsequent intervals should be synchronized with that start time based on the specified sampling interval, regardless of whether there are any log lines in the sampling interval. Currently if a start time before the beginning of the log is specified, the sampling interval starts at the datetime contained in the first line of the log.

Stats output has been reported to be incorrect in some cases. Specific test case/s are needed to fix.

Enhancements

Print the stats header periodically, to make the columns more easily identifiable.

Make the stats output configurable, like the "-o" output of ps. Right now it's a bit wide for a lot of terminals. I'd also like to add more types of stats once the option exists to pick the stats produced.

Add an "offset mode" to the stats output that backtraces operation RESULTs to their originating request. Right now, we only see RESULT related stats (like unindexed searches) in the time interval the RESULT is obtained. Sometimes, it's more useful to see various RESULT stats based on the time the request was logged. Backtracking when an unindexed search started (and thus began to potentially affect performance) is a process that could be done in the stats output.

Add a "pending" operations function to the stats output. In many cases, pending operations can affect performance without ever leaving a RESULT line in the access log, since a long-running unindexed search can easily be abandoned before it completes. If we know the time a pending operation was submitted, we can track long running queries that have not produced a result.
