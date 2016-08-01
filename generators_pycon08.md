# **Notes on David Beazley's Talk on Generator Tricks For Systems Programmers Pycon'2008**
***********

Note: These notes use Python 3, the talks were in Python 2.

The Mission Statement
=======

- Practical uses of generators
- Focus on **systems programming**
- Which loosely includes files, file systems, parsing, networking, threads etc.
- To provide some more compelling examples of using generators

PART 1: Introduction to Iterators and Generators
=====================

Iterators
----

Any object that supports `iter()` and `__next__()` methods
is said to be iterable

    items = [1, 4, 5]
    it = iter(items)
    next(it) # 1
    next(it) # 4
    next(it) # 5
    next(it) # StopIteration

A sample iteration example

    for x in obj:
        # statements

underneath the covers, the for loop works similar to

    _iter = iter(obj)
    while 1:
        try:
            x = next(_iter)
        except StopIteration:
            break
        # statements
        ...

Sample Countdown Iterator
-----

    class Countdown:
        def __init__(self, start):
            self.count = start
        def __iter__(self):
            return self
        def __next__(self):
            if self.count <= 0:
                raise StopIteration
            r = self.count
            self.count -= 1
            return r

Sample Countdown Generator
----

    def countdown(n):
        print("Counting down from", n)
        while n > 0:
            yield n
            n -= 1

Calling a generator function creates a generator object. However, it does
**not** start running the function i.e. doesn't print the first statement

    >>> x = countdown(10)       # Notice no output was produced on calling
    >>> x
    <generator object at posn>
    >>> next(x):
    Counting down from 10       # Function stats executing here
    10
    >>> next(x)                 # yield producues a value, but suspends the function
    9
    >>> next(x)                 # function resumes on next call to next()
    8
    ...
    >>> next(x)
    1
    >>> next(x)
    StopIteration

A generator is a *one-time-operation*. You can iterate over the generated data
once, but if you want to do it again, you have to call the generator function
again.

Generator Expressions
--------

    >>> a = [1,2,3,4]
    >>> b = (2*x for x in a)
    >>> b
    <generator object <genexpr>>
    >>> for i in b: print(b, end = ' ')
    ...
    2 4 6 8

This loops over a sequence of items and applies an operation to each items
However, results are produced one at a time using a generator

#### Generator Expressions

- Does not construct a list
- Only useful purpose is iteration
- Once consumed, can't be reused

#### General Syntax

    (expression for i in s if cond1
                for j in t if cond2
                ...
                if condfinal)

#### What it means

    for i in s:
        if cond1:
            for j in t:
                if cond2:
                    ...
                    if condfinal: yield expression

********************

PART 2 - Processing Data Files
==============

Programming Problem
-----

> **Find out how many bytes of data were transferred by summing up the last column of data in this Apace web server log**

    81.107.39.38 - ... "GET /ply/ HTTP/1.1" 200 7587
    81.107.39.38 - ... "GET /favicon.ico HTTP/1.1" 404 133
    81.107.39.38 - ... "GET /ply/bookplug.gif HTTP/1.1" 200 23903
    81.107.39.38 - ... "GET /ply/ply.html HTTP/1.1" 200 97238
    81.107.39.38 - ... "GET /ply/example.html HTTP/1.1" 200 2359
    66.249.72.134 - ... "GET /index.html HTTP/1.1" 200 444

#### The log file

- Each line of the log looks like this:

        81.107.39.38 - ... "GET /ply/ply.html HTTP/1.1" 200 97238

- The number of bytes is the last column

         bytestr = line.rsplit(None,1)[1]

- It's either a number of a missing value (-)

        81.107.39.38 - ... "GET /ply/ HTTP/1.1" 304 -

- Converting the value

        if bytestr != '-':
            bytes = int(bytestr)

#### A Non-Generator Solution

- A simple for loop

        wwwlog = open("access-log")
        total = 0
        for line in wwwlog:
            bytestr = line.rsplit(None, 1)[1]
            if bytestr != '-':
                total += int(bytestr)

        print("Total", total)

#### A Generator Solution

    wwwlog = open("access-log")
    bytecolumn = (line.rsplit(None, 1)[1] for line in wwwlog)
    bytes = (int(x) for x in bytecolumn if x != '-')
    print("Total", sum(bytes))

This can be thought of as a data processing pipeline.

> access-log -> **wwwlog** -> **bytecolumn** -> **bytes** -> **sum()** -> total**

- Each step is defined by iteration/generation
- At each step of the pipeline, we declare an operation that will *be applied to the entire input stream*
- Instead of focusing on the problem at a line-by-line level, you just break it down into big operations that operate on the whole file
- The glue that holds the pipeline together is the iteration that occurs at each step
- The calculation is being driven by the last step
- The `sum()` function is consuming values being pushed through the pipeline (via `next()` calls)

**Performance/Thoughts**

- Was 5% faster than the iteration version
- At no point in our generator solution did
we ever create large temporary lists
- Thus, not only is that solution faster, it can
be applied to enormous data files
- The generator solution was based on the
concept of pipelining data between
different components
- What if you had more advanced kinds of
components to work with?
- Perhaps you could perform different kinds
of processing by just plugging various
pipeline components together

**Sounds Familiar**

- The Unix philosophy
- Have a collection of useful system utils
- Can hook these up to files or each other
- Perform complex tasks by piping data

******

PART 3 - Fun with Files and Directories
===========

Programming Problem
-----

> **You have hundreds of web server logs scattered across various directories. In additional, some of the logs are compressed. Modify the last program so that you can easily read all of these logs**

    foo/
     access-log-012007.gz
     access-log-022007.gz
     access-log-032007.gz
     ...
     access-log-012008
    bar/
     access-log-092007.bz2
     ...
     access-log-022008

#### os.walk()

- A very useful function for searching the file system

        import os
        for path, dirlist, filelist in os.walk(topdir):
            # path : Current directory
            # dirlist : List of subdirectories
            # filelist : List of files
            ...


-  This utilizes generators to recursively walk
through the file system

#### find

- Generate all filenames in a directory tree
that match a given filename pattern

        import os
        import fnmatc
        def gen_find(file_pattern,top):
            for path, dirlist, filelist in os.walk(top):
                for name in fnmatch.filter(filelist,file_pattern):
                    yield os.path.join(path,name)

- Examples

        pyfiles = gen_find("*.py","/")
        logs = gen_find("access-log*","/usr/www/")

#### A File Opener

- Open a sequence of filenames

        import gzip, bz2
        def gen_open(filenames):
            for name in filenames:
                if name.endswith(".gz"):
                    yield gzip.open(name)
                elif name.endswith(".bz2"):
                    yield bz2.BZ2File(name)
                else:
                    yield open(name)

- This is interesting.... it takes a sequence of filenames as input and yields a sequence of open file objects

#### cat

- Concatenate items from one or more source into a single sequence of items

        def gen_cat(sources):
            for s in sources:
                for item in s:
                    yield s

- Example

        lognames = gen_find("acces-log*", "/user/www")
        logfiles = gen_open(lognames)
        loglines = gen_cat(logfiles)

#### grep

- Generate a sequence of lines that contain a given regular expression

        import re
        def gen_prep(pat, lines):
            patc = re.compile(pat)
            for line in lines:
                if patc.search(line):
                    yield line

- Example

        lognames = gen_find("acces-log*", "/user/www")
        logfiles = gen_open(lognames)
        loglines = gen_cat(logfiles)
        patlines = gen_prep(pat, loglines)

#### Working Example

- Find out how many bytes transferred for a specific pattern in a whole directory of logs

        pat = r"somepattern"
        logdir = "/some/dir/"
        filenames = gen_find("access-log*",logdir)
        logfiles = gen_open(filenames)
        loglines = gen_cat(logfiles)
        patlines = gen_grep(pat,loglines)
        bytecolumn = (line.rsplit(None,1)[1] for line in patlines)
        bytes = (int(x) for x in bytecolumn if x != '-')
        print "Total", sum(bytes)

#### Notes

- Generators decouple iteration from the code that uses the results of the iteration
- In the last example, we're performing a calculation on a sequence of lines
- It doesn't matter where or how these lines are generated
- Thus, we can plug any number of components together up front as long as they eventually produce a line sequence

*********

PART 4 - Parsing and Processing Data
====

Programming Problem
----

> **Web server logs consist of different columns of data. Parse each line into a useful data structure that allows us to easily inspect the different fields**

    81.107.39.38 - - [24/Feb/2008:00:08:59 -0600] "GET ..." 200 7587
    # format: host referrer user [datetime] "request" status bytes

#### Parsing with Regex

- Let's route the lines through a regex parser

        logpats = r'(\S+) (\S+) (\S+) \[(.*?)\] '\
                  r'"(\S+) (\S+) (\S+)" (\S+) (\S+)'
        logpat = re.compile(logpats)
        groups = (logpat.match(line) for line in loglines)
        tuples = (g.groups() for g in groups if g)

- This generates a sequence of tuples

        ('71.201.176.194', '-', '-', '26/Feb/2008:10:30:08 -0600',
        'GET', '/ply/ply.html', 'HTTP/1.1', '200', '97238')

#### Tuples to Dictionaries

- David doesn't like data processing on tuples:

  * They are immutable so you can't Modify
  * To extract specific fields, you have to remember the column number - which is annoying if there are a lot of columns
  * Existing code breaks if you change the number of fields


- Turn tuples to dictionaries

        colnames = ('host', 'referrer', 'user', 'datetime',
                    'method', 'request', 'proto', 'status', 'bytes')
        log = (dict(zip(colnames, t)) for t in tuples)

- which generates a sequence of named fields

        {'status': '200',
        'proto': 'HTTP/1.1',
        'referrer': '-',
        'request': '/ply/ply.html',
        'bytes': '97238',
        'datetime': '24/Feb/2008:00:08:59 -0600',
        'host': '140.180.132.213',
        'user': '-',
        'method': 'GET'}

#### Field Conversion

- You might want to map specific dictionary fields through a conversion function (e.g. `int()`, `float()`)
- Map specific dictionary fields through a function

        def field_map(dictseq, name, func):
            for d in dictseq:
                d[name] = func(d[name])
                yield d

- Example: Convert a few field values

        log = field_map(log, "status", int)
        log = field_map(log, "bytes",
                        lambda s: int(s) if s != '-' else 0)

- Creates dictionaries of converted values

        {'status': 200,                            # Note conversion
        'proto': 'HTTP/1.1',
        'referrer': '-',
        'request': '/ply/ply.html',
        'bytes': 97238,                            # Note conversion
        'datetime': '24/Feb/2008:00:08:59 -0600',
        'host': '140.180.132.213',
        'user': '-',
        'method': 'GET'}

#### Code so Far

    lognames = gen_find("access-log*","www")
    logfiles = gen_open(lognames)
    loglines = gen_cat(logfiles)
    groups = (logpat.match(line) for line in loglines)
    tuples = (g.groups() for g in groups if g)

    colnames = ('host','referrer','user','datetime','method',
                'request','proto','status','bytes')

    log = (dict(zip(colnames,t)) for t in tuples)
    log = field_map(log,"bytes",
                    lambda s: int(s) if s != '-' else 0)
    log = field_map(log,"status",int)

#### Getting Organized

- As a processing pipeline grows, certain parts of it may be useful components on their own

<center> <img src = "getting_organized.png"> </center>
<br>
- A series of pipeline stages can be easily encapsulated by a normal Python function

#### Packaging

- Extending from above, to make it more sane, you may want to package parts of the code into functions.

- **Example:** multiple pipeline stages inside a functions

        def lines_from_dir(filepat, dirname):
            names = gen_find(filepat, dirname)
            files = gen_open(names)
            lines = gen_cat(files)
            return lines

- This is now a general purpose component that can be used as a single element in other pipelines

- **Example:** Parse an Apache log into dicts

        def apache_log(lines):
            groups = (logpat.match(line) for line in lines)
            tuples = (g.groups() for g in groups if g)

            colnames = ('host', 'referrer', 'user', 'datetime', 'method',
                        'request', 'proto', 'status', 'bytes')

            log = (dict(zip(colnames, t)) for t in tuples)
            log = field_map(log, "bytes",
                            lambda s: int(s) if s != '-' else 0)
            log = field_map(log, "status", int)

            return log

#### Example Use

    lines = lines_from_dir("access-log*", "www")
    log = apache_log(lines)

    for r in log:
        print(r)

- Different components have been subdivided according to the data that they process


#### A Query Language

- Now that we have our log, let's do some queries

- Find the set of all documents that 404

        stat404 = set(r['request'] for r in log if r['status'] == 404)

- Print all requests that transfer over a megabyte

        large = (r for r in log if r['bytes'] > 1000000)
        for r in large:
            print(r['request'], r['bytes'])

- Find the largest data transfer

        print("{} {}".format(max((r['bytes'], r['request']) for r in log)))

- Collect all unique host IP addresses

        hosts = set(r['host'] for r in log)

- Find the number of downloads of a file

        sum(1 for r in log if r['request'] == '/ply/ply-2.3.tar.gz')

- Find out who has been hitting robots.txt

        addrs = set(r['host'] for i in log if 'robots.txt' in r['request'])

        import socket
        for addr in addrs:
            try:
                print(socket.gethostbyaddr(addr)[0])
            except socket.herror:
                print(addr)

#### Some Thoughts

- The beauty of generators lies in the fact that you can plug filters at almost any stage

- Like the idea of using generator expressions as a pipeline query language

- You can write simple filters, extract data etc.

- You can pass dictionaries/objects through the pipeline, it becomes quite powerful

- Feels similar to writing SQL queries
