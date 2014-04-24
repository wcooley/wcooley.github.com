[[TableOfContents]]

= Shell Hacks =

This page is for quick shell hacks. More complete ''bash''-scripting documentation is available from the [http://tldp.org/LDP/abs/html/ Advanced Bash-Scripting Guide].

In case it isn't obvious, these are all only tested on ''bash''. ''ksh'' might work but I only rarely use it and I don't test in it.

== Listing Non-System Users ==

{{{awk}}} is a word-splitter.  {{{/etc/passwd}}} is a colon-delimited list of words.  Ergo:

{{{
$ awk -F: '{if ($3 >= 500) { print $1 }}' /etc/passwd
}}}

Replace ''500'' with the beginning UID for non-system users for your system.  Red Hat uses 500; Solaris uses 1000.  If you're using NIS, LDAP, or other NameServiceSwitch back-end, use the {{{getent}}} command (which, conveniently outputs the same format as {{{/etc/passwd}}}):

{{{
getent passwd | awk -F: '{if ($3 >= 500) {print $1}}'
}}}

Neither of these commands will work on AIX, but on AIX you've got {{{lsusers}}} already.  Note that you can also list only the system users by reversing the comparison operator.  You will likely have a user ''nfsnobody'' that is UID 65534 (which corresponds to -1 in signed 16-bit integers) which is also a system user.

== Display Meat of Config File ==

This removes empty lines and lines that start with a '#', usually used as a comment character.

{{{
grep -vE '^($|#)' <foo.conf>
}}}

This one is better; it strips comments and whitespace-only lines, whereas the previous only strips comments starting at the beginning of the line and blank lines:

{{{
alias nocomment="sed -e 's/\([^#]*\)#.*$/\1/; /^[[:space:]]*$/d; /^#/d;'"
}}}

== Find Empty Directories ==

This is the magic for {{{find}}} that finds empty directories in the current working directory.

{{{
find . -empty -maxdepth 1 -type d
}}}


== Hourly Statistics from Log Files ==

This extracts the hour from the syslog timestamps and shows how many log entries occured in each hour.  This is most useful if you pre-process <logfile>, or you remove log file and feed it with a pipe.

{{{
awk '{print $3}' <logfile> |awk -F: '{print $1 ":00"}' |sort -n |uniq -c
}}}


== Batch Processing ==

If you've ever wanted to run a number of jobs at the same time, but had too many jobs to run them all at the same time, you can batch the jobs, so you can run ''N'' jobs in parallel at a time.  Here's what it would look like:

{{{
batchsize=5
batch=0
                                                                                
for x in $(seq -w 1 30); do
    # Start background task here
    (sleep 1; echo $x) &
                                                                                
    batch=$(($batch + 1))
                                                                                
    if [ $((${batch} % ${batchsize})) -eq 0 ]; then
        wait
        echo "End of batch: $x"
    fi
done
                                                                                
echo "Processed ${batch} items"

}}}

== Redirecting I/O Script-wide ==

The {{{exec}}} command can be used to redirect I/O script-wide. If {{{exec}}}
is not given a command to execute, it applies whatever I/O redirections are specified to the current shell itself.

For example, instead of appending '''{{{>/tmp/logfile}}}''' to capture the
output of every command to a file, use this to redirect ''stdout'':{{{
exec >/tmp/logfile
}}}

To direct ''stderr'' to ''stdout'', use this: {{{
exec 2>&1
}}}

Or the reverse, ''stdout'' to ''stderr'': {{{
exec 1>&2
}}}

=== Logging and Monitoring with Tee ===

Sometimes you want to capture all standard output to a log file while
monitoring the output yourself. We can use the {{{exec}}} I/O redirection to do
this also along with [http://tldp.org/LDP/abs/html/process-sub.html process substitution]:{{{
exec > >(tee -a ${0##*/}.log)
}}}

If you wanted to redirect ''stderr'' to a different file: {{{
exec 2> >(tee -a ${0##*/}.err)
}}}

== Printing and Logging to Syslog ==

Let's make use of the ''{{{exec}}}'' to log the output of the script to syslog instead of a file. We have the ''{{{logger}}}'' command, which will take a message either on the command line or from standard input and write it to syslog. The ''{{{-s}}}'' option instructs it to also write the log messages to standard error. And if we did not know about [http://tldp.org/LDP/abs/html/process-sub.html process substitution], we could use a named FIFO to have ''{{{logger}}}'' read from and standard output and error to write to.

{{{
prog=${0##*/}

if [[ -z "$FIFO" ]]; then
    export FIFO="/tmp/${prog}.$RANDOM"
fi

if [[ ! -e "$FIFO" ]]; then
    mkfifo -m 666 "$FIFO"
    trap "rm $FIFO" ERR EXIT
    logger -t "${prog}:" -i -s <"$FIFO" &
fi

exec >"$FIFO" 2>&1
}}}

== Getting the Script Name ==

I used to use {{{prog=$(basename $0)}}} to get the script's basename (which is good for help output, temp files, etc).  However, I've picked up a tip from SUSE's init scripts which uses the parameter expansion available in bash and ksh:{{{
prog=${0##*/}
}}}

It's a little more succinct (if perhaps obscure) and obviates an exec.

== Trace-visible Comments ==

Sometimes when debugging shell scripts, it's nice to be able to tell where you are.  It's not always clear, even when running in trace-mode ({{{set -x}}}).  Usually this is accomplished by inserting {{{echo}}} statements that tell you where you are.  When you're finished debugging, you have to remove or comment-out these (and how many times have you discovered one you forgot to remove?)  Instead, you can use the colon-builtin to provide "traceable" comments.  The colon command is one of those little-used commands that does nothing other than provide a true value, so it's often used in {{{while}}} loops that you expect to exit in ways other than the conditional or to ensure that some particular line always returns true (frequently seen in RPM spec files, for example).  The colon command ignores any parameters given to it, so you can add a comment after the colon and it will be visible when run with tracing turned on and invisible otherwise.  Note that it is actually a command and not a comment so it cannot be used exactly as a comment would, such as at the end of a statement (although you can use the semi-colon statement separator, as you'd expect with any other).  {{{true}}} and {{{false}}} also ignore any supplied parameters, but it seems less obvious than not.

Here's an example script:{{{
#!/bin/bash
: This is an invisible traced comment
set -x
# This is an untraced comment
: This is a traced comment
}}}

Which produces the following output:{{{
$ ./test.sh
+ : This is a traced comment
}}}

== Redirecting To stderr ==

Under {{{bash}}}, {{{/dev/stderr}}} is an internally-recognized device that, when redirected to, writes the output to ''stderr''.  Under Linux and Solaris, {{{/dev/stderr}}} exists as a character device for the current process's ''stderr''[[FootNote(Well, it's actually a symlink to a character device, but the effect is the same.)]].  So writing to {{{/dev/stderr}}} in a shell that didn't provide an internal {{{/dev/stderr}}} should work just fine.  But that's not true with {{{ksh}}} on AIX.  With {{{ksh}}}, one might be inclined to use "{{{print -u 2}}}", but that doesn't work in {{{bash}}}.

If you have to worry about portability, use the obscure redirection to redirect ''stdout'' to ''stderr'':{{{
$ echo "this writes to stderr" 1>&2
}}}

Test it: {{{
$ echo "this writes to stderr" 1>&2 |cat >/dev/null
this writes to stderr
}}}

(Usually one redirects ''stderr'' to ''stdout'' using "{{{2>&1}}}"--this is just the reverse.)

== Cleaning Up on Exit ==

You can automatically clean up on exit using the "{{{trap}}}" built-in command. This command allows commands to be run when signals are received or on certain other condition, such as exit.

Let's say you have a temporary file in the shell variable ''{{{$tmpfile}}}'' that should be removed on exit.
{{{
trap "rm -f $tmpfile" EXIT
}}}

== Function Template: ''usage'' ==

{{{
usage() {
    # Default to 0
    local exitval="${1:-0}"

    if [[ $exitval -eq 1 ]]; then
        # Redirect stdout to stderr
        exec 1>&2
    fi

    echo "Usage: ${0##*/} [-h] [-x]"
    echo "Do something or other."
    echo "  -x          - Set 'x' to true."
    echo "  -h          - show this help screen."

    exit "$exitval"
}

}}}

== Joining Line-Delimited Data ==

The Perl ''join'' function is nice, because you can join a list to create a scalar value with a delimiter. For example, if I have a line-delimited list of names: {{{
bob
joe
mary
}}} I might want to have a comma-separated list, which I can do with ''sed'': {{{
$ sed -e ':a; N; s/\n/,/; ta' <<EOF
> bob
> joe
> mary
> EOF
bob,joe,mary
}}}

In general, where ''DELIM'' is your desired delimiter: {{{
sed -e ':a; N; s/\n/DELIM/; ta'
}}}

== Generate Sequence of Integers without Using 'seq' ==

Brace expansion can be use with a range: {{{
$ echo {1..10}
1 2 3 4 5 6 7 8 9 10
}}}

Also supports a non-1 increment: {{{
$ echo {1..10..2}
1 3 5 7 9
}}}

Descending increment: {{{
$ echo {10..1}
10 9 8 7 6 5 4 3 2 1
}}}

Zero-padding: {{{
$ echo {01..10}
01 02 03 04 05 06 07 08 09 10
}}}

== Iterate Over $PATH ==

Use the pattern-substitution parameter expansion. Note that you must also not quote the variable (which I usually do as a good practice): {{{
$ for p in ${PATH//:/ }; do echo $p; done
/usr/sbin
/usr/bin
/sbin
/bin
}}}

=== Improved pathmunge ===

Red Hat's {{{/etc/profile}}} defines a shell function called {{{pathmunge}}} to conditionally add directories to ''{{{$PATH}}}''. The problem with this implementation is that it runs {{{egrep}}}, which involves forking a process and incurs a modicum of overhead. Generally that's not a big deal--the overhead is minimal on modern processors. But if the host is in a state of distress due to swapping or a fork-bomb or such, these extra processes become a burdensome overhead.

Here's my implementation, which also adds a ''force'' parameter (which unfortunately means it is not idempotent) and changes the ''after'' to ''before'', since ''after'' is what I usually want.

{{{
pathmunge () {
    newpath="$1"

    if [[ ! -d "$newpath" ]]; then return 1; fi
    if [[ "$2" = "force" || "$3" = "force" ]]; then force=1; else force=0; fi

    if [[ $force -ne 1 ]]; then
        for p in ${PATH//:/ }; do
            if [[ "$p" = "$newpath" ]]; then
                : $newpath exists - aborting early
                return 1
            fi
        done
    fi

    : adding $newpath - $LINENO
    if [[ "$2" = "before" ]] ; then
        PATH="$newpath:$PATH"
    else
        PATH="$PATH:$newpath"
    fi
}
}}}