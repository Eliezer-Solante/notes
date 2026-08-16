## What is the CAP Theorem?

The CAP theorem, created by computer scientist Eric Brewer, is a concept that says a distributed system (like a database spread across multiple computers) can only have **two out of three** desirable properties at the same time. These properties are:

1. **Consistency**: Everyone sees the same data at the same time. For example, if you update a bank balance, everyone should see the new amount right away.
2. **Availability**: The system is always up and running, so you can access it even if some parts fail.
3. **Partition Tolerance**: The system keeps working even if the network between computers breaks or slows down (called a partition).

The theorem states you can’t have all three together — you have to pick two based on your needs.

## Why Does This Happen?

In a distributed system, computers talk to each other over a network. If the network fails (partition), the system has to decide:

- Should it wait for all parts to agree (consistency), which might make it unavailable?
- Or should it keep running (availability) but let data differ temporarily?

This trade-off is what the CAP theorem is all about.

## The Three Combinations

1. **CA (Consistency + Availability)**:

- The system ensures everyone sees the same data and stays available.
- **Downside**: It fails if the network partitions (breaks). This is rare in real-world distributed systems.
- **Example**: Traditional single-server databases like MySQL (without replication).

**2. CP (Consistency + Partition Tolerance)**:

- The system keeps data consistent and works even if the network splits.
- **Downside**: It might become unavailable if parts can’t communicate.
- **Example**: Systems like MongoDB or Cassandra can be configured this way.

**3. AP (Availability + Partition Tolerance)**:

- The system stays up and running even if the network fails.
- **Downside**: Data might not be consistent right away (e.g., one user sees an old balance).
- **Example**: Systems like Amazon DynamoDB or Apache Cassandra in some modes.

## Real-World Example

Imagine an online store with servers in two cities. If the network between them fails:

- **CP Choice**: The store waits until both servers agree on stock levels (consistent), but it might go offline for users.
- **AP Choice**: The store keeps running, letting users shop, but stock might show differently on each server until the network is fixed.

## Why It Matters

The CAP theorem helps designers pick the right system for their needs. For example:

- Banks need **CP** for consistent money data.
- Social media needs **AP** to stay online even if data lags a bit.