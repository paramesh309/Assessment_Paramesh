from pyspark.sql import SparkSession
from pyspark.sql.functions import col, lit


spark = SparkSession.builder \
    .appName("Sales Data Transformation") \
    .getOrCreate()

# Define file paths
file_a = "dbfs:/FileStore/tables/GANGU/order_region_a_in_.csv"
file_b = "dbfs:/FileStore/tables/GANGU/order_region_b_in_.csv"


def read_csv_data(file_path, region):
    df = spark.read.csv(file_path, header=True, inferSchema=True)
    df = df.withColumn("region", lit(region))  # Add region info
    return df


df_a = read_csv_data(file_a, "A")
df_b = read_csv_data(file_b, "B")

#  Combine both DataFrames
df_combined = df_a.union(df_b)

#  Data Transformation Function
def transform_data(df):
    # Add total_sales column (QuantityOrdered * ItemPrice)
    df_transformed = df.withColumn("total_sales", col("QuantityOrdered") * col("ItemPrice"))
    
    # Add net_sale column (total_sales - PromotionDiscount)
    df_transformed = df_transformed.withColumn("net_sale", col("total_sales") - col("PromotionDiscount"))
    
    # Filter rows where net_sale <= 0
    df_filtered = df_transformed.filter(col("net_sale") > 0)
    
    # Remove duplicate OrderId values
    df_no_duplicates = df_filtered.dropDuplicates(["OrderId"])
    
    return df_no_duplicates

#  Apply transformation
df_transformed = transform_data(df_combined)

#  Show some transformed data
df_transformed.show(5)

#  Register the DataFrame as a temp view to use SQL queries
df_transformed.createOrReplaceTempView("sales_data")

#  Validation using SQL Queries

#  Total number of records
total_records = spark.sql("SELECT COUNT(*) FROM sales_data")
total_records.show()

#  Total sales amount by region
total_sales_by_region = spark.sql("SELECT region, SUM(total_sales) AS total_sales FROM sales_data GROUP BY region")
total_sales_by_region.show()

#  Average sales amount per transaction
avg_sales = spark.sql("SELECT AVG(total_sales) AS avg_sales FROM sales_data")
avg_sales.show()

#  Check for duplicate OrderId values
duplicates = spark.sql("SELECT OrderId, COUNT(*) FROM sales_data GROUP BY OrderId HAVING COUNT(*) > 1")
duplicates.show()
