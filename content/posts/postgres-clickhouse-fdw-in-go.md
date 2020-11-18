---
title: "Writing a Postgres Foreign Data Wrapper for Clickhouse in Go"
date: 2020-11-18T21:59:24+01:00
draft: false
toc: false
featuredImg: ""
tags: 
  - go
  - postgres
  - clickhouse
  - fdw
  - cgo
  - c
---


[Postgres](https://www.postgresql.org/)(hereinafter mentioned as PG) is a pretty cool database with lots of nice features, one of them little known ones is the ability of having Foreign data wrappers [(hereinafter mentioned as FDWs)](https://wiki.postgresql.org/wiki/Foreign_data_wrappers).

[Clickhouse](https://clickhouse.tech/)(hereinafter mentioned as CH) is another amazing database with an altogether different set of features targeted for OLAP use cases.

## What are Foreign Data Wrappers (FDWs) then?

Well unlike so many names in tech, we can actually infer some idea from the name itself in this case.
So FDWs in essence, allows to access _foreign_ _data_ sources inside Postgres(PG) via a set of [wrapper APIs](https://www.postgresql.org/docs/current/fdwhandler.html).

That is, you can access data sitting in a Mysql/SQlite/Clickhouse(_any other data source_) table inside PG as you would do for a normal PG table. Isn't that amazing!

There are already numerous such FDWs. A list is available [here](https://wiki.postgresql.org/wiki/Foreign_data_wrappers).

One caveat is that the extent of features you can expect from a FDW is dependent on the particular implementation.
We can expect normal read support but other niceties like push-down filters, aggregations or joins, or write support can be missing.

## Accessing Clickhouse(CH) via Postgres(PG)

Given the existence of so many possibilities of accessing other datastores, wouldn't it be fun if we could access Clickhouse from inside Postgres.

Why would you want to do it! you may ask.

Well one reason could be of course, for _fun_.

But more realistically, one of the ambitious use cases at [MessageBird](messagebird.com)(my employer) was the ability to connect Clickhouse to [Looker](looker.com) as no direct integration existed at that time.
It was a bit of a moonshot but we decided to give it a try to see if it would work :)

MessageBird has generously made the full source for our experiment open source! The repository is available [here](https://github.com/messagebird/clickhouse-postgres-fdw).
So you can reference the ideas mentioned in the blog post directly in the code as well :) 

### Now on to writing one FDW!

There are already [documentations](https://www.postgresql.org/docs/current/fdwhandler.html) on how we should approach this and some simple examples are also available on Github. Most of the full fledged FDWs have their code in open so we can consult them as well.
Note that most of them are written in C becauses the FDW API of PG is in C, which makes sense.
I should highlight one particular [FDW](https://github.com/pgspider/sqlite_fdw) that is made for SQLite and has a solid feature set, which helped me a lot while writing the one for Clickhouse.

But what if we want to be adventurous and write one in Go? Well, it should be possible given the existence of [CGo](https://golang.org/cmd/cgo/).

We can expect that it will not be at all trivial. ;)
There are already attempts on making Postgres Extensions in [Go](https://github.com/liztio/k8s-fdw), which gives a very valuable insight.

#### Setting up the build process and interaction between Go and Postgres C API

First we should familiarize ourselves with the [PG Extension Build Infrastructure/PGXS](https://www.postgresql.org/docs/13/extend-pgxs.html) and how to write [C code](https://www.postgresql.org/docs/current/xfunc-c.html#DFUNC) for PG.
These are crucial as we would want to integrate the C code with Go, and knowing how the build process works should help us in understanding where our code will fit.

For writing a FDW we have to provide an entry point in form of a `struct` containing function pointers to the implemented callback functions.
Since we want to write those callback in Go, we can consult documentation for [accessing Go functions in C](https://golang.org/cmd/cgo/#hdr-C_references_to_Go), which says there are specific annotations that should allow us to export go functions outside to the C code.
All the important work is done in these callbacks only.  
Now it should be possible to add functions that PG FDW API expects via Go.

But, how will the C code find the callback functions written in Go land ? [Go build modes](https://golang.org/cmd/go/#hdr-Build_modes) is the answer.
Directly referencing from the documentation, the `-buildmode=c-archive` allows us to:

	Build the listed main package, plus all packages it imports,
	into a C archive file. The only callable symbols will be those
	functions exported using a cgo //export comment. Requires
	exactly one main package to be listed.

Perfect! Now, the exported Go functions are available in the [archive file](https://linux.die.net/man/1/ar).
The only remaining thing is to link the archive with C code during build.
Thankfully, [PGXS](https://www.postgresql.org/docs/13/extend-pgxs.html) provides a `Make` variable `SHLIB_LINK` that can be used to set the shared library used. So we'll use that flag to provide the archive build from Go source files.

You can see it in action [here](https://github.com/messagebird/clickhouse-postgres-fdw/blob/main/Makefile).

<!-- * how linking go code to c via statically linked libraries works ? .a files linking with Go. -->

<!-- And we want a library of sorts which should be callable inside C code. For this we can use go build mode (c-archive), this will create a statically linked archive(see difference btwn shared objects and statically linked libraries and how to build them) -->

#### Understanding inner working of the FDW API and their relation with different Query Stages

<!-- * Figuring out which stages do what, what do you want and where to look for them -->

To actually write a working FDW, we need to familiarize with the different stages a query goes through in PG and how the API functions play them out.
Postgres has an excellent documentation and moreover since all the [source code](https://github.com/messagebird/clickhouse-postgres-fdw/) is open, we can just navigate through the code as well!

It would take more space than a blog post to explain the full internals of Query planner in Postgres and I probably can't describe it well enough.
So, I suggest to go through [the official documentation](https://www.postgresql.org/docs/current/fdw-callbacks.html) which is quite excellent and there are many other excellent references on the web.

I'll try to briefly explain the flow of the API functions for the context of this post.
A very basic plan looks like this:

```
+-------------------------+
|                         |
|                         |
|    GetForeignRelSize    |
|                         |
|                         |
+------------+------------+
             |
             |
             |
+------------v------------+
|                         |
|                         |
|     GetForeignPaths     |
|                         |
|                         |
+------------+------------+
             |
             |
             |
+------------v------------+
|                         |
|                         |
|     GetForeignPlan      |
|                         |
|                         |
+------------+------------+
             |
             |
             |
+------------v------------+
|                         |
|                         |
|     BeginForeignScan    |
|                         |
|                         |
+------------+------------+
             |
             |
             |
+------------v------------+
|                         |
|                         |
|    IterateForeignScan   |
|                         |
|                         |
+-------------------------+
             |
             |
             |
+------------v------------+
|                         |
|                         |
|     EndForeignScan      |
|                         |
|                         |
+-------------------------+

```
There are other functions in the FDW API that I've omitted here (like `ReScanForeignScan`, `AnalyzeForeignTable`, `GetForeignUpperPaths`) but a basic FDW can be done with these.
Also note that this path is only concerned with the read queries. To enable writing on the foreign database, there are separate functions that need to be implemented.
You can see how the read path is implemented for our clickhouse FDW [here](https://github.com/messagebird/clickhouse-postgres-fdw/blob/main/ch_fdw.c).

Important ones to take note of to properly implement the read path of a query are:

* GetForeignRelSize: It should be used to determine the estimated number of rows to be scanned on the foreign server. However, it is also used to extract the restriction clauses present in the query presented by PG and to pass them to the foreign server if it can support them.
  See this [example](https://github.com/messagebird/clickhouse-postgres-fdw/blob/a938204f1a5645cd71dd473b5d0e5f56d2a6f831/ch_fdw.go#L180).

* GetForeignPlan: It should return the planner node(a data structure that contains the query plan). However, it is also used to extract the target columns that can be fetched from remote/foreign servers and pass that info along with restriction clauses, table names to the next stage.
  See this [example](https://github.com/messagebird/clickhouse-postgres-fdw/blob/a938204f1a5645cd71dd473b5d0e5f56d2a6f831/ch_fdw.go#L275).

* BeginForeignScan: It should perform the initalization that is needed to perform the scan on the foreign server, for example: initialize the foreign DB connection, formalize the query running on foreign server and init the state with row iterator to be used in the next stage.
  See this [example](https://github.com/messagebird/clickhouse-postgres-fdw/blob/a938204f1a5645cd71dd473b5d0e5f56d2a6f831/ch_fdw.go#L508).

* IterateForeignScan: It should return a row from the foreign server converted to the PG specific structure. This function should convert the foreign server specific data types to PG column data types.
  See this [example](https://github.com/messagebird/clickhouse-postgres-fdw/blob/a938204f1a5645cd71dd473b5d0e5f56d2a6f831/ch_fdw.go#L620).

* EndForeignScan: It should clean the state being stored for the query, like row iterators, db connections should be closed.
  See this [example](https://github.com/messagebird/clickhouse-postgres-fdw/blob/a938204f1a5645cd71dd473b5d0e5f56d2a6f831/ch_fdw.go#L760).
  
This is a very dense overview of the functionality of a _basic_ FDW. It usually helps to look around the other FDWs that are open source to look for ideas of a sample implementation. But it can differ since the foreign server can be of various types.
Usually, if we take databases that support some dialect of SQL then the hardest things are usually figuring out if the restriction clauses are remote safe, which can involve parsing the full expression clauses and then converting them to remote variants.
Converting the foreign server datatypes to PG types is comparatively easy but is very toiling.

You can look into how [clickhouse FDW](https://github.com/messagebird/clickhouse-postgres-fdw/blob/a938204f1a5645cd71dd473b5d0e5f56d2a6f831/ch_fdw.go) does this to get an idea, but beware that it could be bug prone, since it hasn't been tested thoroughly.

I'll also suggest getting an idea of commonly used PG datatypes and conventions like `OID`, `Tuple`, `RelOptInfo` or just going over [`relation.h`](https://doxygen.postgresql.org/relation_8h.html) reference from PG source code.

### Few Tips and Tricks

These are some ideas that I've seen are fairly used while developing a FDW. Some can help in easy interop between Go and C, whether it is a good idea or not, is up for debate ;)

#### Interfacing C macros within Go
There are a lot of internal macros in PG source which makes it easier to access system cache, lists, heap tuples etc. which aren't directly callable from Go's userland.  
This is because CGo doesn't quite allow directly calling C `#define` macros.  
You can try to simulate the same behaviour using underlying constructs but that can get hairy and cumbersome. Instead one _easy_ idea is to define simple C wrapper functions like
```
void *wrapper_access_list(void *list, int index){
	return access_list(list, index);
}
```
This can now be used directly on Go side. But make sure you cast the results to proper types.

#### Moving out C code

There can be a point where writing C code directly in Go source files is not feasible anymore, because increasing the number of commented lines can get incomprehensive after a point.  

Moving out C code into separate files and then accessing them in Go can be done as well.  
You need to link the Go code with the C symbol definitions that are outside the Go code during build times and it should work.
It can be done by for example, separating the C code into header and source file and including the header file in the Go source code. Now CGo will automatically take care of building the object files.
See it in action [here](https://github.com/messagebird/clickhouse-postgres-fdw/blob/a938204f1a5645cd71dd473b5d0e5f56d2a6f831/ch_helpers.c). For better understanding of different object files, see [here](https://stackoverflow.com/questions/9688200/difference-between-shared-objects-so-static-libraries-a-and-dlls-so).

#### Maintaining execution state of C API inside Go land

Go doesn't allow passing pointers to Go objects (like maps, slices) to C (which makes sense, with Go being a garbage collected language). See [this reference](https://golang.org/cmd/cgo/#hdr-Passing_pointers) for more details.  

But we want to maintain states in Go for objects that are being accessed by C code.  

One example is database connections and cursor(row iterator). As we would want to access connection as a Go variable or in a PG execution stage we would want to use the same cursor which is being initialized in upper stages (i.e planning).  
One trick to make this happen is to keep a map of integers => Go objects and pass that integer around as we move downstream in the query stages.
This integer is always incremented in the FDW's lifetime (in each stage) and is never reused again in calls to FDW.
For example see it [here](https://github.com/messagebird/clickhouse-postgres-fdw/blob/a938204f1a5645cd71dd473b5d0e5f56d2a6f831/ch_fdw.go#L73).

#### Passing FDW state around different stages

PG FDW API also allows to embed internally private information as an opaque `void *` that will be passed around in the query stages.  
This is quite neat IMHO and saves a lot of time for developers to allow bookkeeping :)  
You can pass state around stages by simply providing a `void * fdw_state` where we can put anything (quite _literally_).  
Clickhouse FDW uses this to pass around query state information like extracted remote safe restriction clauses, table names, column names etc. See this [reference](https://github.com/messagebird/clickhouse-postgres-fdw/blob/a938204f1a5645cd71dd473b5d0e5f56d2a6f831/ch_fdw.go#L196).

#### Type conversions between Go and C
<!-- * Type conversions btwn Go and C , thinking how would you translate a native DB type in Go to C (date/time, sql package etc) -->

Since we are fetching the results from the Go driver of a DB and then pushing it down to the C API of PG, we need a conversion step between.  
Normally, it should be easy for fixed size integers and strings but PG has a large catalogue of datatypes like datetime, time zones, arrays and such. Further the foreign database can also have its own list of datatypes that might not be properly represented as a PG type.  
In that case, we have to do the best we can and might have to leave those fancy datatypes out.  
For user defined datatypes like variable length strings or arrays, PG has a nice system of input and output [function for conversions](https://www.postgresql.org/docs/13/xtypes.html).  
We can rely on these functions for converting the value from Go to C.
See this in action [here](https://github.com/messagebird/clickhouse-postgres-fdw/blob/a938204f1a5645cd71dd473b5d0e5f56d2a6f831/ch_fdw.go#L784).

#### Support push down Filters and Aggregates

<!-- * Expression deparsing is one of the major challenges that would need to be implemented.
* Evaluation of exprs that can be implemented on remote or not (is major required for push down filters)
 -->
This is one the nicest features to provide since it will leverage the underlying foreign DB's capability and heavily reduce the amount of bytes travelled over the wire.  
Also, this makes the query being done on PG much closer to how it will be done on the foregin DB because otherwise the query will use PG's query planner.

To support push downed filters, we first have to deparse the expression from query clauses and then evaluate if they can be implemented in terms of foreign databases.  
I mostly leveraged the existing code that [sqlite_fdw](https://github.com/pgspider/sqlite_fdw) has written for the same but tweaked it further to convert the expressions in terms of Clickhouse expressions.
See it in example [here](https://github.com/messagebird/clickhouse-postgres-fdw/blob/3904e5573ac9585d87021952f4c0ffcddee4c7ff/deparse.go).

The functionality can be splitted into two parts, first can be the expression [deparsing](https://github.com/messagebird/clickhouse-postgres-fdw/blob/3904e5573ac9585d87021952f4c0ffcddee4c7ff/deparse.go#L197) and second can be the [evaluation](https://github.com/messagebird/clickhouse-postgres-fdw/blob/3904e5573ac9585d87021952f4c0ffcddee4c7ff/ch_helpers.c#L348).

For pushing down aggregations like `GROUP BY`, the deparse and evaluation are mandatory step but we also have to implement FDW functions like [`GetForeignUpperPaths`](https://www.postgresql.org/docs/current/fdw-callbacks.html#FDW-CALLBACKS-UPPER-PLANNING).

## When this could be a bad idea?

In order to properly understand the FDW's working and further extension, it is required to understand Postgres internals and it's FDW API.  
Further a brief understanding of database design theory is also needed.  
If the teams doesn't have such expertise beforehand, this can be a major bottleneck in providing a production grade interface between Foreign DB(like Clickhouse) and PG.  
It shouldn't be the case that developers fear the code because it is too magical and they don't understand the inner workings.

Next, if one is assumed to have an understanding, there are the challenges of keeping the C to Go interface calls manageable.  
See [CGo performance penalties](https://www.cockroachlabs.com/blog/the-cost-and-complexity-of-cgo/) for example. So it is desirable to keep the codebase sane by keeping the amount of C code minimum.  
We somehow need to provide a balance between the two.

Further challenges can be:

 * We can try to keep much of the code in Go (involving C interface functions) but we also need to be careful while passing memory chunks from Go to C.  
   Any `Go.CString(...)`s aren't claimed back via Go runtime, although in case of C structs PG runtime promises to claim back the memory allocated by palloc as soon as the transaction ends. There are some constructs where we might need to touch PG's HeapTuple memory allocators and manually make sure we free them after use.
 * New collaborators will need to learn tricks around the converting C structs/datatypes to Go ones(and vice versa) and marshalling/unmarshalling results/arguments of PG C functions.

A very extreme way is, To try to keep most of FDW stages in C with only "deparsing and type conversions from Go types to C types" and "Clickhouse query formation" in Go, but if the teams lack expertise in C, this may not be a favourable option.

The end result of this experimental project is available [here](https://github.com/messagebird/clickhouse-postgres-fdw/). I hope that other teams will find the code useful.

If these kinds of challenges excites you, [MessageBird](https://messagebird.com) is [hiring](https://messagebird.com/en/careers/) for lots of attractive roles!

<!-- 
* First stage: receiving options information about remote server and other options from POstgres
getting the relative size of the table:
* Second stage: Building a Foreign scan path (which is usually simple, unless you implement UPPER relation paths or JOIN paths)
* Third stage: Making a foreign plan that would contain the target columns, table names, (conditions, group bys etc)
* fourth stage: Extracting the plan and make connection to Foreign server
* fifth stage: a row iterator from the foreign server that returns the data, this also needs type conversions from native Go types to PG types
* sixth stage: clean up of resources
* advance sstage: Implementing push down aggregations on the foreign server -->