# Post-Roast Updates

## Update 1: Per-user seat hold cap

**Triggered by:** Panel question on abuse control - "What stops one user from holding 200 seats?"

**What changed:**
Added a Redis counter key, `holds:{userId}:count`, with a max of 8 active holds per user and a hold TTL of 180 seconds during active sale traffic. The API increments the counter before creating a hold and decrements it on release, expiry, or booking confirmation.

**Why this is necessary:**
The original design protected the inventory, but it did not cap abusive parallel holds from a single account. Without a per-user ceiling, one user could monopolize hot inventory and turn the seat lock into a denial-of-service against the sale.

**What it costs:**
One more Redis read/write on the hold path, plus a small amount of operational logic to keep the counter consistent if a request times out after the counter increment but before the hold write completes.

**What it still doesn't solve:**
It limits one user, not a coordinated botnet or many accounts acting together. It also does not stop a user from cycling through seats over time, only from stacking too many simultaneous holds.

## Update 2: SQS publish circuit breaker with degraded checkout path

**Triggered by:** Panel question on budget and failure handling - "Your auto-scaling spikes your bill to $3,200 this month - what's your plan?"

**What changed:**
Added a publish circuit breaker around SQS. If publish failures exceed 60 seconds, the API stops waiting on the queue path and falls back to a synchronous payment attempt for low-volume traffic, while emitting a CloudWatch alarm on queue depth > 10,000 and publish error rate > 2 percent. The worker path still owns the normal async checkout flow.

**Why this is necessary:**
The original design assumed the queue would remain healthy. If SQS or the publish path degrades, the system needs an explicit degraded mode instead of letting checkout requests pile up, exhaust retries, and inflate infrastructure cost through retry storms.

**What it costs:**
More branching in the API, a small amount of latency variance during failover, and extra operational work to monitor the breaker and decide when to re-enable async checkout.

**What it still doesn't solve:**
It does not eliminate the cost of a real traffic spike. It only gives us a controlled fallback mode and better visibility when the queue becomes the bottleneck.
