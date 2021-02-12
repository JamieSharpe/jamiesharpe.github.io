---
layout: article
title: CSV - SQLite Extraction and Conversion
tags: Parsing Data
aside:
    toc: true
---

This is a follow-up post from [CSV (siː ɛs viː)](/CSV-Parsing) to show examples on how to write data with python into a CSV formatted file. There will be some references made to the previous post, so be sure to take a glance there first.

# Python and its CSV module

Python already comes with a CSV parsing library, which makes our life so much easier. The nuances from the previous post (almost) all go away!

The documentation on the Python csv module can be read [here](https://docs.python.org/3/library/csv.html). But I hope this post extracts some of the core fundamentals, and provides an example on how to effectively use it. I'm in the digital forensic field, so the focus will be to expand upon one of [Josh Brunty's](https://twitter.com/joshbrunty) scripts they shared [here](https://twitter.com/joshbrunty/status/1359622811321589762).

We are going to look at:

1. Writing data to a CSV file with Python.
2. Extracting data from an SQLite database with Python.
3. Generating a CSV file from parsing a SQLite database with Python.

# Sample CSV writer

Previously, we looked at how to hand craft a CSV file into a supported format to be imported into other software. We learnt there are a few ’‘gotchas’ when dealing with special characters. Rather than dealing with all the pitfalls in formatting, let the Python language, and its inbuilt libraries, do the heavy lifting!

Want to write a basic CSV file in Python? Let's dive straight into the code:

{% highlight python linenos %}
import csv

with open('names.csv', mode='w', newline='') as csv_file:

    # Write the header row
    fieldnames = ['First Name', 'Last Name', 'Quote', 'Favourite Colour']
    writer = csv.DictWriter(csv_file, fieldnames=fieldnames)
    writer.writeheader()

    # Construct our data
    person = {
        'First Name': 'Riley',
        'Last Name': 'Meyer',
        'Quote': 'To be, or not to be, that is the question',
        'Favourite Colour': 'Blue'
    }

    # Write our data to the file.
    writer.writerow(person)
{% endhighlight %}

Here is a very basic sample on how Python can be used to produce a CSV file. The keen eyed will recognise the data set is from the previous post. This example produces the following file:

{% highlight text linenos %}
First Name,Last Name,Quote,Favourite Colour
Riley,Meyer,"To be, or not to be, that is the question",Blue
{% endhighlight %}

Notice how the fields in the data have been automatically formatted to accommodate the commas, not to be confused with delimiter.

To keep with the previous post, here's the imported file in Excel:

![names.csv imported into Excel](/images/2021-02-12/names.csv%20-%20excel.png)

## Breakdown

If you understand the code, great, you can skip this section. But let’s have a look at the different parts of the code.

Quickly running over some of the basics, we `import csv`, as we are going to use this library.
Using a [context manager](https://www.python.org/dev/peps/pep-0343/) we [open]( https://docs.python.org/3/tutorial/inputoutput.html#reading-and-writing-files) a file `names.csv`: Setting the file access `mode` to `w`rite, and the `newline` to the empty string `''` following the guidelines from the [Python CSV documentation](https://docs.python.org/3/library/csv.html#csv.writer).

The `fieldnames` array of strings acts in two parts. It is used to form the column headers for the CSV file, as well as the dictionary lookup keys for the data. The `writer` variable holds the `csv.DictWriter` object that is setup to use the opened file, and to expect the `fieldnames` access key format for the data we are going to provide. This is followed by writing the first row containing the header fields to the file.

Now we move on to constructing the data to be inserted into the CSV file. A `person` dictionary is created, where the keys match those defined in `fieldnames`, and for each key, we give it a respective value.

The final step, we write the value of `person` into the CSV file with `writer.writerow(person)`. The writer object handles the dictionary key lookup with the `fieldnames` we provided it.

## New lines

This also works when the field contains newlines, quote characters, or any other type of special character.

Here's a look at the escaping in action:

The Python code:

{% highlight python linenos %}
import csv

with open('names.csv', mode='w', newline='') as csv_file:

    # Write the header row
    fieldnames = ['First Name', 'Last Name', 'Quote', 'Favourite Colour']
    writer = csv.DictWriter(csv_file, fieldnames=fieldnames)
    writer.writeheader()

    # Construct our data
    person = {
        'First Name': 'Riley',
        'Last Name': 'Meyer',
        'Quote': '"To be, or not to be, that is the question"\n- opening phrase of a soliloquy given by Prince Hamlet',
        'Favourite Colour': 'Blue'

    }

    # Write our data to the file.
    writer.writerow(person)
{% endhighlight %}

The file is produces:

{% highlight python linenos %}
First Name,Last Name,Quote,Favourite Colour
Riley,Meyer,"""To be, or not to be, that is the question""
- opening phrase of a soliloquy given by Prince Hamlet",Blue
{% endhighlight %}

And here it is correctly imported into Excel:

![names.csv with newlines and quotes imported into Excel](/images/2021-02-12/names.csv%20-%20newlines%20and%20quotes%20-%20excel.png)

# Parsing data from one source to a CSV destination

The above examples only show the syntax and structure on how to create a CSV file from data. But this is hard coded and not very practical; we can do better than that.

Let’s look at the sample provided by [Josh Brunty](https://twitter.com/joshbrunty/status/1359622811321589762). For blog post completeness, the file can be seen here:

{% highlight python linenos %}
#!/usr/bin/env python3
#: Title        : iphone_sms
#: Author       : "Josh Brunty" <josh [dot] brunty [at] marshall [dot] edu>
#: Date         : 05/04/2011
#: Version      : 1.0.2
#: Description  : Dump/interpret iphone sms.db messages table 
#: Options      : None

#: 03/22/2011   : v1.0.0 Initial Release
#: 04/12/2011   : v1.0.1 updated flags translations
#: 05/04/2011   : v1.0.2 added extended output formats, updated code schema
#: 10/05/2011   : v2.0.0 converted to python from bash


import sqlite3, argparse
from time import strftime, localtime, gmtime

sms_flags = {
    2 : 'Recd SMS',
    3 : 'Sent SMS/MMS',
    4 : 'Recd MMS',
    33 : 'Unsent' ,
    35 : 'Failed Send',
    129 : 'Deleted'}
read_flags = {
    0 : 'Unread',
    1: 'Read' }

def printdb(args):
    '''Prints the rows from the iPhone sms.db, interpreting the flags.'''
    
    if not args.noheader:
        print('File: "{}"'.format(args.database))
        print('Record #,Date,Type,Phone Number,AdressBook ID,Duration')
    
    try: 
        conn = sqlite3.connect(args.database)
        c = conn.cursor()

        for ROWID, address, date, text, flags, read in c.execute(
            'select ROWID, address, date, text, flags, read from message'):

            #convert flags object to sms_flag dictionary value
            type = sms_flags.get(flags, 'Unknown')
            
            #convert read object to read_flags dictionary value
            status = read_flags.get(read, 'Unknown') 
    
            #convert timestamp to local time or utc
            if args.utc:
                time = strftime('%Y-%m-%d %H:%M:%S (UTC)', gmtime(date))
            else:
                time = strftime('%Y-%m-%d %H:%M:%S (%Z)', localtime(date))

            print('{},{},{},{},"{}",{}'.
                format(ROWID, time, type, address, text, status))

    except sqlite3.Error:
            print('SQLite Error: wrong or incompatible database')

def main():
    parser = argparse.ArgumentParser(
        description='Process iPhone SMS database.',
        epilog='Converts timestamps to local time and interprets flag values.  \
        Prints to stdout.')

    parser.add_argument('database', help='an iPhone sms.db database')
    parser.add_argument('-n', '--no-header', dest='noheader',
        action='store_true', help='do not print filename or column header')
    parser.add_argument('-u', '--utc', dest='utc', action='store_true',
        help='Show UTC time instead of local')
    parser.add_argument('-V', '--version', action='version',
        version='%(prog)s v2.0.0')

    args = parser.parse_args()

    printdb(args)

if __name__ == '__main__':
    main()
{% endhighlight %}

The script performs the following outlines operations:

1.	Parses the arguments for:
    1.	`database` - file path to extract data from.
    2.	`--no-header` - to determine if the header row is to be outputted.
    3.	`--utc` - time localisation setting.
    4.	`--version` to print the current script version.
2.	Opens an SQLite connection to the `database` file path.
3.	Queries the database’s `message` table and extracts the specified columns.
4.	Parses/Processes the returned row of data.
5.	Outputs the result.

Unfortunately, I don’t have a suitable `sms.db` file on hand to show a sample of the output when running this script. However, let’s use this code as a base as a reference on how to make your own.

## Creating a SQLite parser for sms.db

We are going to work with a sample `sms.db` [Joshua Hickman]( https://thebinaryhick.blog/2020/04/16/ios-13-images-images-now-available/) has created, and kindly provided access to, an iPhone SE iOS (v13.3.1) extraction that can be found here at [Digital Copora](https://downloads.digitalcorpora.org/corpora/mobile/ios_13_4_1/) (Warning: 9GB+ file size). You can find the file we are going to use at `iOS 13.4.1 Extraction.zip[\Extraction\Apple iPhone SE (GSM) Full Image - 13-4-1.tar[\Extraction\Apple iPhone SE (GSM) Full Image - 13-4-1\private\var\mobile\Library\SMS\sms.db]]`.

|File Name|MD5 Hash                        |SHA-1 Hash                              |
|---------|--------------------------------|----------------------------------------|
|sms.db   |8def954b34ff9e0ca40403d05366e030|877b5fb664bcd9f6906bfbfe81c03091b1a32f58|

Take the time to explore the database in a browser of your choice. Here it is open in [DB Browser for SQLite](https://sqlitebrowser.org/):

![sms.db loaded into DB Browser for SQLite.](/images/2021-02-12/sms.db - DBBrowser for SQLite-1.png)

We can’t write a CSV file if we don’t know how to parse the data. This isn’t a blog post on how to reconstruct the connections between tables, but here’s an overview of how the tables can be linked. The `message` table contains a plethora of information on each individual message sent and received. Each entry has a many to one relationship with the `chat` table. The foreign→primary key relationship is between `message.handle_id`→`chat.ROWID`. With a small query we can quickly parse the data into a useable format. 

{% highlight SQL linenos %}
SELECT
	message.ROWID as "ID",
	chat.ROWID as "Chat ID",
	chat.chat_identifier as "Chat Handle",
	message.date as  "Date",
	message.text as "Message Content",
	message.is_from_me as "From Account Owner"
FROM
	message
LEFT JOIN chat ON
	message.handle_id = chat.ROWID
ORDER BY
	message.date ASC
{% endhighlight %}

And here’s a screenshot of the results:

![sms.db message extraction query.](/images/2021-02-12/sms.db%20-%20DBBrowser%20for%20SQLite-2.png)

Now you could copy the results from your database viewer of choice, and paste them into a file, or directly into Excel to analyse, but that’s not why we’re here.

## Python sms.db SQLite Parser

Using Josh’s file as a basic structure, we can make a script to parse, and output the data we queried:

{% highlight python linenos %}
import sqlite3


def print_db():
    sql_query = """
                SELECT
                    message.ROWID as 'ID',
                    chat.ROWID as 'Chat ID',
                    chat.chat_identifier as 'Chat Handle',
                    message.date as  'Date',
                    message.text as 'Message Content',
                    message.is_from_me as 'From Account Owner'
                FROM
                    message
                LEFT JOIN chat ON
                    message.handle_id = chat.ROWID
                ORDER BY
                    message.date ASC
                """

    conn = sqlite3.connect('sms.db')
    c = conn.cursor()

    print('ID, Chat ID, Chat Handle, Date, Message Content, From Account Owner')

    for message_id, chat_id, chat_handle, message_date, message_text, message_from_account in c.execute(sql_query):
        print(f'{message_id}, {chat_id}, {chat_handle}, {message_date}, {message_text}, {message_from_account}')


def main():
    print_db()


if __name__ == '__main__':
    main()
{% endhighlight %}

![Running Python script on sms.db](/images/2021-02-12/sms.db_parser.py.png)


The script follows a watered-down process model of Josh’s version. I've stripped out the exception handling, argument parsing, and hard coded the file name, to highlight the processing and output stages. The removed code is good practice, and I highly recommend writing software following good practices; although this is purely for a demonstration.

## Python sms.db SQLite to CSV

We can combine the sample CSV writer code at the start of this post, into our newly created script:

{% highlight python linenos %}
import sqlite3
import csv


def extract_db():
    sql_query = """
                SELECT
                    message.ROWID as 'ID',
                    chat.ROWID as 'Chat ID',
                    chat.chat_identifier as 'Chat Handle',
                    message.date as  'Date',
                    message.text as 'Message Content',
                    message.is_from_me as 'From Account Owner'
                FROM
                    message
                LEFT JOIN chat ON
                    message.handle_id = chat.ROWID
                ORDER BY
                    message.date ASC
                """

    conn = sqlite3.connect('sms.db')
    c = conn.cursor()

    with open('sms.db extraction.csv', mode = 'w', newline = '', encoding = 'utf-8') as csv_file:

        # Write the header row
        fieldnames = ['ID', 'Chat ID', 'Chat Handle', 'Date', 'Message Content', 'From Account Owner']
        writer = csv.DictWriter(csv_file, fieldnames = fieldnames)
        writer.writeheader()

        for message_id, chat_id, chat_handle, message_date, message_text, message_from_account in c.execute(sql_query):

            # Construct our data
            parsed_message = {
                'ID': message_id,
                'Chat ID': chat_id,
                'Chat Handle': chat_handle,
                'Date': message_date,
                'Message Content': message_text,
                'From Account Owner': message_from_account,
            }

            # Write our data to the file.
            writer.writerow(parsed_message)


def main():
    extract_db()


if __name__ == '__main__':
    main()
{% endhighlight %}

The only difference made, is the `csv_file` object has been opened with `encoding = ‘utf-8’` to accommodate for special characters such as emojis used in the message content.

We can see the file contents here:

![CSV file contents.](/images/2021-02-12/sms.db_parser.py-2.png)

There’s one more thing we can do here, just to further push the reason we may want to create a script to extract and generate a CSV file. The `Date` column stores the data in nanoseconds relative to `CFAbsoluteTime`; you can find the documentation [here]( https://developer.apple.com/documentation/corefoundation/cfabsolutetime). We can easily convert this to something human readable. We can `import datetime`, and in the data construction section of our code we can change the `Date` key assignment to `'Date': datetime.datetime.fromtimestamp(978307200 + message_date//1e9),` . 

And now the generated CSV file contains human readable dates:

![CSV file contents with converted dates.](/images/2021-02-12/sms.db_parser.py-3.png)

And we can import the file into Excel; note the imported encoding is set to `utf-8`:

![CSV file imported into Excel](/images/2021-02-12/sms.db_parser.py-excel.png)

Here’s the revised script:

{% highlight python linenos %}
import sqlite3
import csv
import datetime


def extract_db():
    sql_query = """
                SELECT
                    message.ROWID as 'ID',
                    chat.ROWID as 'Chat ID',
                    chat.chat_identifier as 'Chat Handle',
                    message.date as  'Date',
                    message.text as 'Message Content',
                    message.is_from_me as 'From Account Owner'
                FROM
                    message
                LEFT JOIN chat ON
                    message.handle_id = chat.ROWID
                ORDER BY
                    message.date ASC
                """

    conn = sqlite3.connect('sms.db')
    c = conn.cursor()

    with open('sms.db extraction.csv', mode = 'w', newline = '', encoding = 'utf-8') as csv_file:

        # Write the header row
        fieldnames = ['ID', 'Chat ID', 'Chat Handle', 'Date', 'Message Content', 'From Account Owner']
        writer = csv.DictWriter(csv_file, fieldnames = fieldnames)
        writer.writeheader()

        for message_id, chat_id, chat_handle, message_date, message_text, message_from_account in c.execute(sql_query):

            # Construct our data
            parsed_message = {
                'ID': message_id,
                'Chat ID': chat_id,
                'Chat Handle': chat_handle,
                'Date': datetime.datetime.fromtimestamp(978307200 + message_date//1e9),
                'Message Content': message_text,
                'From Account Owner': message_from_account,
            }

            # Write our data to the file.
            writer.writerow(parsed_message)


def main():
    extract_db()


if __name__ == '__main__':
    main()
{% endhighlight %}

I encourage you to make a copy and try it yourself. Feel free to expand on it or use this as a template/guide to writing a parser for something else entirely.

# Remarks

This post scratches the surface of creating a CSV file with Python. Our next step is to look into skipping the middle man, CSV, and writing our parsed data directly into a `.xlsx` spreadsheet format.
