title: HadoopFsRelation

# HadoopFsRelation -- Relation Of File-Based Data Sources

`HadoopFsRelation` is a <<spark-sql-BaseRelation.md#, BaseRelation>> and <<spark-sql-FileRelation.md#, FileRelation>>.

`HadoopFsRelation` is <<creating-instance, created>> when:

* `DataSource` is requested to <<spark-sql-DataSource.md#resolveRelation, resolve a relation>> for <<spark-sql-FileFormat.md#, file-based data sources>>

* `HiveMetastoreCatalog` is requested to hive/HiveMetastoreCatalog.md#convertToLogicalRelation[convert a HiveTableRelation to a LogicalRelation over a HadoopFsRelation] (for hive/RelationConversions.md[RelationConversions] logical post-hoc evaluation rule for `parquet` or `native` and `hive` ORC formats)

[source, scala]
----
// Demo the different cases when `HadoopFsRelation` is created

import org.apache.spark.sql.execution.datasources.{HadoopFsRelation, LogicalRelation}

// Example 1: spark.table for DataSource tables (provider != hive)
import org.apache.spark.sql.catalyst.TableIdentifier
val t1ID = TableIdentifier(tableName = "t1")
spark.sessionState.catalog.dropTable(name = t1ID, ignoreIfNotExists = true, purge = true)
spark.range(5).write.saveAsTable("t1")

val metadata = spark.sessionState.catalog.getTableMetadata(t1ID)
scala> println(metadata.provider.get)
parquet

assert(metadata.provider.get != "hive")

val q = spark.table("t1")
// Avoid dealing with UnresolvedRelations and SubqueryAliases
// Hence going stright for optimizedPlan
val plan1 = q.queryExecution.optimizedPlan

scala> println(plan1.numberedTreeString)
00 Relation[id#7L] parquet

val LogicalRelation(rel1, _, _, _) = plan1.asInstanceOf[LogicalRelation]
val hadoopFsRel = rel1.asInstanceOf[HadoopFsRelation]

// Example 2: spark.read with format as a `FileFormat`
val q = spark.read.text("README.md")
val plan2 = q.queryExecution.logical

scala> println(plan2.numberedTreeString)
00 Relation[value#2] text

val LogicalRelation(relation, _, _, _) = plan2.asInstanceOf[LogicalRelation]
val hadoopFsRel = relation.asInstanceOf[HadoopFsRelation]

// Example 3: Bucketing specified
val tableName = "bucketed_4_id"
spark
  .range(100000000)
  .write
  .bucketBy(4, "id")
  .sortBy("id")
  .mode("overwrite")
  .saveAsTable(tableName)

val q = spark.table(tableName)
// Avoid dealing with UnresolvedRelations and SubqueryAliases
// Hence going stright for optimizedPlan
val plan3 = q.queryExecution.optimizedPlan

scala> println(plan3.numberedTreeString)
00 Relation[id#52L] parquet

val LogicalRelation(rel3, _, _, _) = plan3.asInstanceOf[LogicalRelation]
val hadoopFsRel = rel3.asInstanceOf[HadoopFsRelation]
val bucketSpec = hadoopFsRel.bucketSpec.get

// Exercise 3: spark.table for Hive tables (provider == hive)
----

The optional <<bucketSpec, bucketing specification>> is defined exclusively for <<spark-sql-DataSource.md#, non-streaming file-based data sources>> and used for the following:

* <<spark-sql-SparkPlan-FileSourceScanExec.md#outputPartitioning, Output partitioning scheme>> and <<spark-sql-SparkPlan-FileSourceScanExec.md#outputOrdering, output data ordering>> of the corresponding <<spark-sql-SparkPlan-FileSourceScanExec.md#, FileSourceScanExec>> physical operator

* [DataSourceAnalysis](logical-analysis-rules/DataSourceAnalysis.md) post-hoc logical resolution rule (when executed on a <<InsertIntoTable.md#, InsertIntoTable>> logical operator over a <<spark-sql-LogicalPlan-LogicalRelation.md#, LogicalRelation>> with `HadoopFsRelation` relation)

=== [[creating-instance]] Creating HadoopFsRelation Instance

`HadoopFsRelation` takes the following to be created:

* [[location]] <<FileIndex.md#, FileIndex>> (for <<sizeInBytes, sizeInBytes>> and <<inputFiles, inputFiles>>)
* [[partitionSchema]] Partition spark-sql-StructType.md[schema]
* [[dataSchema]] Data spark-sql-StructType.md[schema]
* [[bucketSpec]] <<spark-sql-BucketSpec.md#, Bucketing specification>> (optional)
* [[fileFormat]] spark-sql-FileFormat.md[FileFormat]
* [[options]] Options
* [[sparkSession]] SparkSession.md[SparkSession]

`HadoopFsRelation` initializes the <<internal-properties, internal properties>>.

=== [[inputFiles]] Files to Scan -- `inputFiles` Method

[source, scala]
----
inputFiles: Array[String]
----

NOTE: `inputFiles` is part of the <<spark-sql-FileRelation.md#inputFiles, FileRelation Contract>> for the list of files to read for scanning this relation.

`inputFiles` simply requests the <<location, FileIndex>> for the <<FileIndex.md#inputFiles, inputFiles>>.

=== [[sizeInBytes]] `sizeInBytes` Method

[source, scala]
----
sizeInBytes: Long
----

NOTE: `sizeInBytes` is part of the <<spark-sql-BaseRelation.md#sizeInBytes, BaseRelation Contract>> for the estimated size of a relation (in bytes).

`sizeInBytes`...FIXME

=== [[toString]] Human-Friendly Textual Representation -- `toString` Method

[source, scala]
----
toString: String
----

NOTE: `toString` is part of the `java.lang.Object` contract for the string representation of the object.

`toString` is the following text based on the <<fileFormat, FileFormat>>:

* <<spark-sql-DataSourceRegister.md#shortName, shortName>> for <<spark-sql-DataSourceRegister.md#, DataSourceRegister>> data sources

* *HadoopFiles* otherwise

=== [[getColName]] `getColName` Internal Method

[source, scala]
----
getColName(
  f: StructField): String
----

`getColName`...FIXME

NOTE: `getColName` is used when...FIXME

=== [[internal-properties]] Internal Properties

[cols="30m,70",options="header",width="100%"]
|===
| Name
| Description

| overlappedPartCols
a| [[overlappedPartCols]]

[source, scala]
----
overlappedPartCols: Map[String, StructField]
----

Mutable...FIXME

Used when...FIXME

|===
