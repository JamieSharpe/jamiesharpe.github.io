---
layout: article
title: CSV (siː ɛs viː)
aside:
    toc: true
---

Comma Seperated Values (CSV) is a format used to store data within a file in the form of rows and columns for data exchange. 

This post is to show some of the nuances when dealing with CSV files.

The principle is very simple, but let us define the components of a CSV file:

* Field - The container that holds the value.
* Delimiter - What seperates each field on the same row (`,`).
* Quote Character - The character used to wrap a field containing special characters.
* Line Terminator - The character used to determing the end of a row of data (`\n`)

A common CSV file may look like this:

```
First Name,Last Name,Favourite Colour
Riley,Meyer,Blue
Bradley,Gibson,Green
```

Upon parsing, we can produce the following table:


| First Name | Last Name | Favourite Colour |
| ---------- | --------- | ---------------- |
| Riley      | Meyer     | Blue             |
| Bradley    | Gibson    | Green            |

Here's a screenshot of the file after being imported into Excel:

![alt text](/images/2021-02-10/sample.csv%20-%20excel.png)

In this sample, the delimiter is the comma (`,`) character, and each field is the data inbetween.

But what happens when we get a little creative? Lets add a quote column to our sample file.

```
First Name,Last Name,Quote,Favourite Colour
Riley,Meyer,To be, or not to be, that is the question,Blue
Bradley,Gibson,Aspire to inspire before we expire.,Green
```

What we want is the following:


| First Name | Last Name | Quote                                     | Favourite Colour |
| ---------- | --------- | ----------------------------------------- | ---------------- |
| Riley      | Meyer     | To be, or not to be, that is the question | Blue             |
| Bradley    | Gibson    | Aspire to inspire before we expire.       | Green            |

However this is how the data is parsed in Excel:

![alt text](/images/2021-02-10/sample%20incorrect.csv%20-%20excel.png)

Note how it picks up the next field from using the comma within the quote.

A similar result happens in LibreOffice Calc:

![alt text](/images/2021-02-10/sample%20incorrect.csv%20-%20calc.png)

One solution, is to change the delimiter character to something other than a comma. A character that is not used among the data set, such as a pipe (`|`). This may work if you are certain this delimiter character is not among the dataset, which may not always be the case, especially when dealing with data that has been entered by a human; chat logs, Twitter/Instagram biographies, etc.

The solution is to wrap the fields containing special characters that are used to structure the file with a quote charatcer such as double quotes (`"`).

Here's the revised CSV file:

```
First Name,Last Name,Quote,Favourite Colour
Riley,Meyer,"To be, or not to be, that is the question",Blue
Bradley,Gibson,Aspire to inspire before we expire.,Green
```

Here is the revised CSV file when imported into Excel:

![alt text](/images/2021-02-10/sample%20corrected%20-%20excel.png)

And here it is imported into LibreOffice Calc:

![alt text](/images/2021-02-10/sample%20corrected%20-%20calc.png)

But what happens if the field we are wrapping in the quote character, contains said quote character? Well, you just escape it by prepending it with the quote character.

```
First Name,Last Name,Quote,Favourite Colour
Riley,Meyer,"""To be, or not to be, that is the question"" - Prince Hamlet",Blue
Bradley,Gibson,"""Aspire to inspire before we expire."" - Eugene Bell Jr.",Green
```

Here it is imported into Excel:

![alt text](/images/2021-02-10/sample%20corrected%20with%20quotes%20-%20excel.png)

And in Calc:

![alt text](/images/2021-02-10/sample%20corrected%20with%20quotes%20-%20calc.png)

See how the Double quotes are escaped when parsed. The integrity of the data is maintained.

Wrapping a field in a quotation character is not limited to being able to use the delimiter character, but all special characters: new lines, tabs, commas, quotes, even a null terminator.


```
First Name,Last Name,Quote,Favourite Colour
Riley,Meyer,"""To be, or not to be, that is the question""
 - opening phrase of a soliloquy given by Prince Hamlet",Blue
Bradley,Gibson,"""Aspire to inspire before we expire.""
 - Quote by Eugene Bell Jr.",Green
```

Imported into Excel:

![alt text](/images/2021-02-10/sample%20corrected%20with%20quotes%20and%20newlines%20-%20excel.png)

And in Calc:

![alt text](/images/2021-02-10/sample%20corrected%20with%20quotes%20and%20newlines%20-%20calc.png)

See how content of the quote field is maintained. The excel preview doesn't display the new line character, but does parse it correctly. LibreOffice Calc's preview does show the new line character. The end result is identical, the content matches that of the CSV fields.

And just to show an extreme case of it working, lets go meta:

Here's a CSV file containing the content of all of the above CSV files along with the filename and a brief description.

```
File Name,Description,File Content
sample.csv,Basic CSV file.,"First Name,Last Name,Favourite Colour
Riley,Meyer,Blue
Bradley,Gibson,Green"
sample incorrect.csv,Contains incorrectly formatted CSV field.,"First Name,Last Name,Quote,Favourite Colour
Riley,Meyer,To be, or not to be, that is the question,Blue
Bradley,Gibson,Aspire to inspire before we expire.,Green"
sample corrected.csv,Corrected field.,"First Name,Last Name,Quote,Favourite Colour
Riley,Meyer,""To be, or not to be, that is the question"",Blue
Bradley,Gibson,Aspire to inspire before we expire.,Green"
sample corrected with quotes.csv,Inserting quotes.,"First Name,Last Name,Quote,Favourite Colour
Riley,Meyer,""""""To be, or not to be, that is the question"""" - Prince Hamlet"",Blue
Bradley,Gibson,""""""Aspire to inspire before we expire."""" - Eugene Bell Jr."",Green"
sample corrected with quotes and newlines.csv,And new lines.,"First Name,Last Name,Quote,Favourite Colour
Riley,Meyer,""""""To be, or not to be, that is the question""""
 - Opening phrase of a soliloquy given by Prince Hamlet"",Blue
Bradley,Gibson,""""""Aspire to inspire before we expire.""""
 - Quote by Eugene Bell Jr."",Green"
```

And here it is all correctly parsed into Excel:

![alt text](/images/2021-02-10/meta.csv%20-%20excel.png)

and LibreOffice Calc:

![alt text](/images/2021-02-10/meta.csv%20-%20calc.png)
