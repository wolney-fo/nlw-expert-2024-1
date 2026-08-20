# VoteX

VoteX is a real-time polling and voting API built with Node.js, Fastify, PostgreSQL, and Redis. It delivers instant vote tally updates to connected clients via WebSockets while ensuring transactional integrity and vote uniqueness through signed session cookies and relational persistence.

## Table of Contents

- [VoteX](#votex)
  - [Table of Contents](#table-of-contents)
  - [About the Project](#about-the-project)
  - [Features](#features)
  - [Engineering](#engineering)
    - [Functional Requirements](#functional-requirements)
    - [Non-Functional Requirements](#non-functional-requirements)
    - [Scale Assumptions and Capacity Estimates](#scale-assumptions-and-capacity-estimates)
    - [Business Rules](#business-rules)
  - [Flowcharts](#flowcharts)
  - [Stack](#stack)
  - [Getting Started](#getting-started)
    - [Prerequisites](#prerequisites)
    - [Installation](#installation)
    - [Running Locally](#running-locally)
  - [Roadmap](#roadmap)
  - [License](#license)

## About the Project

Traditional voting systems often face a tradeoff between real-time responsiveness and data consistency. Aggregating vote counts directly in a relational database during high-concurrency voting events introduces severe lock contention and query latency. Conversely, relying purely on in-memory counters risks data loss and lacks relational auditability.

VoteX addresses this challenge through a deliberate hybrid architecture:

- **PostgreSQL** serves as the persistent source of truth, storing polls, options, and individual votes with a unique constraint on voter sessions to guarantee one vote per user per poll.
- **Redis Sorted Sets** maintain atomic, high-performance in-memory vote rankings and score counters, eliminating heavy `COUNT(*)` database aggregations.
- **WebSockets with Pub/Sub** broadcast instantaneous score updates to subscribed spectators whenever a vote is cast or changed.

## Features

- **Poll Creation**: Create custom polls with dynamically defined voting options.
- **Poll Retrieval with Live Scores**: Fetch poll details along with current vote counts aggregated directly from Redis
- **Session-Based Voting**: Cast votes anonymously using signed HTTP cookies, preventing duplicate votes without requiring a user registration flow
- **Vote Mutation Handling**: Automatically detect if a user changes their vote on an existing poll, updating the database record and adjusting Redis score tallies accordingly
- **Real-Time Live Results**: Stream real-time score updates to connected WebSocket clients as votes occur

## Engineering

### Functional Requirements

1. **Create a poll**: given a poll title and a list of option titles => persist the poll and its options in PostgreSQL and return the generated poll ID
2. **Retrieve a poll**: given a poll ID => return the poll metadata, options, and current score for each option fetched from Redis
3. **Vote on a poll**: given a poll ID, a target option ID, and a session identifier => record the vote in PostgreSQL, update the option score in Redis, and broadcast the update to active subscribers
4. **Stream poll results**: given a poll ID => establish a WebSocket connection and push real-time score changes to the connected client

Supporting requirements:

- The API must issue a signed `sessionId` cookie on the first vote if no session cookie exists on the incoming request
- If a client with an existing session votes for a different option in the same poll, the previous vote must be replaced, decrementing the old option score and incrementing the new option score in Redis
- If a client attempts to vote for the same option they already voted for, the system must reject the request with an error

### Non-Functional Requirements

- **Low-latency score updates**: Real-time score broadcasts must be dispatched immediately via in-memory pub/sub without polling the relational database
- **Transactional consistency**: Database writes and Redis score mutations must remain consistent; duplicate votes must be rejected at both application and database constraint levels
- **Stateless API server**: Session identification relies on cryptographic signed cookies, allowing the HTTP server to scale horizontally behind a load balancer without shared session state

### Scale Assumptions and Capacity Estimates

VoteX is designed to handle live, high-concurrency voting events (such as live streams, conferences, or televised polls) where thousands of users vote within short intervals while thousands of spectators watch live result updates

**Assumptions**

- A high-traffic live poll with 100,000 active voting participants
- Voting window concentrated over a 10-minute peak period
- 10,000 concurrent spectators connected to the WebSocket stream
- Average number of options per poll: 4 options
- Average vote record size in PostgreSQL: ~100 bytes

**Estimates**

- **Write throughput (peak voting)**: `100,000 votes / 600 seconds ≈` **167 writes/sec** average, with peak burst capacity designed for **1,000+ writes/sec**
- **Database storage per poll**: `100,000 * 100 bytes ≈` **10 MB** of relational data per 100k-vote poll
- **Redis memory footprint per poll**: A Sorted Set with 4 option UUIDs and scores occupies **less than 1 KB** in Redis memory
- **Database read queries avoided**: Without Redis, 10,000 spectators polling results every 2 seconds would produce `10,000 / 2 =` **5,000 aggregation queries/sec** (`SELECT COUNT(*) GROUP BY poll_option_id`). With Redis Sorted Sets and WebSocket Pub/Sub, relational read load during the entire event is **0 queries/sec**

This estimate justifies the separation between the relational storage layer (PostgreSQL) and the caching/real-time layer (Redis + WebSockets): PostgreSQL handles single-row atomic appends while Redis handles all tallying and real-time read throughput in memory

### Business Rules

- A voter is identified by a signed `sessionId` HTTP-only cookie valid for 30 days
- A user session can only hold at most
- Voting for a different option within the same poll automatically retracts the previous vote and applies the new vote
- Voting for the same option already chosen results in a `400 Bad Request` rejection
- A poll must have a title and at least one option to be created

## Flowcharts

The following sequence diagram illustrates the lifecycle of a vote and real-time event distribution:

```mermaid
sequenceDiagram
    autonumber
    actor User as Client / Voter
    actor Spectator as Spectator (WS)
    participant Fastify as Fastify Server
    participant Prisma as PostgreSQL (Prisma)
    participant Redis as Redis (Sorted Sets)
    participant PubSub as Voting Pub/Sub

    Spectator->>Fastify: WebSocket Connection (/polls/:pollId/results)
    Fastify->>PubSub: Subscribes socket to pollId channel

    User->>Fastify: POST /polls/:pollId/votes (with sessionId cookie)
    Fastify->>Prisma: Creates / Updates vote in votes table
    Fastify->>Redis: ZINCRBY (Updates option score in Redis)
    Fastify->>PubSub: Emits event with updated score ({ pollOptionId, votes })
    PubSub-->>Spectator: Real-time notification via WebSocket
```

## Stack

- Node.js
- Fastify
- Prisma ORM
- PostgreSQL
- Redis

## Getting Started

### Prerequisites

- **Node.js** (v20 or newer recommended)
- **npm** (v10 or newer)
- **Docker** and **Docker Compose** (for running PostgreSQL and Redis)

### Installation

1. Clone the repository:

   ```bash
   git clone https://github.com/wolney-fo/votex.git
   cd votex
   ```

2. Install dependencies:

   ```bash
   npm install
   ```

3. Configure environment variables:

   Create a `.env` file in the root directory:

   ```env
   DATABASE_URL="postgresql://docker:docker@localhost:5432/polls?schema=public"
   ```

### Running Locally

1. Start the PostgreSQL and Redis containers:

   ```bash
   docker compose up -d
   ```

2. Run database migrations:

   ```bash
   npx prisma migrate dev
   ```

3. Start the development server:

   ```bash
   npm run dev
   ```

The HTTP server will be available at `http://localhost:3333`.

## Roadmap

- [x] Poll creation and retrieval endpoints
- [x] Session-based vote tracking with signed cookies
- [x] Redis Sorted Sets for atomic vote tallying
- [x] WebSocket Pub/Sub for real-time score broadcasting
- [ ] Redis Pub/Sub integration for horizontal multi-instance scaling
- [ ] User authentication and poll ownership management
- [ ] Automated end-to-end and integration tests
- [ ] Web frontend dashboard for poll creation and live visualization

## License

MIT by [Wolney Oliveira](https://github.com/wolney-fo)
