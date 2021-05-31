---
layout: post
title: Selecting Random Samples From Large SQL Server Tables
categories: sqlserver
---

Recently, there was a situation where we needed to select 10,000 random rows, from a table containing several hundred million rows.

Having thought about it for a little while, I couldn't think of a better solution than ordering by `NEWID()` and selecting the `TOP 10000` rows, but given the size of the table we're working with, it was very slow and inefficient.

Thankfully, Brent Ozar had a blog post to get us moving in the right direction: [How to Get a Random Row from a Large Table](https://www.brentozar.com/archive/2018/03/get-random-row-large-table/).

As Brent's post explains, the ordering by `NEWID()` is so slow because it calculated the NEWID value for every row in the table and then filters the return to the first 10,000 rows.

Ultimately, the best solution from Brent's post, involves getting the largest ID value from the table, then generating a random number and geting the modulo of that random number for the maximum ID value in the table and finally, selecting the record with that ID value.

This does require a table with an integer ID column and also, is not guaranteed to return a record, if the randomly selected ID value doesn't exist in the table.

For our purposes though, we had a `BIGINT` identity column and the very vast majority of ID values between 1 and the max value did exist, so this was perfect.

There is also a nod to our required use in Brent's post:

> If you wanted 10 rows, you’d have to call code like this 10 times (or generate 10 random numbers and use an IN clause.)

As we wanted 10,000 rows, we decided to use a [Tally table](https://www.sqlservercentral.com/articles/the-numbers-or-tally-table-what-it-is-and-how-it-replaces-a-loop-1), (called `dbo.Numbers` in this example) to get 10,000 rows and generate a `NEWID()` for each of those rows and store those 10,000 random numbers in a temporary table.

~~~ sql
/* Get the highest ID in the table to use
 * in the modulo calculation.
 */
DECLARE @MaxID BIGINT = (SELECT MAX(ID) FROM dbo.TableToSample);

/* Create a temporary table to store the
 * randomly selected IDs
 */
CREATE TABLE #RandomIDs (
    SampleID INT NOT NULL,
    RandomID BIGINT NOT NULL
)

/* To allow for any IDs no longer present in the table,
 * select 10% more than needed at this point.
 */
INSERT INTO #RandomIDs (RandomID)
SELECT n.N AS SampleID ABS(CHECKSUM(NEWID())) % MaxID AS RandomID
FROM dbo.Numbers AS n
WHERE n.N BETWEEN 1 AND 11000;

/* Select from the table to be sampled inner joined
 * with the temporary table to only return rows with
 * a randomly selected ID value, and only keep the 
 * first 10000 matches.
 */
SELECT TOP 10000 tts.ID, tts.Col1, tts.Col2
FROM dbo.TableToSample AS tts INNER JOIN
#RandomIDs AS rnd ON tts.ID = rnd.RandomID
ORDER BY rnd.SampleID;
~~~

**TODO** - Need performance numbers between this and Select Top 1000 Order By NEWID() implementation and a conclusion.