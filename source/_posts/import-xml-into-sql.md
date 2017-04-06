---
title: Import XML file into PostgreSQL
date: 2017-01-08
---
I will not talk about whether it's a good idea or not to store dump in XML, let's suppose you have this huge XML file and you need to load it in your database. In fact the bigger the file is the more problem you have, because you need to think about import performance and memory consumption. Also the schema of the data inside your dump may be messy or totally irrational. All this determines the required flexibility of your import method.

For the sake of this article i will use [stackoverflow comments dump](https://archive.org/download/stackexchange) which is nearly 10Gb after unpack.
``` sql
CREATE TABLE IF NOT EXISTS comments (
  "Id" serial PRIMARY KEY,
  "PostId" VARCHAR(255) NOT NULL,
  "Score" VARCHAR(255) NOT NULL,
  "Text" TEXT NULL,
  "CreationDate" DATE NOT NULL,
  "UserId" VARCHAR(255) NULL
);
```
That's would a table for our comments dataset. Column names taken from xml node attributes.

### Native import in MySQL
To demonstrate how easy this stuff can be, let's look at how MySQL handles that kind of task.
``` sql
LOAD XML LOCAL INFILE '/var/lib/mysql/dump/Comments.xml' INTO TABLE comments ROWS IDENTIFIED BY '<row>';
```
Mysql solution works on version >=5.5. If you are interested you can find more about it in [official Mysql docs](https://dev.mysql.com/doc/refman/5.5/en/load-xml.html).
And that's a great solution beca
--- try to do it in Postgres
### Import using Python
As the mentioned above techniques works out of the box, they are not that flexible. If our xml file was not that properly formed we can run into troubles with MySQL way and Postgres code can become way too complex. So why can't we use some external script to do all heavy lifting in a simple and clean way. I decided to write Python script to import data from XML to Postgres table and that's what i'm end up with.
http://stackoverflow.com/questions/7491479/xml-data-to-postgresql-database

1) Problem 1: http://www.linuxask.com/questions/how-to-remove-bom-from-utf-8
2) Из xml в int сразу нельзя: ::text::int