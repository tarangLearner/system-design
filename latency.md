Here is a clear breakdown of these four fundamental performance and reliability metrics in system design:



1. Latency

Latency is the time it takes for a single request to travel from its source to its destination and return a response (often measured as Round Trip Time (RTT) or response time).

Unit of Measurement: Milliseconds (ms) or microseconds ($\mu$s).
Analogy: Think of latency as the time it takes for a single car to travel from Point A to Point B on a highway.
Key Considerations:

Percentiles matter: In system design, average latency can be misleading. Engineers look at p95 or p99 latency (the maximum response time experienced by 95% or 99% of requests) to account for slow tail latencies.
Common causes of high latency: Network distance (geography), disk I/O, heavy computation, or database locks.



2. Throughput

Throughput is the rate at which a system processes requests or processes data over a given unit of time.

Unit of Measurement:

Requests Per Second (RPS) or Queries Per Second (QPS) for web applications.
Operations Per Second (IOPS) for storage/databases.
Bits/Bytes Per Second (bps / Bps) for data pipelines.
Analogy: Continuing the highway analogy, throughput is the total number of cars passing through a toll booth per minute.
Key Difference from Latency: A system can have high throughput even with high latency by handling many requests concurrently/in parallel.



3. Bandwidth

Bandwidth is the maximum theoretical capacity of a communication channel or network link to transmit data over a unit of time.

Unit of Measurement: Megabits per second (Mbps) or Gigabits per second (Gbps).
Analogy: Bandwidth is the width of the highway (the number of lanes available). A 10-lane highway has higher bandwidth than a 2-lane highway.
Relationship with Latency & Throughput:

Bandwidth vs. Throughput: Bandwidth is the maximum potential capacity, whereas throughput is the actual rate of data being processed right now (throughput is bounded by bandwidth).
Bandwidth-Delay Product: Network performance is governed by both bandwidth (capacity) and latency (delay).



4. Availability

Availability measures the percentage of time a system remains operational, functional, and accessible to process requests when needed.

Unit of Measurement: Percentage (e.g., 99.9%, 99.99%), often described as "Nines" of availability.
The "Nines" Breakdown:

99% ("Two Nines"): ~3.65 days of downtime per year.
99.9% ("Three Nines"): ~8.76 hours of downtime per year.
99.99% ("Four Nines"): ~52.6 minutes of downtime per year.
99.999% ("Five Nines"): ~5.26 minutes of downtime per year.
Key Metrics:

MTBF (Mean Time Between Failures): Average operational time before a failure occurs.
MTTR (Mean Time to Repair): Average time required to recover from a failure.
$\text{Availability} = \frac{\text{MTBF}}{\text{MTBF} + \text{MTTR}}$



Summary Comparison



Metric

Focus

Key Question Answered

Latency

Speed / Delay

"How long does one request take?"

Throughput

Rate / Volume

"How many requests can we handle per second?"

Bandwidth

Capacity

"What is the maximum data pipeline width available?"

Availability

Reliability

"Is the system up and working right now?"

https://bytebytego.com/guides/top-5-strategies-to-reduce-latency/