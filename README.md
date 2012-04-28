Using Pig, Hadoop, Sqoop and Avro to Mine the Enron Emails
==========================================================

Code for creating and querying an Avro encoded repository of the UC Berkeley Enron email archive.

The Berkeley Enron Emails
-------------------------

### Introduction

Email is a rich source of information for analysis by many means. During the investigation of the Enron scandal of 2001, 517,431 messages from 114 inboxes of key Enron executives were collected. These emails were published and have become a common dataset for academics to analyze document collections and social networks. Andrew Fiore and Jeff Heer at UC Berkeley have cleaned this email set and provided it as a MySQL archive. 

In this project we will convert this MySQL database of Enron emails into Avro format for analysis on Hadoop with Pig.

We hope that this dataset can become a sort of common set for examples and questions, as anonymizing one's own data in public forums can make asking questions and getting authoritative answers tricky.

More information about the Enron Emails is available:

[Document Classification on Enron Email Dataset](http://people.cs.umass.edu/~ronb/enron_dataset.html)
[Enron Scandal on Wikipedia](http://en.wikipedia.org/wiki/Enron_scandal)
[UC Berkeley Enron Email Analysis](http://bailando.sims.berkeley.edu/enron_email.html)

Setting up the Enron Database
-----------------------------

### Installing MySQL

MySQL is a simple but powerful open-source database. MySQL 5.5 is available [here](http://dev.mysql.com/downloads/), and [easy setup instructions](http://dev.mysql.com/doc/refman/5.5/en/installing.html) are available.

### Download and Load the Enron Emails

A MySQL 5.5 compatible version of the emails is available [here](https://s3.amazonaws.com/rjurney_public_web/images/enron.mysql.5.5.20.sql.gz).

Download the database archive, unpack it, create the enron database and then load the archive into it.

    [bash]$ tar -xvzf enron.mysql.5.5.20.sql.gz
    [bash]$ mysql -u root -e 'create database enron'
    [bash]$ mysql -u root < enron.mysql.5.5.20.sql

This will take some time, as the data is loaded into many tables with relationships and indexes. This is a good example of how structured, relational data works. Data processing in relational databases is front-loaded to optimize query performance later. With Hadoop and Pig, semi-structured or un-structured data is processed in parallel on many machines via [MapReduce](http://research.google.com/archive/mapreduce.html). We might say that concurrency on many networked machines replaces indexes on one machine. This allows us to manipulate many kinds of data in an ad hoc fashion without having to rigorously structure it before processing.

### Inspect and Query the Emails

    [bash]$ mysql -u root enron
    
    mysql> show tables;
    
    +-----------------+
    | Tables_in_enron |
    +-----------------+
    | bodies          |
    | categories      |
    | catgroups       |
    | edgemap         |
    | edges           |
    | headers         |
    | mailgraph       |
    | messagecats     |
    | messages        |
    | people          |
    | recipients      |
    +-----------------+
    11 rows in set (0.00 sec)

As we can see, this data is highly structured.
    
    mysql> select * from messages limit 1;
    +-----------+----------------------------------------------+---------------------+-----------+----------+--------------------+
    | messageid | smtpid                                       | messagedt           | messagetz | senderid | subject            |
    +-----------+----------------------------------------------+---------------------+-----------+----------+--------------------+
    |         1 | <2614099.1075839927264.JavaMail.evans@thyme> | 2001-10-31 05:23:56 | -0800 PST |        1 | Path 30 mitigation |
    +-----------+----------------------------------------------+---------------------+-----------+----------+--------------------+
    1 row in set (0.01 sec)
    
Querying a single email to return it as a document we might see in our inbox is complex. And yet this is precisely the format that is most convenient for analysis. This is the limitation of highly structured, relational data. elect a single email as we might view it in raw format.

    mysql> select m.smtpid as id, 
           m.messagedt as date, 
           s.email as sender,
           (select group_concat(CONCAT(r.reciptype, ':', p.email) SEPARATOR ', ') from recipients r join people p ON r.personid=p.personid where r.messageid = 511) as to_cc_bcc,
           m.subject as subject, 
           SUBSTR(b.body, 1, 200) as body
                from messages m 
                join people s
                    on m.senderid=s.personid
                join bodies b 
                    on m.messageid=b.messageid 
                        where m.messageid=511;
                        
    +-----------------------------------------------+---------------------+----------------------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+-------------------------------------+--------------------------------------------------------------------------------------------------------------+
    | id                                            | date                | sender               | to_cc_bcc                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | subject                             | body                                                                                                         |
    +-----------------------------------------------+---------------------+----------------------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+-------------------------------------+--------------------------------------------------------------------------------------------------------------+
    | <25772535.1075839951307.JavaMail.evans@thyme> | 2002-02-02 12:56:33 | pete.davis@enron.com | to:pete.davis@enron.com cc:albert.meyers@enron.com cc:bill.williams@enron.com cc:craig.dean@enron.com cc:geir.solberg@enron.com cc:john.anderson@enron.com cc:mark.guzman@enron.com cc:michael.mier@enron.com cc:pete.davis@enron.com cc:ryan.slinger@enron.com bcc:albert.meyers@enron.com bcc:bill.williams@enron.com bcc:craig.dean@enron.com bcc:geir.solberg@enron.com bcc:john.anderson@enron.com bcc:mark.guzman@enron.com bcc:michael.mier@enron.com bcc:pete.davis@enron.com bcc:ryan.slinger@enron.com | Schedule Crawler: HourAhead Failure | 

    Start Date: 2/2/02; HourAhead hour: 11;  HourAhead schedule download failed. Manual intervention required. |
    +-----------------------------------------------+---------------------+----------------------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+-------------------------------------+--------------------------------------------------------------------------------------------------------------+
    1 row in set (0.04 sec)
    

With our data in Avro format, we'll be able to more easily access email as documents to analyze both their structured and unstructured components with whatever tools we prefer.

### Dumping MySQL

We can run that same query to dump the results as TSV, or "Tab Separated Value." MySQL's mysql client allows us to dump a query as TSV using the -e and -B options. -e 

    [bash]$ mysql -u root -B -e "select m.smtpid as id, m.messagedt as date, s.email as sender, (select group_concat(CONCAT(r.reciptype, ':', p.email) SEPARATOR ', ') from recipients r join people p ON r.personid=p.personid where r.messageid = 511) as to_cc_bcc, m.subject as subject, SUBSTR(b.body, 1, 200) as body from messages m join people s on m.senderid=s.personid join bodies b on m.messageid=b.messageid;" enron > dump.sql 

We can now load our sql dump in Pig. I prefer to use several parameters when I use Pig in local mode. The '-l /tmp' option lets me put my pig logs in /tmp so they don't clutter my working directory. '-x local' tells Pig to run in local mode instead of Hadoop mode. '-v' enables verbose output, and '-w' enables warnings. These last two options are useful for debugging problems when working with a new dataset.

    [bash]$ pig -l /tmp -x local -v -w
    grunt> emails = LOAD 'dump.sql' AS (message_id:chararray, date:chararray, from:chararray, to_cc_bcc:chararray, subject:chararray, body:chararray);
    grunt> illustrate emails
    
        -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
    | emails     | message_id:chararray                          | date:chararray      | from:chararray          | to_cc_bcc:chararray                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | subject:chararray    | body:chararray                                                                                                                                                                                                 | 
    -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
    |            | <28471049.1075863303407.JavaMail.evans@thyme> | 2001-08-09 06:51:37 | michael.tully@enron.com | to:pete.davis@enron.com, cc:albert.meyers@enron.com, cc:bill.williams@enron.com, cc:craig.dean@enron.com, cc:geir.solberg@enron.com, cc:john.anderson@enron.com, cc:mark.guzman@enron.com, cc:michael.mier@enron.com, cc:pete.davis@enron.com, cc:ryan.slinger@enron.com, bcc:albert.meyers@enron.com, bcc:bill.williams@enron.com, bcc:craig.dean@enron.com, bcc:geir.solberg@enron.com, bcc:john.anderson@enron.com, bcc:mark.guzman@enron.com, bcc:michael.mier@enron.com, bcc:pete.davis@enron.com, bcc:ryan.slinger@enron.com | File Restoration     | I have already asked Paul about this, but I wanted to send this as kind of a reminder.\nDavid Porter from Real Time would like the following files restored from Tape Backup:\n\nP:\\RealTime\\Increment\\Wind | 
    -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

Now we've got our data in document format in Pig, with a schema. Lets save our data in Avro format to persist this schema. To do so, we need to register the jars that Avro needs, as well as Piggybank for the AvroStorage UDF itself. We'll also define a short form of the AvroStorage command, as the fully qualified name is java-long.

    grunt> /* Piggybank */
    grunt> register /me/pig/contrib/piggybank/java/piggybank.jar
    grunt> 
    grunt> /* Load Avro jars and define shortcut */
    grunt> register /me/pig/build/ivy/lib/Pig/avro-1.5.3.jar
    grunt> register /me/pig/build/ivy/lib/Pig/json-simple-1.1.jar
    grunt> register /me/pig/build/ivy/lib/Pig/jackson-core-asl-1.7.3.jar
    grunt> register /me/pig/build/ivy/lib/Pig/jackson-mapper-asl-1.7.3.jar
    grunt> register /me/pig/build/ivy/lib/Pig/joda-time-1.6.jar
    grunt> 
    grunt> define AvroStorage org.apache.pig.piggybank.storage.avro.AvroStorage();
    grunt> STORE emails INTO '/tmp/enron' USING AvroStorage();

Pig generates as many segments as there are mappers. In this case, 0-6 or 7 mappers.

    [bash]$ ls /tmp/enron/

    part-m-00001.avro       part-m-00004.avro       
    part-m-00002.avro       part-m-00005.avro       
    part-m-00000.avro       part-m-00003.avro       part-m-00006.avro

We can cat these Avro encoded files using a simple python utility I wrote, called [cat_avro](https://github.com/rjurney/Collecting-Data/blob/master/src/python/cat_avro). A less robust Ruby version of cat_avro is available [here](https://github.com/rjurney/Collecting-Data/blob/master/src/ruby/bin/cat_avro).

The script uses the Python Avro library, and is pretty simple:

    from avro import schema, datafile, io
    
    ...
    
    for record in df_reader:
      if i > 20:
        break
      i += 1
      if field_id:
        pp.pprint(record[field_id])
      else:
        pp.pprint(record)

    [bash]$ cat_avro /tmp/enron/part-m-00001.avro
    
    ...
    
    {u'body': u'Please use contract .2774,  acitivity 852981, for the NYPA volumes we \\ndiscussed.\\n\\nAlso, let me know what the exact volume is after you do the allocation.',
     u'date': u'2001-03-02 01:12:00',
     u'from': u'chris.germany@enron.com',
     u'message_id': u'<22603533.1075853865696.JavaMail.evans@thyme>',
     u'subject': u'NYPA for Feb',
     u'to_cc_bcc': u'to:pete.davis@enron.com, cc:albert.meyers@enron.com, cc:bill.williams@enron.com, cc:craig.dean@enron.com, cc:geir.solberg@enron.com, cc:john.anderson@enron.com, cc:mark.guzman@enron.com, cc:michael.mier@enron.com, cc:pete.davis@enron.com, cc:ryan.slinger@enron.com, bcc:albert.meyers@enron.com, bcc:bill.williams@enron.com, bcc:craig.dean@enron.com, bcc:geir.solberg@enron.com, bcc:john.anderson@enron.com, bcc:mark.guzman@enron.com, bcc:michael.mier@enron.com, bcc:pete.davis@enron.com, bcc:ryan.slinger@enron.com'}

    Avro Schema: {"fields": [{"doc": "autogenerated from Pig Field Schema", "type": ["null", "string"], "name": "message_id"}, {"doc": "autogenerated from Pig Field Schema", "type": ["null", "string"], "name": "date"}, {"doc": "autogenerated from Pig Field Schema", "type": ["null", "string"], "name": "from"}, {"doc": "autogenerated from Pig Field Schema", "type": ["null", "string"], "name": "to_cc_bcc"}, {"doc": "autogenerated from Pig Field Schema", "type": ["null", "string"], "name": "subject"}, {"doc": "autogenerated from Pig Field Schema", "type": ["null", "string"], "name": "body"}], "type": "record", "name": "TUPLE_0"}

cat_avro prints 20 records and then the schema of the records. Note that a schema is included with each file, so that it lives with the data. This is convenient. From now on we don't have to cast our data as we load it like we did before.

    




