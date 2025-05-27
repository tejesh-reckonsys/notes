---
date: 2025-02-03 18:09:43+05:30
title: DB Optimizations
---

# Django ORM basics

ORM stands for "Object Relational Mapping," which means that it maps from tables in relational database to objects. This makes it easier to access the information from database, but at the same time makes it oblivious how it communicates with database internally. This leads to un-optimized database communications.

This section explains some common methods used in Django ORM, and how they communicate with database.

There are mainly 2 sections/categories of methods:
1. Read type methods
2. Write type methods
## 1. Read type methods

Most of the database calls happen to read the data from database. In Django, main thing to know about read type methods is ***when ORM makes the database query***.

To read some data from database, the most common methods are:

- `get()`
- `first()`
- `count()`
- `all()`
- `filter()` and `exclude()`
- `values()` and `values_list()`
- `order_by()`

These are mainly of 2 categories:

1. Immediately query database
2. Don't immediately query database

### Immediately query database
Out of these, `get()`, `first()` and `count()` makes a database query as soon as executed.

For example:

```py
box = Box.objects.get(id="...")  # Makes a database query and stores in box
print(box)  # Displays already stored data
# Prints box

box_count = Box.objects.count()  # Makes a database query
print(box_count)  # Displays already stored data
# Prints box count
```

### Don't immediately query database

The other methods, `all()`, `filter()`, `exclude()`, `values()`, `values_list()` and `orderby()`, don't immediately makes a database query.
These methods talks to database only when evaluating the *queryset*.

Evaluating the queryset happens in following cases:

- **Iteration:** Looping through queryset using `for x in queryset:`
- **Displaying:** Printing the queryset, using `repr()`, etc.
- **`len()`:** Using `len()` to find the length
- **Converting to a list:** Using `list(queryset)`
- **Using as condition:** Such as `if queryset:`

> [!important] 
> Once evaluated, the Django stores the result of queryset, and reuses the stored result. But this don't work of there are more methods called on the queryset after evaluation.

For example, the following statements don't make any database query:

```py
queryset = Lead.objects.all()  # No database query
queryset = queryset.exclude(is_dummy=True)  # No database query
queryset = queryset.filter(assigned_to_id__isnull=True)  # No database query
queryset = queryset.filter(status__in=[1, 4])  # No database query
queryset = queryset.order_by("-updated_at")  # No database query
```

While these statements makes a database query.
```py
lead_count = queryset.count()  # Makes a query, because of count()
lead_ids = list(queryset.values("id"))  # Makes a query, because list()
for lead in queryset:  # Makes a query, because of for loop
    print(lead)

print(queryset)  # No query, because it is already evaluated

for lead in queryset.filter(box_stage_id="..."): # Another query, because we filtered the cached result
    print(lead)

```

This allows if chaining of methods to perform complex queries. But need to be aware of when a database query happens, to ignore such operations while chaining.


#### Advantages of Django storing queryset response

As mentioned, after evaluating the queryset, Django stores the result in memory, so that another database query isn't necessary.
The main advantage of this is when looping through the queryset is necessary, but also need to find count or whether it exists.

Consider the following example:
```py
boxes = Box.objects.all()
if boxes.exists():
    box_count = boxes.count()
    for box in boxes:
        ...
```

This seems better at first glance, because its using database calls to find whether boxes exist, and get their count, which is faster than python.
But this isn't the optimized way :x:. This block makes 3 different queries for finding whether boxes exist, getting the count, and looping through them.

Check the following alternative:
```py
boxes = Box.objects.all()
if boxes:
    box_count = len(boxes)
    for box in boxes:
        ...
```

The advantage of this is, as looping through the queryset is necessary, Django should fetch the results from database either way. Hence, it's better to remove the extra effort of using queries for finding count and whether it exists. It fetches the results during `if boxes:`, and for subsequent calls, it uses the result in memory.

## 2. Write type methods

The following are some of basic methods for writing or storing to database:

3. **`create()`:** Creates a row in database.
4. **`save()`:** Update the row in database based on object. [Part of object, not queryset]
5. **`update()`:** Update some fields in database.
6. **`bulk_create()`:** Create multiple instances at once.
7. **`bulk_update()`:** Update multiple instances at once. Same as update, but this takes list of objects to update.

All of these methods immediately makes a database query. When creating multiple objects, bulk methods are better than individual methods, because they make way less queries.
This is one way to reduce the number for queries.

> [!important] 
> Creating an instance with `Model(...)`, for example `Box(...)`, creates an object in python. ***This don't store in database until calling*** `save()`.

### Issues with `bulk_create()` and `bulk_update()`
There are 2 major issues when using the bulk create and update methods:

1. The signal events of Django aren't triggered.
2. The library used for position fields *doesn't support `bulk_create()`*. It sets all the positions to 0.

---
# Basic query optimization

## Reducing number of fields

When using `get()` or any query, Django by default gets all the fields of the model. This isn't much of an issue for small models.
But if the model has too many fields, then it can be slow because of getting unnecessary fields.

For example, lets say you only need the name of a `Box`, then this is the common, but wrong :x:, way:

```py
boxes = Box.objects.all()
box_names = [box.box_name for box in boxes]
```

There are multiple ways to reduce the fields fetched both when getting an instance, as well as for multiple instances.

### 1. `values()` and `values_list()`

These accept a list of fields in the parameters, and only fetches those fields.

For example:
```py
box_values = Box.objects.values("id", "box_name")[:10]
print(list(box_values))
```

This only fetches `id` and `box_name` for each box, and gives as list of dictionaries.

### 2. `only()` and `defer()`

`only()` fetches ***only*** the provided fields. `defer()` is the opposite, it fetches all the fields other than the provided.

For example:
```py
boxes = Box.objects.only("box_name", "type_of_box")[:10]
for box in boxes:
    print(box.box_name, box.type_of_box)
```

> [!tip]+ "Issue with `defer()`"
> When using `defer()`, the deferred fields should change for every new unnecessary field added.

> [!important]
> `only()` and `defer()` can't mix with one another. Either use `only()` or use `defer()`.

> [!warning]
> When using `only()`, make sure to ***not use any fields that aren't fetched***. Otherwise, Django makes another database call.

## Reducing number of queries

The common mistake made in ORM is the `n+1` query problem. Following example illustrates this.

Consider a `BoxStage` model, which has a foreign key to `Box` model. Then the following is common, but un-optimized way, to get box information of each stage.

```py
stages = BoxStage.objects.all()
box_names = {}
for stage in stages:
    box_names[stage.id] = stage.box.box_name
```

***This is wrong way to do this :x:***
Here, `for stage in stages:` makes a database query to fetch the stages. ***But, this doesn't fetch the box information.***
When you do `stage.box.name`, this makes a separate query to fetch the box information. And since this is in a loop, it makes n separate queries.

Hence, this called as `n+1` query problem, because for `n` stages, this makes `n+1` queries.


There are 3 major solutions for this:

1. `annotate()`
2. `select_related()`
3. `prefetch_related()`

### 1. `annotate()`

This creates a new field that Django fetches along with object. Django has many ways to provide logic for the fields.

The common ones are: `F()`, `Count()`, `Exists()`. And there are also complicated ways using `Subquery()`.

For example:
```py
from django.db.models import F

stages = BoxStage.objects.filter(...).annotate(box_name=F("box__box_name"))
for stage in stages:
    print(stage.box_name)
```

### 2. `select_related()`

This is helpful for 2 types of relations:

1. One to One
2. Many to One

This fetches the specified model fields using SQL join.

For example:
```py
stages = BoxStage.objects.filter(...).select_related("box")
for stage in stages:
    print(stage.box.box_name)
```

> [!tip]
> It's better to use `select_related()` with `only()` or `values()` to only get required fields.

### 3. `prefetch_related()`

This is helpful for following types of relation:

1. One to One
2. Many to One
3. One to Many
4. Many to Many

This makes a second query to fetch the objects needed, using foreign key.

For example: the following gets the boxes, along with all the stages for each box.
```py
boxes = Box.objects.prefetch_related("boxstage_set")[:10]

box_stages = {}
for box in boxes:
    box_stages[box.id] = [stage.stage_name for stage in box.boxstage_set.all()]
```

> [!important]
> `prefetch_related()` passes ids of all objects in query, which can make query large if there is no pagination.

> [!tip]
> As a general rule of thumb, prefer `select_related()` for one to one and many to one relations.
>
> Prefer `prefetch_related()` for one to many or many to many.
> 
> But if the related objects are way less than original objects, it is better to use `prefetch_related()`.

## Note on database joins

Database is good at handling joins, so it's not necessary to actively ignore joins. But it's better to analyze the generated query. And optimize/remove the join only when it's too expensive.

Django provides `explain()` method to analyze the query.

```py
boxes = Box.objects.filter(type_of_box=1).order_by("-created_at")
print(boxes.explain())
```

The main places where joins causes an issue are:

- Complex filtering containing current model fields along with joined model fields. Especially if it's one to many or many to many.
- Having too many joins.