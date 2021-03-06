# HiveTableScans Execution Planning Strategy

`HiveTableScans` is an ../spark-sql-SparkStrategy.md[execution planning strategy] (of HiveSessionStateBuilder.md#planner[Hive-specific SparkPlanner]) that <<apply, replaces HiveTableRelation logical operators with HiveTableScanExec physical operators>>.

=== [[apply]] Planning Logical Plan for Execution -- `apply` Method

[source, scala]
----
apply(
  plan: LogicalPlan): Seq[SparkPlan]
----

NOTE: `apply` is part of ../catalyst/GenericStrategy.md#apply[GenericStrategy Contract] to plan a logical query plan for execution (i.e. generate a collection of ../SparkPlan.md[SparkPlans] for a given ../spark-sql-LogicalPlan.md[logical plan]).

`apply` converts (_destructures_) the input ../spark-sql-LogicalPlan.md[logical query plan] into projection expressions, predicate expressions, and a HiveTableRelation.md[HiveTableRelation].

`apply` requests the `HiveTableRelation` for the HiveTableRelation.md#partitionCols[partition columns] and filters the predicates to find so-called pruning predicates (that are expressions with no references and among the partition columns).

In the end, `apply` creates a "partial" HiveTableScanExec.md[HiveTableScanExec] physical operator (with the `HiveTableRelation` and the pruning predicates only) and ../spark-sql-SparkPlanner.md#pruneFilterProject[pruneFilterProject].
