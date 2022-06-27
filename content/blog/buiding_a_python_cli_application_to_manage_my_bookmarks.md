+++
title = "Building a Python CLI application to manage my Bookmarks"
date = 2022-06-20

+++

Recently I have started to highlight text in my Kobo ereader as I go through a book.
Once I've created a few bookmarks the next things I would like to do would be to do
would be to somehow import them to my computer, ideally in a format such as markdown
(since I like to use [Obsidian](https://obsidian.md/) to organise my notes), so I
built [a small program](https://github.com/videbar/kobo-highlights) do it.

<!-- more -->

# My ereader bookmarks

To be more specific about what I wanted, when reading a book, my erader has the option
of selecting part of the text in the book and highlighting the selection. This
highlight is then saved and in my ereader I can check all the text that I have
highlighted for a given book. It's also possible to add annotations to the highlighted
text. This annotations are free text that are linked to the highlight and that can also
be access from the erader menu.

The problem is that, I couldn't find a way of taking these bookmarks (the highlighted
text and the potential annotations) out of the ereader and into my computer. Ideally,
I would have some program that I could run on my computer when my ereader is connected.
This program would be able to understand the bookmarks in the ereader, get both the
highlights and the annotations, and understand from which book they are coming from.
Then it would group them in a convenient format.

# Designing the application

Since I could not find anything close to what I wanted I decided to build my own
application. It could be a CLI application written in Python using [Typer library](
https://typer.tiangolo.com/). I had used Typer before and I knew it works very well for
this kind of small CLI utilities. I also could used [Rich](
https://github.com/Textualize/rich) to have somewhat nice interface, I have been
wanting to try it out for some time and this seems like a project to give it a go.

The other thing I needed was a way of retrieving the necessary information from my
ereader. This is, I needed a way for my program to obtain a list of the bookmarks
in my ereader, the highlighted text, and the potential annotations. It also needed
to know to which book each bookmark corresponded to (the book title and author). I
started looking  and I found a few website like [this one](
https://www.epubor.com/export-kobo-highlights-and-notes.HTML) that explain that you can
easily export your bookmarks in as a `.txt` file by enabling it  in the configuration
of the ereader. It would be easy to take this `.txt` file, parse it to extract the
information I wanted and format it into something like markdown.

But as I looked into this in more detailed it was clear that it was not a good
approach. For once in order to create the `.txt` files I would need to select each
book in my ereader and tell it to export the highlights before my computer could 
retrieve them. This is not a major problem, but a minor annoyance. The main issue was
that the file containing the bookmarks didn't appear to have a proper structure.
All bookmarks from the same book would be added to a single file without any separator
between them beyond a few empty lines (the number of empty lines
wasn't consistent either). What's more, if the bookmark had annotations, the
annotation text would mix with the highlighted text. Because all of this, parsing the
contents of the `.txt` file to retrieve the bookmarks in a somewhat general way would
be nearly impossible.

When I realised this I started digging around at the files in my ereader and I found a
`KoboReader.sqlite` file. This file corresponds to a [SQLite database](
https://www.sqlite.org/fileformat2.html) and, after inspecting the database, I found
out plenty of information. Notably, there were two tables that could be useful for
this project. The first one was a Bookmark table that looks like this (here only the
relevant columns are shown):

|       VolumeID      |         Text         |     Annotation     |                UUID               |
|:-------------------:|:--------------------:|:------------------:|:---------------------------------:|
| \<ebook file path\> | \<highlighted text\> | \<annotated text\> | \<universally unique identifier\> |
|         ...         |          ...         |         ...        |                ...                |
|         ...         |          ...         |         ...        |                ...                |

I could easily query this table from my application to obtain the necessary information
using [a SQLite libary](https://docs.python.org/3/library/sqlite3.html). And because
this is a proper database instead of a simple `.txt` file, no parsing is necessary. I
noticed in the Bookmark table some rows that didn't have any data in the `Text`
nor in the `Annotation` entries. I deduced that this correspond to bookmarking a
page, which is something I can also do on my ereader. I was not interested in this
kind of bookmarks, but I should be able to easily ignore them when querying the
database.

The other interesting table that I found in `KoboReader.sqlite` is a Content table
that contains information about all the books that are currently stored in the ereader.
It looks like this (again, some irrelevant columns have are not being shown):

|      ContentID      |      Title     |   Attribution   |    ContentType    |
|:-------------------:|:--------------:|:---------------:|:-----------------:|
| \<ebook file path\> | \<book title\> | \<book author\> | \<type of entry\> |
|         ...         |       ...      |       ...       |        ...        |
|         ...         |       ...      |       ...       |        ...        |

The column `ContentID` in the `content` table represents the same information as the
`VolumeID` in the `Bookmark` table. The column `ContentType` is an interesting one.
There are two types of entries in the content table, general information about each
book and information regarding specific chapters or section of the book, the former
have a value of `6` for the `ContentType` and the later have a `9` (I don't know the
reason for this numbering choice).

For the entries that correspond to general information about books, the columns `Title`
and `Attribution` contain the book title and author respectively.

So the plan was clear, use Typer to build a CLI application that could query the SQLite
database, retrieve the bookmarks and book information, and format them nicely in a
makdown document. I would also be able to display information about the bookmarks and
for that I would use Rich. The application will also need a way of keeping track of
the bookmarks that it has already imported, so that it is able to only import new
bookmarks when it's called multiple times.

# The technical details

So that's exactly what I built. I called the final application Kobo Highlights and it
can be called from the terminal using the command `kh`. It supports the following
subcommands:

* `kh config`: It is used to manage the configuration of the program. Basically Kobo
Highlights needs to know two things, where is my erader mounted and where to I want
the final markdown files to be created. This two things are stored in a config file
and the `config` command can be used to see the current configuration and to create
a new one. Kobo Highlights can't work with a proper configuration, so if it called
and it can't find one it will prompt the user to create one interactively.

* `kh ls`: It lists the available bookmarks. By default it will only list new
bookmarks, this is, bookmarks that are on the ereader but have not been imported yet.
The `-all` flag can be use to list all the bookmarks in the ereader instead,
independently of whether or not they have been imported.
The `ls` command will list for each bookmark the highlighted text, the annotation
(in case there's one), the book title and author and the ID. The ID of the bookmark
is the `UUID` assigned to the bookmark by the ereader which turned out to be a very
useful field.
The book title is quried from the content table, and the book author is extracted from
the folder structure in the ereader. Because I use [Calibre](
https://calibre-ebook.com/) to manage my ebooks, the folder structure on my ereader
follows the schema `/<author name(s)>/<ebook file>`, but this may not work properly
with other folder structures. It is unfortunate but I didn't find any other way of
obtaining the author information. In they future I may add an option for the
application to ignore the author and only look at the book title.

* `kh import`: Finally, the main point of the tool, this command will import a set
of bookmarks and store them as a markdown document. The markdown documents will
follow a simple structure in which the highlighted text is included as a block
quote and the potential annotations are added bellow. The `import` command can be
called with multiple options, it supports `all` to import all bookmarks in the ereader
and `new` to only import the ones that weren't imported before, but it also supports
being called with a list of IDs, a book title, or a book author.

One thing I had to consider was how to keep track of the bookmarks that have already
been imported. Originally I queried the markdown document and looked at the text 
inside each block quote. This works ok but it has some problems.

The main issue is that it's not easy to parse markdown text. This is because markdown
is a format for designed for human readability, unlike JSON, which is designed mainly
for serialization. In order to parse the markdown files I first convert them to html
and then use [Beautiful Soup](https://www.crummy.com/software/BeautifulSoup/bs4/doc/)
to parse the html text. This works fine on most cases but it's only a matter of time
before some estrange edge case breaks it.

The other issue is that sometimes I may want to modify the highlighted text. An example
of this is when I highlight part of a sentence and I want to add some punctuation or
change the capitalization. The current markdown-to-html parser allows to add emphasis
to the highlighted text, but as soon as you modified a single character it will not
recognize the bookmark and it will re-import it the next time it's called.

In order to solve these issues I got rid of my markdown parser and instead
store the IDs (I told you there are a useful field) of the bookmarks that have been
imported in a hidden JSON file in the same directory as all the markdown files. Then
every time the application needs to know what highlighted have been imported, all it
needs to do is load this JSON file.

Another thing to point out is that when querying the ereader database, the application
doesn't access the SQLite file directly, instead it creates a local copy of the file
in the computer and queries the data from the copy. Once it's done it deletes the
local file. The reason for doing this is that the database doesn't only contain the
Bookmarks but also a lot of data that seems important for the ereader's functionality.
Since I'm not that familiar with SQLite, I'd rather not interact with the ereader
database directly, but use a copy instead, just in case.

# Where to find the program

In case you want to have a look or try Kobo highlights, [it is available on pypi](
https://pypi.org/project/kobo-highlights/) and the entire source code is hosted
[in github](https://github.com/videbar/kobo-highlights).
