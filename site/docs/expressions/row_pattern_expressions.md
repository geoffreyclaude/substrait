# Row Pattern Expressions

Row pattern expressions provide metadata values for `MEASURES` expressions of a
`MatchRecognizeRel`.

The initial surface includes only metadata measure expressions:

- nullary `CLASSIFIER()`
- nullary `MATCH_NUMBER()`

These expressions are scoped to row pattern recognition. They are not general
scalar functions and should not appear outside `MEASURES` expressions of a
`MatchRecognizeRel` in the initial surface.

## Row Pattern Expression Structure

| Component | Description | Protobuf Field | Required |
| --- | --- | --- | --- |
| Metadata Expression | The row-pattern metadata value to evaluate. | `expression_type` | Yes |
| Output Type | The declared type of the metadata value. | `output_type` | Yes |

=== "RowPatternExpression Message"

    ```proto
%%% proto.algebra.RowPatternExpression %%%
    ```

## CLASSIFIER

`RowPatternClassifier` represents nullary `CLASSIFIER()`. It returns the
primary pattern variable name associated with the final row of the retained
match in one-row mode. For an empty match, it returns null. Arguments such as
`CLASSIFIER(A)` are not part of the initial surface.

The declared `output_type` for `CLASSIFIER()` must be nullable when the
enclosing pattern can emit empty matches.

## MATCH_NUMBER

`RowPatternMatchNumber` represents nullary `MATCH_NUMBER()`. It returns the
ordinal match number within the current partition.

## Scope

The initial surface treats row-pattern metadata as measure-only. Metadata in
`DEFINE`, `PARTITION BY`, `ORDER BY`, and ordinary relations is deferred.

`output_type` is a semantic requirement even though protobuf parsing cannot
enforce presence for message fields.

Engine-specific metadata such as `MATCH_SEQUENCE_NUMBER()` and `EXCLUDED()` is
not part of this native expression surface.

## Example

```protobuf
--8<-- "examples/proto-textformat/row_pattern_expression/classifier.textproto"
```
