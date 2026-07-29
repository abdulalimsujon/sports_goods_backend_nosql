# MongoDB Learning Notes (Conversation 1)

> These notes summarize the topics we covered so far.

## 1. Mongoose vs Mongosh

  Mongosh                        Mongoose
  ------------------------------ ---------------------------------
  MongoDB Shell                  Node.js ODM
  Used to run queries directly   Used inside Node/NestJS apps
  `find()` returns a cursor      `find()` returns a Query object
  No `.exec()`                   `.exec()` is available

**Why doesn't `.exec()` work in mongosh?**

``` js
db.users.find()
```

returns a **Cursor**, not a Mongoose Query.

------------------------------------------------------------------------

## 2. Comparison Operators

``` js
db.users.find({
  age: {
    $gt: 20,
    $lt: 30
  }
})
```

Equivalent to:

``` text
age > 20 AND age < 30
```

------------------------------------------------------------------------

## 3. `$and`

``` js
db.users.find({
  $and: [
    { age: { $gt: 20 } },
    { gender: "Male" }
  ]
})
```

All conditions must be true.

------------------------------------------------------------------------

## 4. `$or`

``` js
db.users.find({
  $or: [
    { age: 20 },
    { gender: "Female" }
  ]
})
```

Any one condition can be true.

------------------------------------------------------------------------

## 5. `$not`

``` js
db.users.find({
  age: {
    $not: { $gt: 30 }
  }
})
```

Meaning:

``` text
NOT(age > 30)
```

------------------------------------------------------------------------

## 6. `$nor`

``` js
db.users.find({
  $nor: [
    { gender: "Male" },
    { age: { $lt: 30 } }
  ]
})
```

Meaning

``` text
NOT(
 gender=="Male"
 OR
 age<30
)
```

Equivalent to

``` text
gender!="Male"
AND
age>=30
```

------------------------------------------------------------------------

## 7. `$exists`

``` js
db.users.find({
  phone:{
    $exists:true
  }
})
```

Field exists.

``` js
db.users.find({
  phone:{
    $exists:false
  }
})
```

Field does not exist.

------------------------------------------------------------------------

## 8. `$type`

``` js
db.users.find({
  age:{
    $type:"string"
  }
})
```

Checks the BSON type.

------------------------------------------------------------------------

## 9. `$size`

``` js
db.users.find({
  interests:{
    $size:2
  }
})
```

Matches arrays having exactly 2 elements.

------------------------------------------------------------------------

## 10. `$all`

``` js
db.users.find({
  interests:{
    $all:["Sports","Music"]
  }
})
```

Array must contain **all** specified values.

Difference:

-   `$in` = any value
-   `$all` = all values

------------------------------------------------------------------------

## 11. `$elemMatch`

For arrays.

``` js
db.scores.find({
 results:{
   $elemMatch:{
      $gte:80,
      $lt:85
   }
 }
})
```

One array element must satisfy both conditions.

Especially useful for arrays of objects.

------------------------------------------------------------------------

# Update Operators

## `$set`

``` js
db.users.updateOne(
 {name:"John"},
 {
   $set:{
      age:30
   }
 }
)
```

Updates existing field or creates a new field.

------------------------------------------------------------------------

## Positional `$`

``` js
db.users.updateOne(
{
 "education.degree":"BSc"
},
{
 $set:{
   "education.$.major":"CSE"
 }
})
```

`$` = first matched array element.

------------------------------------------------------------------------

## `$push`

``` js
$push:{
 interests:"Music"
}
```

Always adds the value.

Duplicates allowed.

------------------------------------------------------------------------

## `$each`

``` js
$push:{
 interests:{
   $each:[
     "A",
     "B"
   ]
 }
}
```

Adds multiple values.

Works with `$push` and `$addToSet`.

------------------------------------------------------------------------

## `$addToSet`

``` js
$addToSet:{
 interests:"Music"
}
```

Adds only if not already present.

With multiple values:

``` js
$addToSet:{
 interests:{
   $each:[
      "Reading",
      "Writing"
   ]
 }
}
```

------------------------------------------------------------------------

## `$pop`

``` js
$pop:{
 interests:1
}
```

Remove last element.

``` js
$pop:{
 interests:-1
}
```

Remove first element.

------------------------------------------------------------------------

## `$pull`

``` js
$pull:{
 interests:"Music"
}
```

Removes every matching value.

With condition:

``` js
$pull:{
 scores:{
   $lt:60
 }
}
```

------------------------------------------------------------------------

## `$pullAll`

``` js
$pullAll:{
 interests:[
   "Music",
   "Writing"
 ]
}
```

Removes all specified values.

Cannot use conditions.

------------------------------------------------------------------------

# Difference

  Operator      Duplicate               Multiple Values   Condition
  ------------- ----------------------- ----------------- -----------
  `$push`       ✅                      `$each`           ❌
  `$addToSet`   ❌                      `$each`           ❌
  `$pull`       Removes matches         N/A               ✅
  `$pullAll`    Removes listed values   ✅                ❌
  `$pop`        First/Last only         ❌                ❌

------------------------------------------------------------------------

More topics will be appended as we continue learning.
