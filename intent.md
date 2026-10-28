# Intent

## Goal
Support Coordinator wants to preview which tickets a CSV file import will produce before the import approval. The preview never changes the original data and performing the approval is a different step. The preview returns either the tickets in input order, normalizing priorities or one error code explaining why the file is not acceptable.

## Scope
 allowed changes: only  src/import-tickets.mjs and tests/import-tickets.test.mjs.
 exclusions: changing or fixing file data; performing or approving the import; saving the results; saving the tickets in the csv, any approval or ingestion flow.
 non-goals: UI, database or storage, network, writing files, new dependencies
 anything not defined in the scope: ask first
 normalizations:
  - normalizations are performed before row validation
  - Title and Priority value are trimmied
  - empty or null priority is normalized to "normal" value
protected paths: src/tickets.mjs, .github/ and CLAUDE.md.
tests to write:
 - one test per acceptance case in the Acceptance section, plus any edge case the rules above define.


## Constraints

 validations:
  - header accepts two columns: title and priority. Priority is optional. Title is mandatory column and its values cannot be empty after trim.
  - no extra or duplicate columns are allowed.
  - priority must be one of the allowed values
  - header can be accept in any order. But no short rows are accepted: rows data must match the columns in number.
rejections reasons -> Error Codes
    - Missing title, extra/unknown/duplicate columns, malformed rows or CSV, empty input -> unknown-column
    - empty title value after trimming -> empty-title
    - More than 100 rows discarding header and including malformed rows checked before any row validation -> row-limit
    - priority that isn't low,normal, or urgent after trimming -> invalid-priority

## Plan
confirm scope first, then parse the CSV, check the headers, apply the row limit, trim and validate values, build the result, and write tests from the Acceptance cases.

## Acceptance
The preview is complete when every case below returns exactly the expected result, and each one
is covered by a test in `tests/import-tickets.test.mjs`. The cases cover the seven required
categories:

- Valid explicit priority: `Printer issue,urgent` keeps `urgent`.
- Default priority: a file with only a `title` column, or a blank priority cell, gets `normal`.
- Quoted title containing a comma: `"Printer jammed, tray 2"` stays one title.
- Empty title: a title that is only spaces is rejected with `empty-title`.
- Unknown column: an `owner` column is rejected with `unknown-column`.
- Invalid priority: `critical` is rejected with `invalid-priority`.
- Too many rows: 101 data rows are rejected with `row-limit` before any value is checked.

```json
{
  "schema": 1,
  "change": "ticket-csv-preview",
  "allowed_changes": ["src/import-tickets.mjs", "tests/import-tickets.test.mjs"],
  "protected_paths": ["src/tickets.mjs", ".github/", "CLAUDE.md"],
  "non_goals": ["ui", "database", "network"],
  "requirements": {
    "columns": ["title", "priority"],
    "default_priority": "normal",
    "valid_priorities": ["low", "normal", "urgent"],
    "reject_unknown_columns": true,
    "reject_empty_title": true,
    "max_rows": 100,
    "side_effects": "preview-only"
  },
  "verification": ["npm test", "git diff --check"],
  "approval": "ask-before-implementation",
  "acceptance_cases": [
    {
      "csv": "title,priority\nPrinter issue,urgent\n",
      "expected": {"tickets":[{"title":"Printer issue","priority":"urgent"}]}
    },
    {
      "csv": "title\nPassword reset\n",
      "expected": {"tickets":[{"title":"Password reset","priority":"normal"}]}
    },
    {
      "csv": "title,priority\nPassword reset,\n",
      "expected": {"tickets":[{"title":"Password reset","priority":"normal"}]}
    },
    {
      "csv": "title,priority\n\"Printer jammed, tray 2\",low\n",
      "expected": {"tickets":[{"title":"Printer jammed, tray 2","priority":"low"}]}
    },
    {
      "csv": "title,priority\n   ,urgent\n",
      "expected": {"error":"empty-title"}
    },
    {
      "csv": "title,owner\nPrinter issue,Sam\n",
      "expected": {"error":"unknown-column"}
    },
    {
      "csv": "title,priority\nPrinter issue,critical\n",
      "expected": {"error":"invalid-priority"}
    },
    {
      "csv": "title,priority\nPrinter issue,normal\n",
      "repeat": 101,
      "expected": {"error":"row-limit"}
    }
  ]
}
```

## Verification
run the tests and check if only the allowed files changed with git diff --check and ask before expanding or implementing the scope.
