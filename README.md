# dbluigi

A tool based on luigi for doing ETL transformations of a database into another database. Currently this is just vaporware, but once I have fleshed out the idea to see if the tool makes sense and isn't already covered by something else, I might start building it in my copious spare time.

## Background

As part of the work on the [FBI Crime Explorer API](https://github.com/18F/crime-data-api), we discovered rather quickly that databases optimized to reduce redundancy (via the [Fifth Normal Form](https://en.wikipedia.org/wiki/Fifth_normal_form) for instance) often perform terribly when used for API responses. This is because the database server has to do a lot of joins and other operations to turn normalized data into cohesive denormalized records. For instance, if there was a table with basic information about a police agency, a DB query might involve pulling in records from tables representing the agency name, agency type, city information, state information, recent reporting participation, etc. Furthermore, some of these joins might involve subqueries that use only some of the data in a table (for instance, we might have multiple years of police staffing for an agency, but we really only want to show the most recent for the `/agencies` response). To optimize these API queries, we often found ourselves building denormalized views that reassembled records for a single entity (like an Agency) split across multiple tables. These tables are inefficient from a storage perspective (there is a lot of redundancy), but they do perform much better in an API context.

Similarly, we spent a lot of time and effort building rollups from data that is reported at a more granular scale. For instance, we might have a table that stores monthly counts of a specific criminal offense reported by an agency (or another reporting system that provides specific incidents). But for our API and our web interface, we want to just display the annual totals for that offense. Or we want to see a histogram of offender ages for all burglaries in a state in a given year. These rollups were an essential part of the CDE interface, but they were often quite expensive to create, both conceptually and in terms of execution, with some of them taking multiple days to run.

Finally, there were some circumstances where we needed to make some alterations to how data was presented on the site. THis might include scrubbing fields that should not ever be visible (ie, if they contain PII and the database has the potential to be leaked) or simpler situations like changing the name of a police department to match its actual name or fixing some problematic geocoding. In these cases, it's best to never make alterations to the original database tables (because then you don't know how the data was provided to you) and instead, alter your derived tables instead. So, do not edit the police agency name in the original database, but definitely edit it in the denormalized version of that table.

We made sure that none of these data transformations or alterations were manual. Every one of them was implemented as a SQL file that could be run on a local subset of the database for consistency and on the production database. But otherwise, this became a pretty unwieldy process as we added more and more ["after load"](https://github.com/18F/crime-data-api/blob/master/dba/after_load.sh) transformations that would need to be run after a fresh copy of the database was loaded. One common problem is that some transformations would take days to run and possibly fail during the process, requiring a lot of mental energy and babysitting from developers to make sure they completed. In addition, there were complicated dependencies between various steps that are not really represented within the database itself or documented anywhere. Most of these do not fit into the model of primary/foreign keys; for instance, a few of the rollup reports we generated were stipulated that they should only include the counts from agencies that submitted 12 months of data for that calendar year. This is easy enough to build as a workflow: first make an `agency_reporting` table with the number of months a given agency reported in a year and only join in records from agencies for which the `months_reported=12` in a given year. But it's hard to figure out what downstream tables you might need to regenerate if you find a bug in how you computed reporting. Worse still, we actually reimplemented this calculation in several distinct places because we split things up conceptually into many SQL files.

Worse still, failure could strike at any point in the process and it often involved developers having to manually run parts of a script to continue where the SQL process failed. We got a bit better at things as we went on (so for instance, we started a new pattern where we would create a table like `cde_agencies_temp`, do all the stuff we needed to populate it, then when it was good to go, dropping the existing `cde_agencies` table and renaming the temp table to `cde_agencies`. This let us keep the data process semi-automatic at times if we need it (ie, run all the way up to replacing the table) and allowed us to do some sanity checks on the temp table's data.

Finally, the testing for these transformations was often rather slapdash. Because these transformations were being made on the same database, usually the best we could do for testing was unit tests on the derived tables we might make. This required us to commit test DB data in the code (which was rebase hell) and we couldn't really use this for validating the process in production. A better approach might be one that lets you apply some testing as part of the transformation script, althought the exact mechanism for this would have to be determined (and could potentially be slow). But it would be cool if you had some process for validating and failing a transformation step if it doesn't match what you expect (ie, if the database table suddenly has 0 rows or less rows than it had before or every county agency should have " County Police Department" added to its name unless it already has "County" in its name already. Conceivably, such validations could be specified as either SQL (for cases where you need to check each row on a large table) or as Python (if you had specific fixtures on a test DB). But this is something down the road.

The current process we had did check off some important requirements: it was executable and it was idempotent. But it wasn't very good at tracking failure, sandboxing the API DB from the raw source, keeping an audit trail of its actions, or the like. A better process might formalize some of the things that were handled on an ad hoc basis: testing, dependencies, failure logging, even performance auditing. It would create tables in a separate database from the source data. Ultimately, we would still have a system that runs SQL commands, but it would be more automated in how it does that (for small databases, we could even do continuous ETL if we wanted).

## Could We Use Luigi?

This seems like a good case to use something like [Luigi](https://github.com/spotify/luigi) or [Airflow](https://github.com/apache/incubator-airflow), since both are Python open-source libraries for representing workflows as directed acyclic graphs. I am currently leaning towards Luigi however, since it is a bit more like a "distributed make" in functionality and seems to have a smaller footprint than the impressive Airflow. These libraries would allow us to specify dependencies between tasks, and run tasks in the proper order to ensure such dependencies are observed. They can handle tasks that take a long time to execute and capture failures. And they give us a mechanism for representing complicated ETL workflows with very granular elements.

Also, there is precedent for people to extend luigi with projects like [sciluigi](https://github.com/pharmbio/sciluigi) to add special functionality they need. Which is useful, because there are several reasons where stock luigi wouldn't necessarily work:

* Luigi is mainly oriented around files and checks whether a specific file exists or not. There are some cases where we could translate this to checking if a specific database table exists or not, but other alterations like `UPDATE TABLE` would not work in this scheme. We might instead need to have an explicit table of task names and whether they need to be run or not.
* Similarly, luigi doesn't really have support for cascading rebuilds. You would have to go and delete all downstream files if you want to rebuild them. These cascading rebuilds would be common enough we would want them within the process.
* Database integration is possible, but tasks would reimplement similar boilerplate for their functionality.
* Being able to run some sort of validation before a task is marked complete is another thing that would be added to default Luigi.
* We might want to add other database specific things potentially like benchmarking execution times of queries (or affected rows) as well as even auto-failing if a query takes too long or the explain count is too large. 

Sciluigi seems like it's promising, but it's mainly useful for generalizing tasks to be used in multiple workflows. I don't really think we need such generalization in this use case, so it's better to just make our own fork of Luigi. Or so I think. 

## Types of Tasks

Broadly, there are several distinct types of tasks. These would all be just basic subclasses of a generic SQL execution task, but it's useful to have names if you want to reduce boilerplate code or see better the type of processes you are running:

* CREATE TABLE
* CREATE INDEX
* LOAD CSV INTO TABLE
* INSERT INTO TABLE
* DELETE FROM TABLE
* REPLACE TABLE WITH ANOTHER TABLE
* DUMP QUERY TO CSV
* CREATE STORED PROCEDURE
* OTHER SQL

Tasks could be parameterized (ie, run this query for a single year), and Luigi's TaskWrapper could be used for wrapping a complicated table creation as a bunch of smaller tasks (ie, insert records for a single year and run this for a bunch of years)

Any given task could succeed and upon success it should record the time it took to run and the number of rows affected. A task could fail for several reasons:

* Syntax error
* Runtime error
* Execution timeout
* Post-task validation failed
* Pre-execution constraint (ie, reject if EXPLAIN cost is above a certain value)

## Next Steps

Try this out with a simple example and using Postgres perhaps (with SqlAlchemy as wrapper). But this is still TK
