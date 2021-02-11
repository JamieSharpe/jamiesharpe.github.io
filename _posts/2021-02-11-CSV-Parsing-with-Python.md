---
layout: article
title: CSV (siː ɛs viː)
tags: Parsing Data
aside:
    toc: true
---

This is a followup post from [CSV-Parsing](/CSV-Parsing) to show examples on how to write data from python into a CSV formatted file. There will be some references made to the previous post, so to be sure to take a glance there first.

# Python and its CSV module

Python already comes with a CSV parsing library, which makes our life so much easier. The nuances from the previous post (almost) all go away!

The documentation on the python csv module can be read [here](https://docs.python.org/3/library/csv.html). But I hope this post extracts the core fundamentals, and provides examples on how to effectively use it. I'm in the digital forensic field, so the focus will be to expand upon one of [Josh Brunty's](https://twitter.com/joshbrunty) scripts they shared [here](https://twitter.com/joshbrunty/status/1359622811321589762).

# Sample CSV writer

Let's dive straight into the code:

{% highlight python linenos %}
import csv

with open('names.csv', 'w', newline='') as csv_file:

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

Here is a very basic sample on how python can be used to produce a CSV file from data. The keen eyed will recognise the data set is from the previous post. This example produces the following file:

{% highlight text linenos %}
First Name,Last Name,Quote,Favourite Colour
Riley,Meyer,"To be, or not to be, that is the question",Blue
{% endhighlight %}

Notice how the fields in the data has been automatically formatted to accomodate the commas, not to be confused with delimiter.

To keep with the previous post, here's the imported file in Excel:

![names.csv imported into Excel](/images/2021-02-11/names.csv%20-%20excel.png)

This also works when the field contains newlines, quote characters, or any other type of special character.

Here's a look at the escaping in action:

The python code:

{% highlight python linenos %}
import csv

with open('names.csv', 'w', newline='') as csv_file:

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

![names.csv with newlines and quotes imported into Excel](/images/2021-02-11/names.csv%20-%20newlines%20and%20quotes%20-%20excel.png)

# Parsing data from one source to a CSV destination

The above examples only show the syntax and structure on how to create a CSV file from data. But this is hard coded in; we can do better than that.

Lets look at the sample provided by [Josh Brunty](https://twitter.com/joshbrunty/status/1359622811321589762). I'm going to reduce the file down to the extracting, processing, and output sections.

{% highlight python linenos %}
import sqlite3
from time import strftime, gmtime

sms_flags = {
    2  : 'Recd SMS',
    3  : 'Sent SMS/MMS',
    4  : 'Recd MMS',
    33 : 'Unsent',
    35 : 'Failed Send',
    129: 'Deleted' }
read_flags = {
    0: 'Unread',
    1: 'Read' }


def printdb(args):
    '''Prints the rows from the iPhone sms.db, interpreting the flags.'''

    print('Record #,Date,Type,Phone Number,AdressBook ID,Duration')

    conn = sqlite3.connect('sms.db')
    c = conn.cursor()

    for ROWID, address, date, text, flags, read in c.execute(
            'select ROWID, address, date, text, flags, read from message'):

        # convert flags object to sms_flag dictionary value
        message_type = sms_flags.get(flags, 'Unknown')

        # convert read object to read_flags dictionary value
        status = read_flags.get(read, 'Unknown')

        time = strftime('%Y-%m-%d %H:%M:%S (UTC)', gmtime(date))

        print('{},{},{},{},"{}",{}'.format(ROWID, time, message_type, address, text, status))


def main():
    printdb()


if __name__ == '__main__':
    main()
{% endhighlight %}

I've stripped out the exception handling, argument parsing, and hard coded the file name to highlight the processing and output stages. The removed code is good practice, and a highly recommend writing software following good practices. This is purely for a demonstration.

Running this code will output CSV format-like content. But seeing what the basic example above shows, lets make it write the content directly to a CSV formatted file:

{% highlight python linenos %}
import sqlite3
import csv
import datetime

read_flags = {
    0: 'Unread',
    1: 'Read' }


def printdb():
    '''Prints the rows from the iPhone sms.db, interpreting the flags.'''

    conn = sqlite3.connect('sms.db')
    c = conn.cursor()

    with open('sms.db extraction.csv', 'w', newline='', encoding='utf-8') as csv_file:

        # Write the header row
        fieldnames = ['Record #', 'Date', 'Destination Phone Number', 'Message', 'Read Status']
        writer = csv.DictWriter(csv_file, fieldnames=fieldnames)
        writer.writeheader()

        for ROWID, address, date, text, read in c.execute('SELECT ROWID, account, date, text, is_read FROM message'):

            # Construct our data
            parsed_message = {
                'Record #': ROWID,
                'Date': datetime.datetime.fromtimestamp(978307200  + date/1000000000),
                'Destination Phone Number': address,
                'Message': text,
                'Read Status': read_flags.get(read, 'Unknown')
            }

            # Write our data to the file.
            writer.writerow(parsed_message)


def main():
    printdb()


if __name__ == '__main__':
    main()

{% endhighlight %}


The sample data provided by Digital Corpora's [iPhone extraction](https://digitalcorpora.s3.amazonaws.com/corpora/mobile/ios_13_4_1/ios_13_4_1.zip) was used. Some modifications were made to the original script to correctly parse the sample data. The difference is likely due to iPhone sms.db changes since the original author created their version.

Running the above modified code produced a file that can be seen imported into Excel here:

![sms.db extraction.csv imported into Excel](/images/2021-02-2021-02-11/sms.db%20extraction.csv%20-%20excel.png)

Also note the imported encoding is set to `utf-8`, both in the Excel import dialog and within the script, to support additional characters such as emojis.
