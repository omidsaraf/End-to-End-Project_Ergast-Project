### Full Load

````python
from pyspark.sql.functions import col, explode, lit, monotonically_increasing_id, concat_ws

# Define the path to your JSON file
qualifying_path = '/mnt/dldatabricks/01-bronze/*/qualifying.json'

# Read the JSON file into a DataFrame
df = spark.read.json(qualifying_path, multiLine=True)

# Explode the nested Races array
races_df = df.select(explode(col("MRData.RaceTable.Races")).alias("race"))

# Explode the nested QualifyingResults array within each race
qualifying_df = races_df.select(
    col("race.Circuit.circuitId").alias("circuitId"),
    col("race.round").alias("round"),
    explode(col("race.QualifyingResults")).alias("qualifying")
)

# Extract required fields and generate unique IDs for qualifyId and raceId
qualifying_bronze = qualifying_df.select(
    monotonically_increasing_id().cast(IntegerType()).alias("qualify_Id"),  # Generate unique qualifyId
    col("qualifying.Constructor.constructorId").alias("constructor_Id"),
    col("qualifying.number").cast(IntegerType()).alias("number"),
    col("qualifying.position").cast(IntegerType()).alias("position"),
    col("qualifying.Q1").alias("q1"),
    col("qualifying.Q2").alias("q2"),
    col("qualifying.Q3").alias("q3"),
    concat_ws(" ", col("qualifying.Driver.givenName"), col("qualifying.Driver.familyName")).alias("DriverName"),  # Combine names as DriverName
    monotonically_increasing_id().cast(IntegerType()).alias("race_Id")  # Generate unique raceId
)
qualifying_bronze=qualifying_bronze.withColumn("ingestion_date", current_timestamp())

# Write the DataFrame in Delta format to the destination
qualifying_bronze.write.format("delta").mode("append").saveAsTable("F1_Silver.qualifying")

qualifying_silver=spark.read.format("delta").load("/mnt/dldatabricks/02-silver/F1_Silver/qualifying")
display(qualifying_silver)

````
![image](https://github.com/user-attachments/assets/1a6a4691-4636-444c-9486-fdf7fd6c35dd)

![image](https://github.com/user-attachments/assets/a35f4aae-5fe1-4cad-9003-e43e93c85480)

![image](https://github.com/user-attachments/assets/f06d8a13-7acb-4b1f-952a-3663acae3ccc)

### Incremantal Load
````python
from pyspark.sql import *
from pyspark.sql.functions import *
from pyspark.sql.types import *
from delta.tables import DeltaTable

# Define paths
qualifying_path = '/mnt/dldatabricks/01-bronze/*/qualifying.json'

# Read the JSON file into a DataFrame
df = spark.read.json(qualifying_path, multiLine=True)

# Explode the nested Races array
races_df = df.select(explode(col("MRData.RaceTable.Races")).alias("race"))

# Explode the nested QualifyingResults array within each race
qualifying_df = races_df.select(
    col("race.Circuit.circuitId").alias("circuitId"),
    col("race.round").alias("round"),
    explode(col("race.QualifyingResults")).alias("qualifying")
)

# Extract required fields and generate unique IDs for qualifyId
qualifying_bronze = qualifying_df.select(
    monotonically_increasing_id().cast(IntegerType()).alias("qualify_Id"),  # Generate unique qualifyId
    col("qualifying.Constructor.constructorId").alias("constructor_Id"),
    col("qualifying.number").cast(IntegerType()).alias("number"),
    col("qualifying.position").cast(IntegerType()).alias("position"),
    col("qualifying.Q1").alias("q1"),
    col("qualifying.Q2").alias("q2"),
    col("qualifying.Q3").alias("q3"),
    concat_ws(" ", col("qualifying.Driver.givenName"), col("qualifying.Driver.familyName")).alias("DriverName"),  # Combine names as DriverName
    monotonically_increasing_id().cast(IntegerType()).alias("race_Id")  # Generate unique raceId
)

# Deduplicate the source data
qualifying_bronze_dedup = qualifying_bronze.dropDuplicates(["qualify_Id"])

# Process the new DataFrame
qualifying_bronze_new_processed = qualifying_bronze_dedup \
    .withColumn("ingestion_date", current_timestamp()) \
    .select("qualify_Id", "constructor_Id", "number", "position", "q1", "q2", "q3", "DriverName", "race_Id", "ingestion_date")

# Load the existing Delta table
delta_table = DeltaTable.forPath(spark, "/mnt/dldatabricks/02-silver/F1_Silver/qualifying")

# Perform the merge (upsert) operation
delta_table.alias("existing") \
    .merge(
        qualifying_bronze_new_processed.alias("new"),
        "existing.qualify_Id = new.qualify_Id AND existing.q1 = new.q1 AND existing.q2 = new.q2"
    ) \
    .whenMatchedUpdateAll() \
    .whenNotMatchedInsertAll() \
    .execute()

# Read and display the merged data
merged_data = spark.read.format("delta").load("/mnt/dldatabricks/02-silver/F1_Silver/qualifying")
merged_data.display()

````

![image](https://github.com/user-attachments/assets/7ee0e263-f463-4d0a-a737-96fae5c363df)


