---
layout: default
title: Generator Patterns with `yield from`
parent: Backend Patterns & Optimization for High-Volume Systems
nav_order: 1
---

## Generator Patterns with `yield from`

**Problem:** Loading 500k rows into an array to transform and filter them will eat gigabytes of memory. `array_map` and `array_filter` both build full intermediate arrays in memory - you pay for the entire dataset even if you only need the first 100 matching rows.

**Solution:** Generators produce one item at a time. Memory usage stays flat regardless of dataset size. `yield from` lets you compose generators by delegating to sub-generators without flattening into arrays.

```php
function readCsvRows(string $filePath): Generator
{
    $handle = fopen($filePath, 'r');
    fgetcsv($handle); // skip header

    while (($row = fgetcsv($handle)) !== false) {
        yield [
            'email'  => $row[0],
            'name'   => $row[1],
            'amount' => (float) $row[2],
        ];
    }

    fclose($handle);
}

function filterHighValue(Generator $rows, float $threshold): Generator
{
    foreach ($rows as $row) {
        if ($row['amount'] >= $threshold) {
            yield $row;
        }
    }
}

function mergeDataSources(array $filePaths): Generator
{
    foreach ($filePaths as $path) {
        yield from readCsvRows($path);
    }
}

// Process 3 files with 500k rows each - memory stays under 2MB
$allRows = mergeDataSources([
    '/data/transactions_jan.csv',
    '/data/transactions_feb.csv',
    '/data/transactions_mar.csv',
]);

$highValue = filterHighValue($allRows, 1000.00);

foreach ($highValue as $row) {
    $exporter->push($row); // stream out one row at a time
}
```

### Why `yield from` Instead of Nested Loops

```php
// Without yield from - you must manually iterate and re-yield
function merged(array $sources): Generator
{
    foreach ($sources as $source) {
        foreach (readCsvRows($source) as $row) {
            yield $row; // boilerplate
        }
    }
}

// With yield from - delegation is clean
function merged(array $sources): Generator
{
    foreach ($sources as $source) {
        yield from readCsvRows($source);
    }
}
```

`yield from` also propagates the return value of the sub-generator (via `Generator::getReturn()`), which is useful for collecting summaries like row counts.

> **Performance Tip:** A generator processing 1M rows uses ~2MB of memory. The same pipeline with `array_map` + `array_filter` allocates ~200MB+ for intermediate arrays. The trade-off: you lose random access (`$rows[500]`) and can only iterate once. If you need multiple passes, either re-create the generator or `iterator_to_array()` a small, filtered subset.

```mermaid
    %%{init: {'theme':'neutral'}}%%
sequenceDiagram
    participant App as Main Loop (foreach)
    participant Merger as mergeDataSources()<br/>(The Orchestrator)
    participant Reader as readCsvRows()<br/>(The Producer)
    participant Filter as filterHighValue()<br/>(The Processor)
    participant File as CSV File on Disk

    Note over App,File: START: Memory usage ~2MB
    
    App->>Merger: Request next row (foreach)
    activate Merger
    Note right of Merger: yield from delegates to<br/>readCsvRows('jan.csv')
    Merger->>Reader: Request first row
    activate Reader
    
    Reader->>File: Read one line
    File-->>Reader: Line: [email, name, 1500.00]
    Note right of Reader: Yields: ['amount' => 1500.00]
    Reader-->>Merger: (Delegated via yield from)
    deactivate Reader
    
    Merger-->>Filter: Passes row to filter
    activate Filter
    Note right of Filter: Checks: 1500.00 >= 1000.00?<br/>YES
    Note right of Filter: Yields: ['amount' => 1500.00]
    Filter-->>App: (Passes to final foreach)
    deactivate Filter
    
    App->>Exporter: push($row)
    Note over App,File: Row Processed. Memory usage still ~2MB

    %% -- Row 2 (Filtered Out) --
    
    App->>Merger: Request next row
    activate Merger
    Merger->>Reader: Request next row
    activate Reader
    Reader->>File: Read one line
    File-->>Reader: Line: [email, name, 500.00]
    Note right of Reader: Yields: ['amount' => 500.00]
    Reader-->>Merger: (Delegated)
    deactivate Reader
    
    Merger-->>Filter: Passes row to filter
    activate Filter
    Note right of Filter: Checks: 500.00 >= 1000.00?<br/>NO. No yield happens.
    Note over Filter: Filter continues its internal foreach<br/>to find next match.
    deactivate Filter

    Note over App,File: The pipeline "pauses" until Filter yields<br/>or Reader reaches EOF.
```
