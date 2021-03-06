title: LogicalPlanStats

# LogicalPlanStats -- Statistics Estimates and Query Hints of Logical Operator

`LogicalPlanStats` adds statistics support to spark-sql-LogicalPlan.md[logical operators] and is used for spark-sql-SparkPlanner.md[query planning] (with or without spark-sql-cost-based-optimization.md[cost-based optimization], e.g. spark-sql-Optimizer-CostBasedJoinReorder.md[CostBasedJoinReorder] or [JoinSelection](execution-planning-strategies/JoinSelection.md), respectively).

[[statsCache]]
With `LogicalPlanStats` every logical operator has <<stats, statistics>> that are computed only once when requested and are cached until <<invalidateStatsCache, invalidated>> and requested again.

Depending on spark-sql-cost-based-optimization.md[cost-based optimization] being enabled or not, <<stats, stats>> computes the spark-sql-Statistics.md[statistics] with FIXME or FIXME, respectively.

NOTE: Cost-based optimization is enabled when spark-sql-properties.md#spark.sql.cbo.enabled[spark.sql.cbo.enabled] configuration property is turned on, i.e. `true`, and is disabled by default.

Use `EXPLAIN COST` SQL command to explain a query with the <<stats, statistics>>.

[source, scala]
----
scala> sql("EXPLAIN COST SHOW TABLES").as[String].collect.foreach(println)
== Optimized Logical Plan ==
ShowTablesCommand false, Statistics(sizeInBytes=1.0 B, hints=none)

== Physical Plan ==
Execute ShowTablesCommand
   +- ShowTablesCommand false
----

You can also access the statistics of a logical plan directly using <<stats, stats>> method or indirectly requesting `QueryExecution` for spark-sql-QueryExecution.md#stringWithStats[text representation with statistics].

[source, scala]
----
val q = sql("SHOW TABLES")
scala> println(q.queryExecution.analyzed.stats)
Statistics(sizeInBytes=1.0 B, hints=none)

scala> println(q.queryExecution.stringWithStats)
== Optimized Logical Plan ==
ShowTablesCommand false, Statistics(sizeInBytes=1.0 B, hints=none)

== Physical Plan ==
Execute ShowTablesCommand
   +- ShowTablesCommand false
----

[source, scala]
----
val names = Seq((1, "one"), (2, "two")).toDF("id", "name")

// CBO is turned off by default
scala> println(spark.sessionState.conf.cboEnabled)
false

// CBO is disabled and so only sizeInBytes stat is available
// FIXME Why is analyzed required (not just logical)?
val namesStatsCboOff = names.queryExecution.analyzed.stats
scala> println(namesStatsCboOff)
Statistics(sizeInBytes=48.0 B, hints=none)

// Turn CBO on
import org.apache.spark.sql.internal.SQLConf
spark.sessionState.conf.setConf(SQLConf.CBO_ENABLED, true)

// Make sure that CBO is really enabled
scala> println(spark.sessionState.conf.cboEnabled)
true

// Invalidate the stats cache
names.queryExecution.analyzed.invalidateStatsCache

// Check out the statistics
val namesStatsCboOn = names.queryExecution.analyzed.stats
scala> println(namesStatsCboOn)
Statistics(sizeInBytes=48.0 B, hints=none)

// Despite CBO enabled, we can only get sizeInBytes stat
// That's because names is a LocalRelation under the covers
scala> println(names.queryExecution.optimizedPlan.numberedTreeString)
00 LocalRelation [id#5, name#6]

// LocalRelation triggers BasicStatsPlanVisitor to execute default case
// which is exactly as if we had CBO turned off

// Let's register names as a managed table
// That will change the rules of how stats are computed
import org.apache.spark.sql.SaveMode
names.write.mode(SaveMode.Overwrite).saveAsTable("names")

scala> spark.catalog.tableExists("names")
res5: Boolean = true

scala> spark.catalog.listTables.filter($"name" === "names").show
+-----+--------+-----------+---------+-----------+
| name|database|description|tableType|isTemporary|
+-----+--------+-----------+---------+-----------+
|names| default|       null|  MANAGED|      false|
+-----+--------+-----------+---------+-----------+

val namesTable = spark.table("names")

// names is a managed table now
// And Relation (not LocalRelation)
scala> println(namesTable.queryExecution.optimizedPlan.numberedTreeString)
00 Relation[id#32,name#33] parquet

// Check out the statistics
val namesStatsCboOn = namesTable.queryExecution.analyzed.stats
scala> println(namesStatsCboOn)
Statistics(sizeInBytes=1064.0 B, hints=none)

// Nothing has really changed, hasn't it?
// Well, sizeInBytes is bigger, but that's the only stat available
// row count stat requires ANALYZE TABLE with no NOSCAN option
sql("ANALYZE TABLE names COMPUTE STATISTICS")

// Invalidate the stats cache
namesTable.queryExecution.analyzed.invalidateStatsCache

// No change?! How so?
val namesStatsCboOn = namesTable.queryExecution.analyzed.stats
scala> println(namesStatsCboOn)
Statistics(sizeInBytes=1064.0 B, hints=none)

// Use optimized logical plan instead
val namesTableStats = spark.table("names").queryExecution.optimizedPlan.stats
scala> println(namesTableStats)
Statistics(sizeInBytes=64.0 B, rowCount=2, hints=none)
----

NOTE: The <<stats, statistics>> of a Dataset are unaffected by spark-sql-CacheManager.md#cacheQuery[caching] it.

NOTE: `LogicalPlanStats` is a Scala trait with `self: LogicalPlan` as part of its definition. It is a very useful feature of Scala that restricts the set of classes that the trait could be used with (as well as makes the target subtype known at compile time).

=== [[stats]] Computing (and Caching) Statistics and Query Hints -- `stats` Method

[source, scala]
----
stats: Statistics
----

`stats` gets the spark-sql-Statistics.md[statistics] from <<statsCache, statsCache>> if already computed. Otherwise, `stats` branches off per whether spark-sql-cost-based-optimization.md#spark.sql.cbo.enabled[cost-based optimization is enabled] or not.

[NOTE]
====
Cost-based optimization is enabled when spark-sql-properties.md#spark.sql.cbo.enabled[spark.sql.cbo.enabled] configuration property is turned on, i.e. `true`, and is disabled by default.

---

Use [SQLConf.cboEnabled](SQLConf.md#cboEnabled) to access the current value of `spark.sql.cbo.enabled` property.

[source, scala]
----
// CBO is disabled by default
val sqlConf = spark.sessionState.conf
scala> println(sqlConf.cboEnabled)
false
----
====

[[stats-cbo-disabled]]
With spark-sql-cost-based-optimization.md#spark.sql.cbo.enabled[cost-based optimization disabled] `stats` requests `SizeInBytesOnlyStatsPlanVisitor` to compute the statistics.

[[stats-cbo-enabled]]
With spark-sql-cost-based-optimization.md#spark.sql.cbo.enabled[cost-based optimization enabled] `stats` requests `BasicStatsPlanVisitor` to compute the statistics.

In the end, `statsCache` caches the statistics for later use.

`stats` is used when:

* `JoinSelection` execution planning strategy matches a logical plan:
   * [that is small enough for broadcast join](execution-planning-strategies/JoinSelection.md#canBroadcast) (using `BroadcastHashJoinExec` or `BroadcastNestedLoopJoinExec` physical operators)
   * [whose a single partition should be small enough to build a hash table](execution-planning-strategies/JoinSelection.md#canBuildLocalHashMap) (using `ShuffledHashJoinExec` physical operator)
   * [that is much smaller (3X) than the other plan](execution-planning-strategies/JoinSelection.md#muchSmaller) (for `ShuffledHashJoinExec` physical operator)
   * ...

* `QueryExecution` is requested for spark-sql-QueryExecution.md#stringWithStats[stringWithStats] for `EXPLAIN COST` SQL command

* `CacheManager` is requested to spark-sql-CacheManager.md#cacheQuery[cache a Dataset] or spark-sql-CacheManager.md#recacheByCondition[recacheByCondition]

* `HiveMetastoreCatalog` is requested for `convertToLogicalRelation`

* `StarSchemaDetection`

* `CostBasedJoinReorder` is spark-sql-Optimizer-CostBasedJoinReorder.md#apply[executed] (and does spark-sql-Optimizer-CostBasedJoinReorder.md#reorder[reordering])

=== [[invalidateStatsCache]] Invalidating Statistics Cache (of All Operators in Logical Plan) -- `invalidateStatsCache` Method

```scala
invalidateStatsCache(): Unit
```

`invalidateStatsCache` clears <<statsCache, statsCache>> of the current logical operators followed by requesting the [child logical operators](catalyst/TreeNode.md#children) for the same.
