# AI Prompt Log - Situation 1

## Overview
- **Tool:** Gemini Web - Model 3.5 Flash Lite
- **Objective:** Ask about the problems after think for the solutions.

## (1): Ask about handle inconsistency of Redis and Kafka

**Goal:** Need to find the way to solve inconsistency problem.

**Prompt**
```
I am designing the system that handles the users to get a limited discount. 

I have the system that splits into two parts. 

The first part is the Redis handles the discount count. Then, return the result back to the server and the server will send the message to Kafka if the user can get the discount. 

The second part is the server that subscribe the Kafka topic and writing to PostgreSQL.

How can I handle if there is a crash on the server so the result from Redis does not send to Kafka.
```

## (2): Confirm the understanding of replacing Kafka with Redis.

**Goal:** Need to ensure that, the Redis Stream solution can handle the given 50,000 users per 5 minutes load and can replace Kafka to reduce complexity.

**Prompt**
```
I would like to go with approach two, but if the Redis can act as stream, do I have to use Kafka for the situations of 50,000 users can make a request within 5 minutes?
```