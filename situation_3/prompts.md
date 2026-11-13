# AI Prompt Log - Situation 3

## Overview
- **Tool:** Gemini Web - Model 3.5 Flash Lite
- **Objective:** Ask about the problems after think for the solutions.

## (1): Ask about Redis streaming is enough for 100,000 requests / second

**Goal:** Need to know that, the redis can handle up to 100,000 requests / second or not.

**Prompt**
```
I'm designing the system that the user can request for the coupon that has a limited size of 100 and the requests can be 100,000 at the same time. Can Redis handle or decrease the coupon count and append to the stream so the later process can read the stream further to write to PostgreSQL?
```

## (2): Ask about Redis max streaming capacity per second

**Goal:** Need to know the maximum load the the Redis can handle.

**Prompt**
```
Then what is the maximum load that single Redis can handle and how to resolve it?
```

## (3): Ask about the concern of missing record, when decoupling XADD to server.

**Goal:** Confirm understanding of missing record if decopling XADD to server.

**Prompt**
```
About decouple the stream write, it can cause the missing record if the server is failed before calling xadd.
```

## (4): Reason of decoupling

**Goal:** Ask for the reason of why the AI suggest for decoupling to observe any concern to finalize decision.

**Prompt**
```
Why you suggest me to decoupling at the first place?
```
