# MongoDB Aggregation Notes (Progress)

## Aggregation Pipeline

-   `aggregate([])` returns all documents (similar result to `find({})`,
    but aggregation runs a pipeline).
-   Pipeline = Multiple stages where the output of one stage becomes the
    input of the next.

``` js
db.users.aggregate([
  { $match: { age: { $gte: 20 } } },
  { $project: { name: 1 } }
])
```

------------------------------------------------------------------------

## `$addFields`

Purpose: Add a temporary field (does **not** modify the original
collection).

``` js
{
  $addFields: {
    country: "Bangladesh"
  }
}
```

-   Can overwrite an existing field in the pipeline output.
-   Original documents remain unchanged.

------------------------------------------------------------------------

## `$project`

Purpose: - Include fields - Exclude fields - Rename fields - Create
calculated fields

``` js
{
  $project: {
    name: 1,
    age: 1
  }
}
```

Difference: - `$addFields` → keeps existing fields + adds new ones. -
`$project` → reshapes the document.

------------------------------------------------------------------------

## `$out`

-   Writes the pipeline result into a collection.
-   If the target collection exists, it is **replaced**.
-   Old documents are removed.

``` js
{
  $out: "adultUsers"
}
```

Memory: \> `$out` = Replace the whole target collection.

------------------------------------------------------------------------

## `$merge`

-   Writes results into a target collection.
-   Can insert/update/merge documents.

``` js
{
  $merge: {
    into: "adultUsers",
    on: "_id",
    whenMatched: "merge",
    whenNotMatched: "insert"
  }
}
```

Memory: \> `$merge` = Merge into existing collection.

------------------------------------------------------------------------

## `$group`

Purpose: - Create summary reports.

``` js
{
  $group: {
    _id: "$age",
    count: { $sum: 1 }
  }
}
```

### Common Accumulators

  Operator      Purpose
  ------------- --------------------------
  `$sum`        Total / Count
  `$avg`        Average
  `$min`        Minimum
  `$max`        Maximum
  `$push`       Store values in an array
  `$addToSet`   Unique values
  `$first`      First value
  `$last`       Last value

### `_id`

-   `_id: "$field"` → Group by field.
-   `_id: null` → Group the **entire collection** into one group.

Example:

``` js
db.users.aggregate([
  {
    $group: {
      _id: null,
      totalSalary: { $sum: "$salary" },
      averageSalary: { $avg: "$salary" },
      totalEmployees: { $sum: 1 }
    }
  }
])
```

------------------------------------------------------------------------

## `$$ROOT`

`$$ROOT` = Current document.

``` js
$push: "$$ROOT"
```

Stores the **entire document**.

``` js
$push: "$name"
```

Stores only the name.

Real-life: Group employees by department while keeping each employee's
full information.

------------------------------------------------------------------------

## `$unwind`

Purpose: - Convert one array into multiple documents.

``` js
{
  $unwind: "$products"
}
```

Example:

Before

``` js
{
  products: ["Laptop","Mouse","Keyboard"]
}
```

After

``` js
{ products:"Laptop" }
{ products:"Mouse" }
{ products:"Keyboard" }
```

### Real Use Case

To find how many times each product was sold:

``` js
db.orders.aggregate([
  { $unwind: "$products" },
  {
    $group:{
      _id:"$products",
      totalSold:{ $sum:1 }
    }
  }
])
```

Why? Without `$unwind`, `products` is one array. After `$unwind`, every
product becomes an individual document that can be grouped.

------------------------------------------------------------------------

## `$bucket`

Purpose: Group numeric values into ranges.

``` js
db.users.aggregate([
  {
    $bucket:{
      groupBy:"$age",
      boundaries:[20,40,60,80],
      default:"Other",
      output:{
        count:{ $sum:1 },
        names:{ $push:"$name" }
      }
    }
  }
])
```

Rules: - Lower boundary = Included - Upper boundary = Excluded

Example: - 20 \<= age \< 40 - 40 \<= age \< 60 - 60 \<= age \< 80 -
Others → `default`

Real-life: Age groups, Salary ranges, Price ranges, Marks ranges.

------------------------------------------------------------------------

## Thinking in Stages

Rule:

> One new task = One new stage.

Common stages:

-   `$match` → Filter
-   `$addFields` → Add field
-   `$project` → Reshape document
-   `$group` → Summary
-   `$unwind` → Flatten array
-   `$sort` → Sort
-   `$limit` → Limit
-   `$skip` → Skip
-   `$lookup` → Join
-   `$bucket` → Range grouping
-   `$merge` / `$out` → Save result


# `$lookup`

## Definition

`$lookup` দুটি collection-এর মধ্যে matching field ব্যবহার করে data **join** করে। এটি SQL-এর **LEFT JOIN**-এর মতো কাজ করে।

**সহজভাবে:** এক collection-এর document-এর সাথে অন্য collection-এর related document যোগ করার জন্য `$lookup` ব্যবহার করা হয়।

---

## Referencing Example

### Step 1: Create `users` Collection

```javascript
db.users.insertOne({
  _id: 1,
  name: "John"
});
```

### Step 2: Create `orders` Collection

```javascript
db.orders.insertMany([
  {
    product: "Laptop",
    userId: 1
  },
  {
    product: "Mouse",
    userId: 1
  },
  {
    product: "Keyboard",
    userId: 1
  }
]);
```

---

## Relationship

```text
users._id
    │
    ▼
orders.userId
```

এখানে `orders.userId` হলো `users._id`-এর **Reference**।

---

## `$lookup` Query

```javascript
db.users.aggregate([
  {
    $lookup: {
      from: "orders",
      localField: "_id",
      foreignField: "userId",
      as: "orders"
    }
  }
]);
```

---

## Parameter Explanation

| Parameter | Description |
|-----------|-------------|
| `from` | যে collection-এর সাথে join করবে |
| `localField` | বর্তমান collection-এর matching field |
| `foreignField` | অন্য collection-এর matching field |
| `as` | Joined data যে field-এর নামে return হবে |

---

## Flow

```text
users Collection
      │
      │ localField (_id)
      ▼
orders Collection
      │
      │ foreignField (userId)
      ▼
$lookup
      │
      ▼
orders[] (Joined Result)
```

---

## Expected Output

```javascript
{
  _id: 1,
  name: "John",
  orders: [
    {
      product: "Laptop",
      userId: 1
    },
    {
      product: "Mouse",
      userId: 1
    },
    {
      product: "Keyboard",
      userId: 1
    }
  ]
}
```
# MongoDB `$facet` Operator

## Definition

The `$facet` stage allows you to run **multiple aggregation pipelines in parallel** on the **same set of input documents** and combines their results into a single document.

> **One collection scan → Multiple independent results**

It is commonly used for dashboards, reports, analytics, and search results where multiple summaries are needed from the same data.

---

# Syntax

```javascript
db.collection.aggregate([
  {
    $facet: {
      pipeline1: [
        // aggregation stages
      ],
      pipeline2: [
        // aggregation stages
      ],
      pipeline3: [
        // aggregation stages
      ]
    }
  }
])
```

- Each key inside `$facet` represents a separate aggregation pipeline.
- Every pipeline receives the same input documents.
- The result of each pipeline is returned as an array.

---

# Example Data

```javascript
db.users.insertMany([
  { name: "John", age: 22, gender: "Male", salary: 40000 },
  { name: "Alice", age: 25, gender: "Female", salary: 50000 },
  { name: "Bob", age: 35, gender: "Male", salary: 70000 },
  { name: "Sara", age: 42, gender: "Female", salary: 90000 },
  { name: "David", age: 31, gender: "Male", salary: 65000 },
  { name: "Emma", age: 28, gender: "Female", salary: 55000 }
]);
```

---

# Example 1: Multiple Reports

```javascript
db.users.aggregate([
  {
    $facet: {
      totalUsers: [
        {
          $count: "count"
        }
      ],

      maleUsers: [
        {
          $match: {
            gender: "Male"
          }
        },
        {
          $count: "count"
        }
      ],

      averageSalary: [
        {
          $group: {
            _id: null,
            averageSalary: {
              $avg: "$salary"
            }
          }
        }
      ]
    }
  }
]);
```

### Output

```javascript
[
  {
    totalUsers: [
      { count: 6 }
    ],
    maleUsers: [
      { count: 3 }
    ],
    averageSalary: [
      {
        _id: null,
        averageSalary: 61666.67
      }
    ]
  }
]
```

---


---

# How `$facet` Works

```
             Collection
                 │
                 ▼
            $facet Stage
       ┌─────────┼─────────┐
       ▼         ▼         ▼
 Pipeline A  Pipeline B  Pipeline C
       │         │         │
       ▼         ▼         ▼
 Result A   Result B   Result C
       └─────────┼─────────┘
                 ▼
       Single Output Document
```
- MongoDB reads the input documents once.
- Every pipeline processes the same input independently.
- The final result combines all pipeline outputs into a single document.

---

# Common Stages Used Inside `$facet`

- `$match`
- `$group`
- `$sort`
- `$project`
- `$limit`
- `$skip`
- `$lookup`
- `$unwind`
- `$bucket`
- `$count`

Each sub-pipeline behaves like a normal aggregation pipeline.

---

# When to Use `$facet`

Use `$facet` when you need multiple results from the same collection, such as:

- Dashboard statistics
- Analytics reports
- Search filters
- Pagination with total document count
- Multiple summaries in a single query

---

# `$group` vs `$facet`

| `$group` | `$facet` |
|----------|----------|
| Groups documents into categories. | Runs multiple aggregation pipelines simultaneously. |
| Produces one grouped result. | Produces multiple independent results. |
| Uses accumulators like `$sum`, `$avg`, `$min`, `$max`. | Can contain `$group`, `$match`, `$sort`, `$lookup`, etc. |
| One output structure. | One document containing multiple result arrays. |

---

# Advantages

- Reads the collection only once.
- Runs multiple aggregations in a single query.
- Reduces the need for multiple database requests.
- Ideal for dashboards and reporting.

---

# Interview Points

### What is `$facet`?

`$facet` is an aggregation stage that executes multiple independent aggregation pipelines on the same input documents and returns all results in a single document.

### Why use `$facet`?

- Improves performance by avoiding multiple collection scans.
- Returns multiple summaries in a single aggregation query.
- Simplifies dashboard and analytics queries.

### Important Notes

- Every pipeline receives the same input documents.
- Each pipeline returns an array.
- Pipelines do not share data with each other.
- The final output is a single document containing all pipeline results.

