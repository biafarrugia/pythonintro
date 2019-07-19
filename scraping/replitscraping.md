# Scraping in Python using repl.it

[Repl.it](https://repl.it/) is an online environment for writing and running code. It supports a wide range of languages, including Python. And most importantly of all, it allows you to install libraries such as `scraperwiki`, which is used for scraping and storing the resulting data.

In this tutorial I explain how to do that.

## Start a new repl

First, go to the [my repls](https://repl.it/repls) page and click on **new repl** in the main menu.

Choose **Python** for your repl, ignore the GitHub section (this is used if you want to import code from a GitHub repo), but give your repl a name like 'test scraper'.

Then click **Create Repl**.

You will be taken to that new repl. The screen is divided into 3 sections:

* On the left is the *Files* area. Currently there is just one file: `main.py`.
* In the middle is that file: `main.py`. There should be a cursor flashing on line 1, ready for you to add code to that file.
* On the right is the *console*, where you can see the output if and when you run the code in the file.

Most of your work is done in the middle area.

## The first lines of code: importing libraries

The first few lines of code import some **libraries** which we will need to solve particular problems. We need:

* The `scraperwiki` library to fetch webpages and also to store data
* The `lxml.html` library to convert those fetched webpages into something that has structure
* The `cssselect` library to drill down into that structure using css selectors
* The `pandas` library to analyse data and also convert to a CSV

Here's those first few lines:

```py
import scraperwiki
import lxml.html
import cssselect
#use pandas to convert to downloadable csv
import pandas as pd
```

## Storing and scraping the URL

Next we use some **variables** to store a URL and a **function** from one of those libraries to fetch a URL and store it:

```py
# Store the URL in a variable to make it easier to change
urltoscrape = "https://en.wikipedia.org/wiki/List_of_twin_towns_and_sister_cities_in_England"
#Store the base URL which we often need to add to relative links
baseurl = "https://en.wikipedia.org"
#Scrape the webpage at that URL into a new variable called 'html'
html = scraperwiki.scrape(urltoscrape)
#Print the contents of that new object
print(html)
```

The function `scrape()` from the `scraperwiki` library (hence `scraperwiki.scrape()`) is used to fetch a webpage from a given URL (which is stored in the variable `urltoscrape`).

At the end of the code we `print()` the results of that function (stored as `html` but you could give that variable any name).

You should see a long string of characters - basically all the HTML for that webpage grabbed from that URL.

We need to drill down to something more specific.

## Drilling down into the HTML

Our next section of code does that using functions from another two libraries:

```py
#Convert that html variable into a new lxml.html object
#(this means it is structured like a tree so we can use lxml.html functions on it)
root = lxml.html.fromstring(html)
# Find something on the page using css selectors
#Specifically anything within a dd tag within a dl tag within a dd tag within a dl tag
#Store the results in a variable we call lis - it'll always be a list
lis = root.cssselect('dl dd dl dd')
#Show how many items are in that new list (how many matches)
print(len(lis))
```

The `.fromstring()` function converts something from a string of characters into an lxml *object*. In this case it is taking the HTML code, which was stored in the variable `html`, and converting that.

That object - `root` is then used with `.cssselect()` to extract elements within particular CSS selectors. The CSS selector `dl dd dl dd` means "anything within a dd tag within a dl tag within a dd tag within a dl tag".

Finally we use the `print()` command to show us how many matches it found.

## Start storing the data

So we've grabbed a bunch of matches from the HTML where text is within a dd tag within a dl tag within a dd tag within a dl tag.

But we need to store each of those matches.

To do that we need to *loop* through the list of matches and store each in turn. Here's the code:

```py
#create a dictionary to store what we are about to extract
record = {}
#loop through those lis
for li in lis:
  #Print the text within that tag and any child tags
  print(li.text_content())
  #This next line is an alternate troubleshooted version in case there are odd characters that cause an error
  #print(li.text_content().encode('utf-8').strip())
  #Store that in the dictionary, under the key 'address'
  record['address'] = li.text_content()
  record['country'] = li.text_content().split(',')[-1]
  #If we were scraping <a > links and wanted to extract the href attribute, we would use this
  #detaillink = baseurl+li.attrib['href']
  #record['link'] = detaillink
  #Save the whole dictionary as a new row in a table in a sqlite database
  scraperwiki.sqlite.save(['address'],record, table_name='twintowns')
```

The list of matches is called `lis`. As we loop through it we call the match `li`.

We then extract a certain quality of that match. For example to get the text within it, we use `.text_content()`.

In one case we also split that text on any commas by adding `.split(',')`. Because that creates a list (a single item is split into many), ann *index* is used to extract the *last* item in that list: `[-1]`

Right at the start of the code you'll notice we created a variable called `record` like so: `record = {}`

The `{}` indicates that this is a dictionary object, which is useful for storing data. Later on we add data to it like so:

`record['address'] =`

Once we have added what we want to that dictionary, we save the whole dictionary in turn as a row in a table:

`scraperwiki.sqlite.save(['address'],record, table_name='twintowns')`

There are quite a few things happening here, so let's break them down:

First, `scraperwiki.sqlite.save()` saves a data record to a table in a sqlite database. It needs 3 ingredients:

* The unique key of the table - this is `['address']`
* The name of the dictionary object with the data for the row. That is `record`
* The name of the table, specified with the `table_name=` parameter. The name here is `twintowns`

## Accessing that database

If we want to save this as a CSV file we need to get it back out of the database, and that's the last step. Here's the code:

```py
#print the results of querying that table in that database
print(scraperwiki.sql.select("count(*) from twintowns"))
#Store the results of another query
polandtowns = scraperwiki.sql.select("* from twintowns where address LIKE '%Poland%'")
#Print the contents of that variable
print(polandtowns)
#Convert that list of dicts into a pandas dataframe
polandtownsdf = pd.DataFrame(polandtowns)
#Export as a CSV file using the .to_csv function https://pandas.pydata.org/pandas-docs/stable/reference/api/pandas.DataFrame.to_csv.html
polandtownsdf.to_csv('out.csv')
```

To get data out of a database you can *query* it. That's what this is:

`scraperwiki.sql.select("* from twintowns where address LIKE '%Poland%'")`

If you wanted to get all data you would just specify

`scraperwiki.sql.select("* from twintowns")`

This means 'select all from the table called twintowns'

But the example in the code adds a filter: `where address LIKE '%Poland%`

This limits the results to those containing the characters 'Poland'

The results of the query are put in a variable called `polandtowns`

Next, the `.DataFrame()` function from the `pandas` library (called `pd` for short in the code) converts that data to a data frame object. That is stored in `polandtownsdf`:

`polandtownsdf = pd.DataFrame(polandtowns)`

Finally, the `.to_csv()` function creates a CSV from the dataframe that is attached to it with a period. You just need to specify the name including ".csv".

`polandtownsdf.to_csv('out.csv')`

## The final code

```py
import scraperwiki
import lxml.html
import cssselect
#use pandas to convert to downloadable csv
import pandas as pd

# Store the URL in a variable to make it easier to change
urltoscrape = "https://en.wikipedia.org/wiki/List_of_twin_towns_and_sister_cities_in_England"
#Store the base URL which we often need to add to relative links
baseurl = "https://en.wikipedia.org"
#Scrape the webpage at that URL into a new variable called 'html'
html = scraperwiki.scrape(urltoscrape)
#Print the contents of that new object
print(html)

#Convert that html variable into a new lxml.html object
#(this means it is structured like a tree so we can use lxml.html functions on it)
root = lxml.html.fromstring(html)
# Find something on the page using css selectors
#Specifically anything within a dd tag within a dl tag within a dd tag within a dl tag
#Store the results in a variable we call lis - it'll always be a list
lis = root.cssselect('dl dd dl dd')
#Show how many items are in that new list (how many matches)
print(len(lis))
#create a dictionary to store what we are about to extract
record = {}
#loop through those lis
for li in lis:
  #Print the text within that tag and any child tags
  print(li.text_content())
  #This next line is an alternate troubleshooted version in case there are odd characters that cause an error
  #print(li.text_content().encode('utf-8').strip())
  #Store that in the dictionary, under the key 'address'
  record['address'] = li.text_content()
  record['country'] = li.text_content().split(',')[-1]
  #If we were scraping <a > links and wanted to extract the href attribute, we would use this
  #detaillink = baseurl+li.attrib['href']
  #record['link'] = detaillink
  #Save the whole dictionary as a new row in a table in a sqlite database
  scraperwiki.sqlite.save(['address'],record, table_name='twintowns')

#print the results of querying that table in that database
print(scraperwiki.sql.select("count(*) from twintowns"))
#Store the results of another query
polandtowns = scraperwiki.sql.select("* from twintowns where address LIKE '%Poland%'")
#Print the contents of that variable
print(polandtowns)
#Convert that list of dicts into a pandas dataframe
polandtownsdf = pd.DataFrame(polandtowns)
#Export as a CSV file using the .to_csv function https://pandas.pydata.org/pandas-docs/stable/reference/api/pandas.DataFrame.to_csv.html
polandtownsdf.to_csv('out.csv')

```
