# Match Recognize Relation

`MatchRecognizeRel` represents SQL row pattern recognition over ordered
partitions, corresponding to the core shape of `MATCH_RECOGNIZE`.

The initial surface is intentionally small. It establishes a first-class
relation, row-pattern-scoped field references, one-row output,
after-match-skip modes, metadata measure expressions, and deterministic direct
output rules. More advanced row-pattern feature families can be added in
follow-up schema changes.

## Match Recognize Operation

| Signature | Value |
| --- | --- |
| Inputs | 1 |
| Outputs | 1 |
| Property Maintenance | Distribution and orderedness are not preserved unless a consumer can prove otherwise. |
| Direct Output Order | Determined by the one-row output mode, partition fields, and measures as described below. |

### Match Recognize Properties

| Property | Description | Required |
| --- | --- | --- |
| Input | The relational input. | Required |
| Partition Expressions | Input-rooted field references that define independent row-pattern partitions. | Optional, defaults to a single partition |
| Sort Fields | Input-rooted field references with explicit direction and null ordering. | Required for portable row-pattern semantics |
| Measures | Named expressions emitted by the relation. | Optional |
| Rows Per Match | Explicit one-row output mode. | Required |
| After Match Skip | Explicit resume position after a non-empty match. | Required |
| Pattern | Row pattern AST to evaluate within each ordered partition. | Required |
| Definitions | Predicates keyed by primary pattern variable. Missing definitions are interpreted as `TRUE`. | Optional |

`RelCommon.emit` is not supported for `MatchRecognizeRel` in the initial
surface. Use a parent relation to drop, reorder, or reshape fields. `RelRoot`
names may name the final fields but do not change the direct output order.

=== "MatchRecognizeRel Message"

    ```proto
%%% proto.algebra.MatchRecognizeRel %%%
    ```

### Rows Per Match

The initial surface supports explicit `ONE ROW PER MATCH`. Producers should set
`rows_per_match.one_row` instead of relying on omitted defaults.

All `ALL ROWS PER MATCH` modes are deferred, including `SHOW EMPTY MATCHES`,
`OMIT EMPTY MATCHES`, and `WITH UNMATCHED ROWS`.

### After Match Skip

The initial surface supports:

- `AFTER MATCH SKIP PAST LAST ROW`
- `AFTER MATCH SKIP TO NEXT ROW`

After an empty match, matching resumes at the next input row to guarantee
progress. The initial surface defines empty matches only when they are
associated with a current input row being attempted; boundary-only zero-width
matches that have no current input row are deferred.

Variable skip targets such as `TO FIRST A` and `TO LAST A` are deferred.

### Row Pattern AST

The row pattern is represented as a tree. The initial surface supports primary
variables, concatenation, alternation, grouping, boundary anchors, and greedy
quantifiers.

=== "RowPattern Message"

    ```proto
%%% proto.algebra.RowPattern %%%
    ```

Boundary anchors are valid only at pattern boundaries. Quantifiers are greedy;
reluctant and possessive quantifiers are deferred.

### Definitions

Each `PatternDefinition` binds a boolean predicate to one primary pattern
variable. Definitions may use ordinary input-rooted field references and direct
row-pattern-rooted field references. A row-pattern-rooted reference uses
`FieldReference.RowPatternVariableReference` to make the pattern variable
explicit; ordinary table qualifiers are not pattern variables.

In the initial surface, direct row-pattern-rooted field references are valid
only when the referenced pattern variable denotes at most one row in the
expression context. In a `DEFINE` predicate this includes the variable currently
being defined. Multi-row variable references require follow-up navigation or
row-pattern aggregate support. If the variable denotes zero rows for an empty or
optional portion of a match, the field reference evaluates to null.

Missing definitions for primary variables are interpreted as `TRUE`.

### Measures

Each `RowPatternMeasure` has a field name and an expression. Measures may use
ordinary scalar `Expression` composition, direct row-pattern-rooted field
references for variables that denote at most one row, and the initial row-pattern
metadata expressions `CLASSIFIER()` and `MATCH_NUMBER()`.

Metadata expressions are measure expressions in the initial surface. Metadata in
`DEFINE`, classifier arguments, `MATCH_SEQUENCE_NUMBER()`, and `EXCLUDED()` are
not part of this native surface.

For one-row measures, row-pattern metadata and direct single-row variable
references are evaluated at the final row of the retained match. For an empty
match, direct row-pattern field references evaluate to null, `CLASSIFIER()`
evaluates to null, and `MATCH_NUMBER()` evaluates to the current match ordinal.
Follow-up features are needed for explicit `RUNNING` / `FINAL`, navigation, and
aggregate scope.

## Direct Output

Direct output is derived, not stored in a native `output_schema` field.

For `ONE ROW PER MATCH`, direct output fields are:

1. partition fields, in relation order
2. measure fields, in relation order

For an empty match, partition fields are taken from the input row at which the
empty match occurred.

Measure field names come from `RowPatternMeasure.name`. Measure field types and
nullability are derived from `RowPatternMeasure.expression`.
Expressions that can evaluate to null because of an empty or optional
row-pattern binding widen the corresponding measure field to nullable.

## Validation

Some validity rules cannot be expressed by protobuf parsing alone. Consumers
should reject parsed plans that violate the initial surface, including:

- `RelCommon.emit` on `MatchRecognizeRel`
- missing `rows_per_match.one_row`
- missing `after_match_skip`
- partition or order expressions that are not input-rooted field references
- row-pattern-rooted field references in partition or order expressions
- row-pattern metadata expressions outside measures
- row-pattern field references outside `DEFINE` and `MEASURES`
- row-pattern field references in measures without an at-most-one-row context
- non-nullable output for expressions that can produce null because of an
  empty or optional row-pattern binding
- non-nullable `CLASSIFIER()` output when the enclosing pattern can emit empty
  matches
- concatenation or alternation lists with fewer than two patterns
- anchors outside valid row-pattern boundaries
- patterns with no primary pattern variables, such as anchor-only zero-width
  patterns
- empty matches that are not associated with a current input row
- unresolved or duplicate `DEFINE` variables

## Initial Exclusions

The first native surface does not include:

- variable skip targets
- all `ALL ROWS PER MATCH` modes
- anchor-only zero-width patterns
- boundary-only empty matches with no current input row
- `PERMUTE`
- row-pattern exclusion
- subsets and union variables
- navigation functions such as `FIRST`, `LAST`, `PREV`, or `NEXT`
- row-pattern aggregates
- explicit `RUNNING` or `FINAL`
- classifier arguments
- `MATCH_SEQUENCE_NUMBER()` or `EXCLUDED()`
- native `output_schema`
- extension payload fields such as `schema_version` or `required_features`

## Example

```protobuf
--8<-- "examples/proto-textformat/match_recognize/one_row_concat_repeat.textproto"
```
