# Git for data

It seems that the need for version-controlled data comes times and times again,
yet there is no usable solution for that problem, so I am considering building it myself.
Below are my thoughts on how this could be achieved.


What I would need from such tool:

- understanding of the data format, so that usable diffs and merges can be produced.
- it should record who changed what, when and why.
- I expect it to support branching, merging and detailed diffs.
- it should work from command line in a manner similar to git
- as git, it should allow distributed collaboration in efficient manner
- the source and the manner of development should be open
- it should be easy to integrate with existing project, working as a library.

In other words, I want to be able to do this:

    ```
    mkdir myproject && cd myproject
    gfd init
    gfd exec 'create table books(id:int, author: varchar, title: varchar)'
    gfd exec 'insert into books(1, "Terry Pratchett", "Good omens")'
    gfd status
        created: tables/books/schema
        created: tables/books/data/1

    gfd commit -m 'create books table and add Good omens'
    gfd push origin master
    ... some time later, on another server
    gfd push origin:new_branch
    gfd checkout new_branch
    gfd pull origin new_users
    gfd log 
        commit: 0001
        author: you <your@email.com>
        date: 2 hours ago

            create books table and add Good omens

        created: /tables/bookd (id:int, author: varchar, title: varchar)
        created: /tables/books/1 (1, "Terry Pratchett", "Good Omens")

        commit 0002
        author: someone else<someone@else.com>

            fix missing author

        modified /tables/books/1: 
            -author: "Terry Pratchett"
            +author: "Terry Pratchett & Neil Gaiman"
    ```


No such thing exists as far as I know, at least in the form I would be happy with.

Note 1: if you read this, I assume you are familiar with git internals.
Note 2: I don't really know much about git and databases, this is just the beginning for me. Comments and reading materials are welcome.


# General idea:

Git stores content for files. Git for data would store patches for subsets of data (partitions).

* Perceived data model: let start with a set of rows, same as relational database table. We have set of rows, each of them conforming to some predefined schema. For example lets have datased defined as Users(id: u64, name: string, email: string). Each row must contain unique identifier.

* Versioning: the state of our dataset will be represented by a hash (commit), as in git.

* Partitioning: To make operations more efficient we need to add one more thing: partition strategy. We will come with a way of splitting whole dataset into smaller, manageable parts.
An example of a strategy would be first letter of the name, range of dates, hash of some field(s).
The strategy would be defined once and immutable (transfering immutability to any fields it relies on.

## Data definition:

Storage model: just as git, storage would be key-value store, where key represent a hash of an object.
The object itself can be:

* patchset: list of changes for given partition. For example for partitioning by first letter of a name (unicode aside for a moment), patches data would look like this


    ```
    change1: add row (id=1, name="Abelard", email="...")
    change2: set email to "a@newcorp.com" for id=2
    change3: delete id=7
    ```

Each change refers to single row: it can't be "update users set email=lowercase(email)", though a tool that converts that into set of changes could be created). In other words, change defines what data should be, not what operations should be applied.

Schema change would be included here as well. (Change 4: add field "last visited" with default value of None for id=1,2,3,4)

* tree:

    ```
    set of patchsets, hash of a parent

    partition "a": patchest 1
    partition "b": patchset 2
    parent: hash of a parent.
    ```

That would mean: data exactly as for parent, except for partition "a" and "b", where patchest 1 and 2 should be applied, respectively.

* commit: same as in git, we would record tree hash, author name, date and message

    ```
    tree: hash(tree)
    comment: "Add abelard to our users"
    date: "2019-03-05 11:34:00"
    author: "a.b@whatever.com"
    ```

Having all that, we would be able to track all changes and obtain a state of the whole data set for any given commit.


* Materialisation

We recorded all the changes, so we know what is the history of our data. That doesn't tell us automatically what the data looks like right now. To know that, we would need to start from scratch and apply all changes one by one. For any sizeable repository that would be very expensive operation, yet ultimately data state is what every user wants to see. Also for many usecase, we would want immediate access not only to current state, but also for some historical ones. I imagine achieving that by having materialisation strategies, where we define set of states and dump data into some format for that state. A common materialisation strategy would be "current" to have data in the final form for actual commit. In many cases it would be usefull to have periodic materialisations, so that we could easily see and compare data by month or year.


## Differences between text files and set of rows, and their implications.

With git, we have natural and usable way of using data: files on disk. With data it is a bit more complicated. It would be very hard to map repository content into a single file on filesystem, and to keep track of the mapping if you would edit it. I imagine git-for-data as a tool similar to sqlite file: you would be either using via a command line (having some sort of query language to make things easy), have a dedicated application to aid you or enhance the application you are using to support git-for-data repository format.


## Beyond the basics:

I tend to require trees often, so I will enhance data model to accomodate them, though that hopefuly doesn't change the design much.

Schema migrations look like an expensive thing: it would be required either to track all changes for each partition and record, otherwise data validation would become much harder to achieve (we need to know if a field named "name" exists if we want to change it, and what schema is currently/was at some point to apply any change safely). This requires more thought. I absolutely want to have a rigid schema though (with a JSON-like field maybe as an rarely-used escape hatch).

Data validation (especially relational part of it) may require materialisation.

While the phase "sql" was not used so far and doesn't need to be, its natural to think about having some sql-like layer on top of the whole thing (even if only as a client convenience, not related to server), so I will list some of my nitpick with sql:

* it leans towards users a bit to much. If I'd design a query language (actually, this is relevant for storage format also), I would make everything not-null by default, treat nullable fields as distinct type and require any operations on them to define treatments of nulls explicitely.
* numbers would be defined with type information (u32/i32/u64/i64/decimal/bigint)
* floating point numbers would have "sane type" that does not accept NaN/Infinity and it would by default.

I value correctness over performance, and therefore would like to have support for transactions (with a concepts that are equivalents of multiple tables in a database, cross-table validation and relations would be nice to have) 



## Existing projects

* Filesystem-level snapshots. Kind of work, but obviously have no understanding of the data whatsoever,
so you know that "some data changed between snapshot a and b", but beyond file size you can't say anything else about it. Diffs and merges are out of the question.

* Storing data dumps as text in git. Sort of meets requirements, but doesn't scale very well.

* Database audits & replication: No, it doesn't work. It only works one way - to pull changes from single source. There is no concept of braches. You can't have local modification.
