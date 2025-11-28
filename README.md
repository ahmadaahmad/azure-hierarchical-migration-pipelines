The purpose of this project is to compare the performance between running ADF ForEach loops (data partitioned on day level in datalake) to synapse, and the performance of running COPY INTO script in Synapse dedicated SQL stored procedure. 

1. run the following script or similar to generate random data for a table with 1 billion rows, the script will save data in Parquet format, automatically create yyyy/MM/dd folder structure
```python
## PySpark Data Generator (1 Billion Rows)
from pyspark.sql.functions import rand, expr, to_date, year, month, dayofmonth
from pyspark.sql.types import IntegerType, DoubleType
from pyspark.sql import SparkSession

spark = SparkSession.builder.getOrCreate()

num_rows = 1_000_000_000
start_date = "2020-01-01"
days_range = (
    spark.sql("select datediff(current_date(), date('2020-01-01')) as diff").collect()[0]["diff"]
) - 1

df = (
    spark.range(0, num_rows)
    .withColumnRenamed("id", "CustomerID")
    .withColumn("Age", (rand() * 62 + 18).cast(IntegerType()))
    .withColumn("Salary", (rand() * 90000 + 30000).cast(IntegerType()))
    .withColumn("Balance", (rand() * 4009000 + 1000).cast(DoubleType()))
    .withColumn(
        "MaritalStatus",
        expr("CASE WHEN rand() < 0.33 THEN 'Single' WHEN rand() < 0.66 THEN 'Married' ELSE 'Divorced' END"),
    )
    .withColumn(
        "GeoLocation",
        expr("CASE WHEN rand() < 0.25 THEN 'North' WHEN rand() < 0.5 THEN 'South' WHEN rand() < 0.75 THEN 'East' ELSE 'West' END"),
    )
    .withColumn(
        "CreatedOn",
        to_date(expr(f"date_add(date('{start_date}'), CAST(floor(rand() * {days_range}) AS INT))")),
    )
    .withColumn("year", year("CreatedOn"))
    .withColumn("month", month("CreatedOn"))
    .withColumn("day", dayofmonth("CreatedOn"))
)

df.write.mode("overwrite").partitionBy("year", "month", "day").parquet(
    "abfss://<storagename>@<container>.dfs.core.windows.net/synthetic_customers_partitioned_yyyymmdd/"
)

```


2. Integrate this project github to ADF. create new table in synapse dedicated database that match the schema of the source file.
3. create stored procedure with COPY INTO command that points to synthetic_customers_partitioned_yyyymmdd/
4. compare the performance between ADF copy and COPY INTO usp
