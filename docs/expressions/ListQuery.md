title: ListQuery

# ListQuery Subquery Expression

`ListQuery` is a spark-sql-Expression-SubqueryExpression.md[SubqueryExpression] that represents SQL's spark-sql-AstBuilder.md#withPredicate[IN predicate with a subquery], e.g. `NOT? IN '(' query ')'`.

[[Unevaluable]]
`ListQuery` expressions/Expression.md#Unevaluable[cannot be evaluated] and produce a value given an internal row.

[[resolved]]
`ListQuery` is spark-sql-Expression-SubqueryExpression.md#resolved[resolved] when:

. expressions/Expression.md#childrenResolved[Children are resolved]

. <<plan, Subquery logical plan>> is spark-sql-LogicalPlan.md#resolved[resolved]

. There is at least one <<childOutputs, child output attribute>>

=== [[creating-instance]] Creating ListQuery Instance

`ListQuery` takes the following when created:

* [[plan]] Subquery spark-sql-LogicalPlan.md[logical plan]
* [[children]] Child expressions/Expression.md[expressions]
* [[exprId]] Expression ID (as `ExprId` and defaults to a spark-sql-Expression-NamedExpression.md#newExprId[new ExprId])
* [[childOutputs]] Child output spark-sql-Expression-Attribute.md[attributes]
