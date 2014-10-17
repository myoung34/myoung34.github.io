---
title: Seriously MySQL?
author: myoung
layout: post
comments: true
permalink: /post/seriously-mysql/
image:
  - 
seo_follow:
  - 'false'
seo_noindex:
  - 'false'
categories:
  - MySQL
tags:
  - bug
  - datetime
  - mysql
---
I recently discovered one of the most confusing things I&#8217;ve seen in MySQL.<!--more-->. Lets start with this table: 

```
| Table | Create Table      
| blah  | CREATE TABLE 'blah' (
  'id' int(11) NOT NULL,
    'date' datetime NOT NULL
    ) ENGINE=MyISAM DEFAULT CHARSET=latin1 |
```

What this means is you have a required date. Let&#8217;s say this is to ENFORCE that date is there and valid (without a default value obviously).Now, what you can expect, is that you cannot force the column to take a default value&#8230;it must be specified, so this is what you&#8217;d expect to see:

```
mysql> insert into blah set id=1,date=NULL;
ERROR 1048 (23000): Column 'date' cannot be null
```

Cool&#8230;that&#8217;s right&#8230;so why does this work? 

```
mysql> update blah set date=NULL where date is not null;
Query OK, 1 row affected, 1 warning (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 1
 
 mysql> select * from blah;
 +------+---------------------+
 | id   | date                |
 +------+---------------------+
 |    1 | 0000-00-00 00:00:00 |
 +------+---------------------+
 1 row in set (0.00 sec)
```

&#8230;&#8230;&#8230;Huh? It took on a default value, even specified as null?!If that doesn&#8217;t irritate you, what about this? 

```
mysql> insert into blah set id=2;
Query OK, 1 row affected, 1 warning (0.00 sec)
 
 mysql> select * from blah;
 +------+---------------------+
 | id   | date                |
 +------+---------------------+
 |    1 | 0000-00-00 00:00:00 |
 |    2 | 0000-00-00 00:00:00 |
 +------+---------------------+
 2 rows in set (0.00 sec)
```


&#8230;.So, I&#8217;ve specified that it cannot be NULL, but because datetime doesn&#8217;t actually have a NULL datatype, it accepts the NULL datetime of &#8217;0000-00-00 00:00:00&#8242;. This is completely illogical. What&#8217;s worse, I specified NOT NULL, yet wrote an update to set date=NULL, and it passed. According to the MySQL 5 page, it&#8217;s because: 

```
Illegal DATETIME, DATE, or TIMESTAMP values are converted to the “zero” value of the  appropriate type ('0000-00-00 00:00:00' or '0000-00-00').
```

That&#8217;s fine and dandy, but the MySQL is ignoring the fact that I set date=NULL where the schema shouldn&#8217;t allow it.
