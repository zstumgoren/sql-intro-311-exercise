# SF 311

[311](https://en.wikipedia.org/wiki/3-1-1) is a special number you can call for non-emergency government services. Many cities offer some version of this, and increasingly you can file requests for service online and through mobile apps.

San Francisco provides its [311 data](https://data.sfgov.org/City-Infrastructure/311-Cases/vw6y-z8j6) through an online portal.

## Generating questions

What are some questions you might be able to explore with this data? 

> If you're hard pressed for ideas, review the data displayed in the online portal and cross-reference the data dictionary as you go.

Here are a few possibilities:

* What types of 311 calls are most prevelant?
* What neighborhoods have the highest incidence of 311 calls? 
* Have 311 calls increased over time? What patterns can we find for top causes over time?
* Which departments are handling the most calls?

## Fact-finding

Prior to beginning analysis or even loading the data into SQLite, let's spend some time on fact-finding. Review the main [data page](https://data.sfgov.org/City-Infrastructure/311-Cases/vw6y-z8j6) as well as the [FAQ](https://support.datasf.org/help/311-case-data-faq) in order to answer the below questions:

* Roughly how many records are in the database?
* What kind of reports should we expect to see in this data? What won't be included?
* What time span does the data cover?
* How frequently is the data updated?
* Are there potential duplicates in the data? If so, how can you qualify the language in a story when writing about this data?
* Any other data issues we should be aware of?

## Load the data

We've formulated some questions and have a basic handle on the nature of the data. Now let's load the data into sqlite so we can begin assessing data quality.

* Download the [311 data](https://data.sfgov.org/City-Infrastructure/311-Cases/vw6y-z8j6).
* Create a new database called `311.db` using DB Browser.
* Import the data into a table called `cases` using DB Browser's automatic import feature (`File -> Import -> Table from CSV file...`).

## Get to know the data

We've loaded the data. Now let's do some standard data vetting to get familiar with the data and assess its quality.

To that end, write SQL queries that can answer the below questions.

> Note, it's a good idea to save the queries as you go, for example in a file called `311.sql`.

* How many cases are in the data? Does the number match the figure presented on the data portal?
* Create a *distinct* list of `Category` values and order them alphabetically. What data issues, if any, do we see?

## Speeding up queries

Note that the last query took a minute or more to run (depending on your machine). 

Let's add a [database index](https://en.wikipedia.org/wiki/Database_index) on the `Category` field and re-run the query to see if it improves the execution time. While we're at it, let's add indexes for `RequestType` and `Neighborhood` since we'll be analyzing these fields as well.

An index can be added to the `cases` table by:

* Switching to the `Database Structure` panel
* Selecting the `cases` table
* Clicking the `Create index` button

Once on the index creation pop-up:

* Give the index a name, such as `category_index`
* Use the wizard to select the `Category` field and click the right arrow (`>`) to move choose it as the column to be indexed
* Click `OK` and wait for the index to be created (this will take a few seconds)

Repeat the above process for all three fields.

> NOTE: Creating indexes involves a trade-off between the speed of queries and size of the database. After adding indexes for the three fields mentioned above, the size of the SQLite database should have grown by roughly 300MB.

## Returning to our queries

Next, try running the `DISTINCT` query for the `Category` field to verify the effect of the index. It should now run *much* faster (i.e. less than 1 second!).

Perform the follow queries as well:

* Create a *distinct* list of `RequestType` values and order alphabetically. What data issues, if any, do we see?
* Create a *distinct* list of `Neighborhood` values and order alphabetically. What data issues, if any, do we see?


## Analysis tasks

We've identified some data quality issues and optimized our database for certain types of queries. 

Let's move forward with analysis of some of the questions mentioned above. This will typically require writing SQL queries with a `GROUP BY` clause.

### Top calls by type

Write SQL queries that answer the following questions:

1. What are the top three categories of calls?
1. What percentage of total calls does each category represent?

Here are a few hints to help with question 2:

* Try a [SELECT subquery][] to calculate the percentage.
* The numerator of the select subquery needs to be a decimal for the math to work properly.

[SELECT subquery]: https://a-gentle-introduction-to-sql.readthedocs.io/en/latest/part2/subqueries-revisited.html

### Unpacking the top category

Our analysis revealed that 1 in 3 calls are requests for `Street and Sidewalk Cleaning`. Let's do a little more digging into this particular category.

Write a SQL query that:

* Filters the data for only the `Street and Sidewalk Cleaning` category
* Counts the records by `RequestType` and orders the result from highest to lowest count.

> Hint: You'll need a `WHERE` class and `GROUP BY` for this question.

What are the top three request types? 

What data issues do we notice with the full list?
 
### Sidenote: Indexing for complex queries

Note that the last query took more than 30 seconds, even though we have separate indexes on both the `Category` and `RequestType` fields. 

If you prefix the query with [EXPLAIN QUERY PLAN][], you can see that the SQLite query is only relying on the Category index. So we're presumably taking a hit when the database groups and orders by `RequestType`.

If we add a "composite" (i.e. multi-field) index on both `Category` and `RequestType`, we can dramatically improve the performance of the query to less than 1 second of execution time.

Creating indexes for more complex queries -- in this case involving both a filter and a group by -- can be tricky. Be prepared to experiment in order to fine-tune execution time.

[EXPLAIN QUERY PLAN]: https://sqlite.org/eqp.html

### Human/Animal waste by Neighborhood

Human waste has become a [widely reported problem ](https://www.forbes.com/sites/adamandrzejewski/2019/04/15/mapping-san-franciscos-human-waste-challenge-132562-case-reports-since-2008/#7316f48a5ea5) in parts of San Francisco.

Write a SQL query that counts the reports of `Human or Animal Waste` by Neighborhood and sort from highest to lowest. For this query, you'll need to use the `RequestType` field.

Which three neighborhoods have the higest number of reports?

This SQL query could serve as a starting point for more fine-grained analysis, perhaps using the `RequestDetail` field to pinpoint cases of human waste (as opposed to animal).

> Note that the query may take more than 10 seconds to run, but hopefully less than 20 seconds. We could optimize the query as we did earlier by creating a multi-column index, but we'll hold off on that for now.

### Human/Animal waste over time

Let's examine the data to determine if there is a trendline in the `Human or Animal Waste` request type. This type of analysis could easily be applied to other categories and request types. 

#### Adding a year column

In order to perform a trend analysis of cases over time, let's extract the year value from the `Opened` field and add it to a new `opened_year` column.

* Go to the Database Structure tab in DB Browser * Click on the `cases` table
* Click `Modify table`
* Click `Add field`
* Enter `opened_year` as the field name and make sure the type is `Integer`
* Click `OK` (this may take a few minutes...)

Next, we'll need to populate this field by extracting the year from the `Opened` column for each record. 

Let's start by crafting a `SELECT` query that extracts the 4-digit year from the `Opened` column.

> Add a `LIMIT 10` clause to improve the run-time of the query.

Once you've figured out how to extract the year, write a SQL `UPDATE` statement that populates the new `opened_year` column.

Add an index called `opened_year_idx` using the new `opened_year` column.

#### Top 3 By Year 

Finally, write a SQL query that counts the number of Human or Animal Waste reports by `Neighborhood` and `opened_year` and sorts the data first by neighborhood (alphabetically), then by year (from lowest to highest).

The resulting table should have more than 1000 rows. You'll notice that there a number of entries where the Neighborhoods are `Null`.

Update the query to only include:

* the top three neighborhoods we pinpointed earlier (based on overall counts by Neighborhood)
* 2009 through 2019 (to avoid partial data for 2008 and 2020)

What if any trends can you notice for each of the neighborhoods by examining the table? 

## From DB Browser to Flourish

Try exporting the CSV and create a Flourish line chart that plots yearly values for all three neighborhoods.

This will require using a Pivot Table to aggregate and *reshape* the data into a form suitable for Flourish. 

You'll need to experiment in the PivotChart Fields panel to figure out where to put the `Neighborhood` and `opened_year` fields to get a correctly structured line chart.