# Spark Workshop 1 

Connect to Redshift and execute a very simple Exploratory Data Analysis.

##What we are doing today

- Connect to the Python notebook living in the Spark cluster.
- Query the Redshift database using PySpark from the notebook.
- See the execution of the PySpark shell on the Spark WebUI
- Understand how the data is copied to S3
- Replace the query for a query to S3 directly

### Connect to the Python notebook in the Spark cluster

When creating a new Spark cluster, using our projects, a new Python notebook is automatically installed. We will connect to the Staging cluster (which still allow us to connect to the production redshift)

To connect, visit [Notebook](http://sparkling_aerie-staging.aws:8192/tree)

PySpark automatically makes available the `SparkContext` and the `SQLContext`. Here we will be using the `SQLContext`, which is Spark's entry point into the world of `DataFrames` and `DataSets`. We will be focusing on `DataFrames` for today.

A `DataFrame` in Spark is simple a table-like data structure which can be processed distributed by Spark. It is basically a wrapper around an `RDD`with extra information for the "column" names (the schema).

So let's connect to Redshift now:

- Paste the following code on your notebook session:


```
import subprocess;
key = subprocess.check_output(['/home/hadoop/aws_credentials','access_key'])

import subprocess;
secret = subprocess.check_output(['/home/hadoop/aws_credentials','secret_key'])

import subprocess;
token = subprocess.check_output(['/home/hadoop/aws_credentials','token'])


from pyspark.sql import SQLContext
sql_context = SQLContext(sc)
df = sql_context.read \
    .format("com.databricks.spark.redshift") \
    .option("url", "jdbc:redshift://copy-the-url-from-redshift/snowplow?tcpKeepAlive=true&ssl=true&sslfactory=org.postgresql.ssl.NonValidatingFactory&user=xxxxxxx&password=xxxxx") \
    .option("dbtable", "atomic.events") \
    .option("tempdir", "s3n://sb-wg-aerie-staging/spark/your-name/temp_redshift_unloads") \
    .option("temporary_aws_access_key_id", key) \
    .option("temporary_aws_secret_access_key", secret) \
    .option("temporary_aws_session_token", token) \
    .load()

df.printSchema()
```

#####What does that code do?

- That code simply allows to stablish a connection to Redshift from Spark.
- Creates a `DataFrame` mapped to the table `atomic.events`. 
- Specifies where in S3 the Data will get unloaded.

### Query the Redshift Database using the notebook

Ok, so now we are connected to redshift. Next we can use some `DataFrame` API operations to access the data and do analysis.
Let's do a very simple analysis. Let's check and Plot which are the most common type of events we have.

```
from datetime import datetimedate_object = datetime.strptime('May 1 2016', '%b %d %Y')grouped = df.filter(df['collector_tstamp'] > date_object).groupBy("event").count().cache()grouped.show()

```

Then we can plot the results using python libraries:

```
collected = grouped.collect()
events = map(lambda x: x.event, collected)
counts = map(lambda x: x.__getattr__("count"), collected)

%matplotlib inlineimport matplotlibmatplotlib.use('TkAgg')import pylabpylab.figure(1)x = range(5)pylab.xticks(x, events)pylab.plot(x,counts,"g")pylab.show()

```


### See the execution of the PySpark shell on the Spark WebUI

We can go to the url of the Spark Master [here](http://ip-10-20-1-134.eu-west-1.compute.internal:8088/cluster) to see the PySpark shell running.

If we click on the Application Master link for the PySpark shell application, we arrive at the Application's Spark execution


### Understand how the data is copied to S3

As mentioned, everytime we operate Spark with Redshift, internally an `unload` command is triggered which copies the Data to S3. We can see the content of the copied data in S3 in the provided path. In my case it is s3n://sb-wg-aerie-staging/spark/cscarioni/temp_redshift_unload


