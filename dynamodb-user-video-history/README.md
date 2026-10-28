# First NoSQL Database with Amazon DynamoDB

A hands-on AWS SimuLearn lab exploring how to create a DynamoDB table, store video-viewing history, work with flexible item attributes, and query records.

## Overview

The scenario uses viewer-history data to demonstrate basic NoSQL data modeling with Amazon DynamoDB.

The table stores information such as the user, video watched, viewing timestamp, preferred language, device type, and playback stop position.

> **Scope:** This lab focuses on DynamoDB table operations and data modeling. The Fire TV workflow shown in the scenario provides business context; it does not represent a complete streaming application or device integration implemented in this repository.

## Objectives

- Create a DynamoDB table.
- Define a composite primary key.
- Add and inspect items.
- Store different non-key attributes on different items.
- Query viewing-history records.
- Extend an item with an additional attribute.

## Environment

| Item | Details |
|---|---|
| Learning platform | AWS SimuLearn on AWS Skill Builder |
| Lab | First NoSQL Database |
| AWS service | Amazon DynamoDB |
| Table | `UserVideoHistory` |
| Partition key | `userId` |
| Sort key | `lastDateWatched` |
| Scenario | Store and retrieve user video-viewing history |


| Attribute | Role | Purpose |
|---|---|---|
| `userId` | Partition key | Identifies the user associated with the viewing record |
| `lastDateWatched` | Sort key | Distinguishes viewing records for the same user |
| `videoId` | Non-key attribute | Identifies the video |
| `preferredLanguage` | Non-key attribute | Stores a language preference when present |
| `supportedDeviceTypes` | Non-key attribute | Stores device information |
| `lastStopTime` | Non-key attribute | Stores the playback stop position when present |

Non-key attributes can vary between items. For example, a viewing record can contain `lastStopTime` while another omits it.

## Implementation

### 1. Create the table

Created `UserVideoHistory` with:

- `userId` as the partition key.
- `lastDateWatched` as the sort key.

This structure supports multiple viewing records for one user, provided each record has a distinct sort-key value within that user's partition.

### 2. Add viewing-history items

Added sample records containing user, video, timestamp, language and device information.

The sample data includes multiple records for one user and records belonging to another user.

### 3. Explore flexible attributes

Worked with items containing different sets of non-key attributes.

The scenario includes `lastStopTime` on selected records rather than requiring every item to contain that attribute.

This demonstrates the distinction between a defined primary-key structure and flexible non-key item attributes.

### 4. Query the table

Queried viewing-history records using the table's key structure.

A DynamoDB `Query` requires a partition-key value and can optionally include a condition on the sort key. This supports retrieving a user's viewing history and narrowing it by the viewing timestamp.

## DIY Extension

The lab also provides an additional exercise:

- Create a new item with a new `userId` value.
- Include the required `lastDateWatched` sort-key value.
- Add a Number attribute named `rating`.

This extends the item without adding `rating` to every existing record.

> This section describes the DIY objective. Completion should be documented separately if the extension was performed.

## Validation

| Check | What to verify |
|---|---|
| Table definition | `UserVideoHistory` uses `userId` and `lastDateWatched` as its primary-key attributes |
| Stored items | Sample viewing-history records are visible |
| Multiple records per user | The same `userId` appears with different sort-key values |
| Flexible attributes | Some items contain non-key attributes that others omit |
| Query | Returned records match the requested partition-key value and any supplied sort-key condition |
| DIY extension, if completed | The new item contains both key attributes and a Number attribute named `rating` |

## Key Takeaways

- Model the primary key around the intended access pattern.
- A composite primary key allows multiple records to share a partition-key value.
- The combination of partition key and sort key uniquely identifies an item.
- Non-key attributes do not need to be identical across all items.
- `Query` uses key conditions; it is not an unrestricted search across arbitrary attributes.
- A learning scenario is different from an implemented end-to-end application.
