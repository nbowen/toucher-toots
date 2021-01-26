# Troubleshooting Local MySQL Installs

MySQL is probably overkill for anything you're doing on your local machine, but there are many reasons you should consider using it anyway. It's a database server with all sorts of options for networking and permissions and fancy things, which is very useful in a "real" shared server context -- which your laptop is not.

in 95% of the cases, if you're not writing or using explicitly MySQL-specific code, almost any database should work for your development and testing on your own personal computer. SQLite, MySQL, PostgreSQL -- Oracle if you're a masochist, etc.

However, there are still those other 5% of cases, and you're going to come across MySQL-specific code from other people, because we're all lazy. Or maybe on principle you just want to learn how to get some code connected to your own MySQL server for future reference.

Regardless of why you want to do such a crazy thing, this guide is designed to help you do just that.

## Installing Your MySQL Server

It is beyond the scope of this tutorial, because the chances are good that you came here after installing and failing to use your server. But we'll cover the basics:

### MacOS (Already Running)

Let's start by seeing if you already have MySQL running in some way or another. I'll do this:

`$ ps axwu | grep mysql`

and I will find that yes, I have it running:
```
myuser           15609   0.0  0.7  4810760 121404   ??  S    Fri01PM   1:02.60 /usr/local/Cellar/mariadb/10.5.8/bin/mariadbd --basedir=/usr/local/Cellar/mariadb/10.5.8 --datadir=/usr/local/var/mysql --plugin-dir=/usr/local/Cellar/mariadb/10.5.8/lib/plugin --log-error=/usr/local/var/mysql/MyComputerName.local.err --pid-file=/usr/local/var/mysql/MyComputerName.local.pid
myuser           15542   0.0  0.0  4291404    600   ??  S    Fri01PM   0:00.02 /bin/sh /usr/local/Cellar/mariadb/10.5.8/bin/mysqld_safe --datadir=/usr/local/var/mysql --pid-file=/usr/local/var/mysql/MyComputerName.local.pid
```

If that's the case, for our purposes, we won't uninstall and reinstall -- we'll use what we've got.

### MacOS (Not Running)

If MySQL is not already running on your Mac, chances are you need to install it -- but it's possible you've installed it and not set it up to run automatically.

We'll assume you're using HomeBrew for this.

```
 $ brew search maria
 ```
```
==> Formulae
mariadb ✔                  mariadb@10.2
mariadb-connector-c        mariadb@10.3
mariadb-connector-odbc     mariadb@10.4
mariadb@10.1
==> Casks
maria                      navicat-for-mariadb
```

I can see here that I do have mariadb installed, given that checkmark.

If I did not have it installed, a simple `brew install mariadb` would get it installed.

If this test shows that it is installed, but the earlier test showed it was not running, then it's just a matter of starting it:

### MacOS (Installed but Not Running)

The easiest way to get it started and begin playing with it, in this case, is to first make sure it's not running:

``` ps axwu | grep maria```
```
myuser            7945   0.0  0.0  4268300    708 s001  S+    2:09AM   0:00.00 grep maria
```

And then make it run:

```$ mysql.server start```

```Starting MariaDB
210126 02:09:26 mysqld_safe Logging to '/usr/local/var/mysql/MyComputerName.local.err'.
210126 02:09:26 mysqld_safe Starting mariadbd daemon with databases from /usr/local/var/mysql
 SUCCESS! 
```
This will start it "for now", but not on every startup. That's a different thing for another time, but now we're set for the next steps.

The proof that it is now running comes from the same command as above:

```
$ ps axwu | grep maria
```
```
myuser            8021   0.0  0.6  4804776  96612 s001  S     2:09AM   0:00.27 /usr/local/Cellar/mariadb/10.5.8/bin/mariadbd --basedir=/usr/local/Cellar/mariadb/10.5.8 --datadir=/usr/local/var/mysql --plugin-dir=/usr/local/Cellar/mariadb/10.5.8/lib/plugin --log-error=/usr/local/var/mysql/MyComputerName.local.err --pid-file=/usr/local/var/mysql/MyComputerName.local.pid
myuser            7954   0.0  0.0  4272972   1264 s001  S     2:09AM   0:00.02 /bin/sh /usr/local/Cellar/mariadb/10.5.8/bin/mysqld_safe --datadir=/usr/local/var/mysql --pid-file=/usr/local/var/mysql/MyComputerName.local.pid
```

## Connecting to Your MySQL Server

Whatever client you're going to use to connect to your MySQL server will connect over a *socket* -- of some kind.

You use sockets all the time, and if you know something about them, chances are good that you're thinking of TCP sockets. MySQL does use TCP sockets, but also it's happy to make you think about Unix Sockets.

This is probably the most subtly infuriating thing about setting up MySQL.

### Pro-Tip Interlude: Sockets

In the world of networking, a "socket" is just a thing that your side of a connection can read and write in order to communicate with another process.

When you hit a website in your browser, your browser is opening a socket to the appropriate web server in order to send requests and receive responses.

That's a TCP socket -- and those are identified by their port number at a specific IP address.

There's also such a thing as a "Unix socket" (or "Unix domain socket") -- it's a very similar thing, but specifically used for two processes on the same machine to communicate.

Instead of a port number, it shows up looking like a file in some directory, often in /tmp.

This sounds crazy, and it kind of is, but in grand Unix tradition, if you want a way to read and write data from your process to another process, doing it through a thing that "looks like" a file is par for the course.

Now that you have a MySQL server running, depending on how you got it there, you can do some quick (but non-comprehensive) tests to see if what kinds of sockets it's using to listen for connections.

**Q**: Is MySQL listening for TCP connections?

**A**: By default, MySQL listens on TCP port 3306 -- if it's set up to listen on TCP ports at all. You can check this by testing a connection:

```$ nc -z -v localhost 3306```

```
Connection to localhost port 3306 [tcp/mysql] succeeded!
```

**Q**: Is MySQL listening for Unix socket connections?

**A**: Probably, and there are a few ways to check this. Given MacOS and HomeBrew defaults, you can just look for the presence of `/tmp/mysql.sock` -- which shows up as a "file", but as described above, it's really a socket (see the 's' in the output of the file listing below).

```☭ ls -l /tmp/mysql.sock ```

```
srwxrwxrwx  1 myuser  wheel  0 Jan 26 02:09 /tmp/mysql.sock
```

The fact that MySQL is so "good" at listening for unix socket connections is part of why this is so convoluted. But now that you're an expert in TCP and Unix domain sockets, we'll go back to MySQL-specific setup.

*TODO: describe what to do or not to do if either of those sockets is not working*

### The Localhost Crapshow

The MySQL "monitor" (just read that as "the command line client) is what you get when running the `mysql` command from your shell; you use it to run arbitrary SQL against your databases, or to import dumps from other MySQL databases, or to edit the users and permissions on your databases.

Once it's working, it's very useful for all of those things. Until it's working, it tries very hard to make life easier for you, and about 50% of the time, it fails epically.

The two main ways in which it "helps" unhelpfully are:
* Assumptions about whether you mean to connect over a Unix socket or a TCP socket
* Assumptions about your MySQL username based on your Unix username

The latter is the easy one -- if you're running `mysql` on your command line and you don't specify the -u option for your username, the mysql client guesses that you want to use your unix username.

This is reasonable, but it can be really confusing if you don't notice it while you're trying to test connections from some app that runs externally with a whole different db/username for its specific purpose. 

The relevant official documentation for this craphow can be found at: [Connecting to the MySQL Server Using Command Options](https://dev.mysql.com/doc/refman/8.0/en/connecting.html)

Perhaps the key sentence in that document, if MySQL is not doing what you expect, is this:

> On Unix, MySQL programs treat the host name `localhost` specially, in a way that is likely different from what you expect compared to other network-based programs: the client connects using a Unix socket file. The [`--socket`](https://dev.mysql.com/doc/refman/8.0/en/connection-options.html#option_general_socket) option or the `MYSQL_UNIX_PORT` environment variable may be used to specify the socket name.

In short, if your intention is to connect with a TCP socket (say from a pure-java or pure-python driver), you're not really proving that it will work just by using the MySQL command line monitor. More admissions from that document include:

> To ensure that the client makes a TCP/IP connection to the local server, use [`--host`](https://dev.mysql.com/doc/refman/8.0/en/connection-options.html#option_general_host) or `-h` to specify a host name value of `127.0.0.1` (instead of `localhost`), or the IP address or name of the local server. You can also specify the transport protocol explicitly, even for `localhost`, by using the [`--protocol=TCP`](https://dev.mysql.com/doc/refman/8.0/en/connection-options.html#option_general_protocol) option.

So, when testing from the MySQL command-line monitor, even with usernames/passwords and fine-grained permissions we have not discussed, you haven't proven it will work from your own code. To do that, you'd have to be very careful about the command line options that tell it what kind of socket you're using, and what username you're using, and so on.

### Pro-Tip Interlude: How to Cheat

Putting that all together, if you want to be 100% sure that you're connecting from the command line the way your non-unix-command-line code will connect over TCP, just refuse to rely on the defaults.

Always use the `-h` option to specify the hostname (and if it's the local host, don't say 'localhost', say 127.0.0.1). Always use the `-u` option to specify which MySQL user is connecting, so you're not relying on which Unix user happens to be running the code. And then, use the `-p` option as needed for that user's password.

## MySQL Permissions

If you've made it this far, hopefully you have a good start on reliably connecting to MySQL servers from MySQL command-line clients with an eye towards Unix sockets vs TCP sockets.

But you may not be there yet. MySQL looks at permissions based on a combination of username and hostname. And after the above dissection of how MySQL sees hostnames, you can see how this could be complicated.

In other words, it's possible to tell MySQL that the user 'jimbob' is allowed to access the table 'userdata' when connecting from 'localhost', but not when connecting from outside the local network. And so on.

And we know that 'localhost' is a loaded term in the MySQL world, so the plot thickens.

*TODO: Explain MySQL Permissons*
