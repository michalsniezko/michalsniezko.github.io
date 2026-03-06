---
layout: default
title: Stress Testing Concurrency
parent: Testing, Concurrency, Distributed Locks
nav_order: 3
---

## Stress Testing Concurrency

**Scenario:** You wrote the upsert fix. Your unit test passes. But does it hold under real concurrency? A single-threaded PHPUnit test won't reproduce the race condition - you need multiple processes hitting the same row simultaneously.

**Methodology:** Spawn N parallel processes, each attempting the same upsert. Collect exit codes and stdout. If any process throws an exception or the final row count is wrong, the fix is incomplete.

### Bash Test Script

```bash
#!/bin/bash
set -euo pipefail

EXTERNAL_ID="stress-test-vehicle-$(date +%s)"
CONCURRENCY=20
ERRORS=0
PIDS=()

echo "Spawning $CONCURRENCY concurrent upserts for $EXTERNAL_ID..."

for i in $(seq 1 $CONCURRENCY); do
    php bin/console app:upsert-vehicle \
        --external-id="$EXTERNAL_ID" \
        --make="Make-$i" \
        --model="Model-$i" \
        > "/tmp/upsert_${i}.log" 2>&1 &
    PIDS+=($!)
done

# Wait for all processes and collect failures
for pid in "${PIDS[@]}"; do
    if ! wait "$pid"; then
        ERRORS=$((ERRORS + 1))
    fi
done

# Verify: exactly 1 row should exist
ROW_COUNT=$(psql -t -A -c \
    "SELECT COUNT(*) FROM vehicle WHERE external_id = '$EXTERNAL_ID'")

echo "Errors: $ERRORS / $CONCURRENCY"
echo "Row count: $ROW_COUNT (expected: 1)"

if [[ "$ERRORS" -gt 0 || "$ROW_COUNT" -ne 1 ]]; then
    echo "FAIL: Concurrency test failed"
    for i in $(seq 1 $CONCURRENCY); do
        if grep -qi "exception\|error" "/tmp/upsert_${i}.log"; then
            echo "--- Process $i ---"
            cat "/tmp/upsert_${i}.log"
        fi
    done
    exit 1
fi

echo "PASS: All $CONCURRENCY processes completed, exactly 1 row."
```

### Symfony Command for the Test

```php
#[AsCommand(name: 'app:upsert-vehicle')]
class UpsertVehicleCommand extends Command
{
    public function __construct(private VehicleRepository $repo)
    {
        parent::__construct();
    }

    protected function configure(): void
    {
        $this->addOption('external-id', null, InputOption::VALUE_REQUIRED);
        $this->addOption('make', null, InputOption::VALUE_REQUIRED);
        $this->addOption('model', null, InputOption::VALUE_REQUIRED);
    }

    protected function execute(InputInterface $input, OutputInterface $output): int
    {
        $this->repo->upsert(
            $input->getOption('external-id'),
            $input->getOption('make'),
            $input->getOption('model'),
        );

        $output->writeln('OK');
        return Command::SUCCESS;
    }
}
```

> **Safety First:** Run stress tests against a dedicated test database, not staging or production. Also, 20 concurrent processes on a local machine won't reproduce network-level race conditions (where latency between app server and DB amplifies the window). For realistic testing, run this from multiple containers targeting a shared database instance.
