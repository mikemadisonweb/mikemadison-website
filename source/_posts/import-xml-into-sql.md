---
title: Import XML file into database
date: 2017-01-08
featured_image: import-xml-into-sql.jpeg
tags:
- PostgreSQL
- XML import
- Python
- MySQL
- SQL
---
I will not talk about whether it's a good idea or not to store dump in XML, let's suppose you have this huge XML file and you need to load it in your database. In fact the bigger the file is the more problem you have because you need to think about import performance and memory consumption. Also, the schema of the data inside your dump may be messy or totally irrational. All this determines the required flexibility of your import method.

For the sake of this article, I will use [stackoverflow comments dump](https://archive.org/download/stackexchange) which is nearly 10Gb after unpacking.
```sql
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
To demonstrate how easy this stuff can be, let's look at how MySQL handles that kind of task:
```sql
LOAD XML LOCAL INFILE '/var/lib/mysql/dump/Comments.xml' INTO TABLE comments ROWS IDENTIFIED BY '<row>';
```
The Mysql solution is short and simple, but it works only on version >=5.5.  Column names associated with either node attributes or `field` nodes with required `name` attribute. In the last case value for the particular column will be taken from node text. If you are interested you can find more about it in [official Mysql docs](https://dev.mysql.com/doc/refman/5.5/en/load-xml.html).

Import took just about 2 hours for our test XML dump.

Simplicity in MySQL came with a price, as it's kind of limited in allowed XML formatting. That's can be a huge problem for example if you import relational data.

### XML processing functions in PostgreSQL
I decided to put MySQL example here for a reason. Not that MySQL lacks XML processing, it's PostgreSQL that don't have such a simple solution. Postgres have advanced functionality and that usually leads to overcomplicated solutions. When I was searching for a way to do XML import I stumbled upon [this stackoverflow answer](http://stackoverflow.com/a/7628453) which give me a hint on how to solve my problem but introduced a lot of custom user functions, which wasn't necessary. I ended up with this:
```sql
INSERT INTO comments
  SELECT (xpath('//row/@Id', x))[1]::text::int AS Id,
         (xpath('//row/@PostId', x))[1]::text::int AS PostId,
         (xpath('//row/@Score', x))[1]::text::int AS Score,
         (xpath('//row/@Text', x))[1]::text AS Text,
         (xpath('//row/@CreationDate', x))[1]::text::date AS CreationDate,
         (xpath('//row/@UserId', x))[1]::text::int AS UserId
  FROM unnest(xpath('//row', pg_read_file('dump/Comments.xml')::xml)) x;
```
XPath provide a lot of flexibility, this time we are not limited with specific XML format. This method have some non-obvious quirks though:
- Your dump should be in the data directory of Postgres. That's the pg_read_file function requirement. There are some alternatives to it in Postgres, more on that matter in stackoverflow thread mentioned above.
- As you can see XML field type can't be directly converted to integer. You need a intermediate conversion to text.
- Your dump shoudn't have BOM. You should [remove it](http://www.linuxask.com/questions/how-to-remove-bom-from-utf-8) before running import otherwise you'll receive an error.

And the most important one: it fails on large XML files: `ERROR:  requested length too large`. So we either need to split our file on smaller pieces or use different approach.

### Import using Python
The key in succeeding controlling the whole process is to write your own external script. Obviously, you can't beat above solutions in execution time, but you definitely can reduce memory consumption. 

I used Python 3, lxml module to process dump and psycopg2 for the database connection. With my naive approach at first I got really slow performance, some major improvements were needed. To save you time I will post my final solution and then point out the most important parts:
```python
from lxml import etree
import psycopg2
from psycopg2.extras import execute_values
import datetime
import os
import psutil

def sizeof_fmt(num, suffix='B'):
    for unit in ['','Ki','Mi','Gi','Ti','Pi','Ei','Zi']:
        if abs(num) < 1024.0:
            return "%3.1f%s%s" % (num, unit, suffix)
        num /= 1024.0
    return "%.1f%s%s" % (num, 'Yi', suffix)

def import_xml(filename, connect, insert_command, batch = [], batch_size = 1000):
    count = 0
    cursor = connect.cursor()
    process = psutil.Process(os.getpid())
    for event, element in etree.iterparse(filename, events=('end',), tag='row'):
        count += 1
        row = [element.get(n) for n in ('PostId', 'Score', 'Text', 'CreationDate', 'UserId')]
        batch.append(row)
        # Free memory
        element.clear()
        if element.getprevious() is not None:
            del(element.getparent()[0])
        # Save batch to DB
        if count % batch_size == 0:
            execute_values(cursor, insert_command, batch)
            print("\033[?25lImported rows: {} | Memory usage: {}\r".format(count, sizeof_fmt(process.memory_info().rss)), sep='', end='', flush=True)
            batch = []
    # Save the rest
    if len(batch):
        execute_values(cursor, insert_command, batch)

def import_dump(db_name = 'fts', db_host = 'postgres-db', db_user = 'postgres', db_pass = '', db_table = 'comments'):
    filename = '/dump/Comments.xml'
    start_date = datetime.datetime.now()
    print ("Import data from {}".format(filename))
    connect = psycopg2.connect(database=db_name, user=db_user, host=db_host, password=db_pass)
    connect.autocommit = True
    cursor = connect.cursor()

    insert_command = 'INSERT INTO {} (PostId, Score, Text, CreationDate, UserId) VALUES %s'.format(db_table)
    import_xml(filename, connect, insert_command)
    connect.close()

    end_date = datetime.datetime.now()
    seconds = (end_date - start_date).total_seconds()
    print ("\nExecuted in {}s".format(seconds))

if __name__ == '__main__':
    import_dump()
```
- `etree.iterparse` returns iterator providing (event, element) pairs. In contrast with `etree.parse` iterator provide a way to save memory, you just need to remove previously parsed nodes which you don't need anymore.
- Batch insert is much faster than separate insert for each record. You can tweak `batch_size` property in order to locate a sweet spot in performance.
- psutil module is not required, it's just to visualize script memory consumption.

All in all, it took 2 hours 18 minutes to import 10Gb of stackoverflow comments and used 25Mb of RAM only. Great results!