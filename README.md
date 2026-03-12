# SpringBootJPAHibernate
This repo is for SpringBootJPAHibernate knowledge and Tutorials


# JPA/Hibernate Deep Dive: Performance Optimization in Subzone Search

## 📚 Table of Contents

1. [Introduction](#introduction)
2. [Understanding JPA/Hibernate Fundamentals](#fundamentals)
3. [The Classic N+1 Problem](#n1-problem)
4. [The Cartesian Product Problem](#cartesian-product)
5. [Pagination Nightmare with JOIN FETCH](#pagination-nightmare)
6. [Lazy Loading vs Eager Loading](#lazy-vs-eager)
7. [Filtering with JOIN FETCH and WHERE Clauses](#filtering-with-join-fetch)
8. [Sorting on Child Entity Fields](#sorting-challenges)
9. [The Two-Query Solution](#two-query-solution)
10. [Complete Example Walkthrough](#complete-example)
11. [Best Practices & Patterns](#best-practices)

---

## 1. Introduction {#introduction}

This document provides a deep understanding of JPA/Hibernate behavior and the sophisticated performance optimization techniques implemented in the WMS Location Count subzone search feature by Shalinee Chauhan.

### The Challenge
Search for subzones with:
- **Pagination** (page 2, size 10)
- **Sorting** on child entity fields (e.g., assigned user's name)
- **Filtering** across multiple entity levels (subzone → location → product)
- **Optional associations** (conditionally load users, locations, perform actions)

### Entity Hierarchy
```
CountRequest (1)
  └── CountLayoutSubzone (N) ← We're paginating THIS level
        ├── CountAssignedUser (N)
        │     └── CountPerformAction (N)
        └── CountLayoutLocation (N)
              └── CountPerformAction (N)
```

---

## 2. Understanding JPA/Hibernate Fundamentals {#fundamentals}

### How Hibernate Maps Entities to SQL

#### Entity Definitions
```java
@Entity
@Table(name = "count_layout_subzone")
public class CountLayoutSubzoneEntity {
    @Id
    private String countLayoutSubzoneKey;

    @ManyToOne(fetch = FetchType.LAZY)  // ← Default: LAZY
    @JoinColumn(name = "count_request_key")
    private CountRequestEntity countRequestEntity;

    @OneToMany(mappedBy = "countLayoutSubzoneEntity", fetch = FetchType.LAZY)
    private Set<CountAssignedUserEntity> countAssignedUserEntities;

    @OneToMany(mappedBy = "countLayoutSubzoneEntity", fetch = FetchType.LAZY)
    private Set<CountLayoutLocationEntity> countLayoutLocationEntities;

    private String parentLayoutName;
    private Long totalInvQtyExpected;
    // ... other fields
}

@Entity
@Table(name = "count_assigned_user")
public class CountAssignedUserEntity {
    @Id
    private String assignedUserKey;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "count_layout_subzone_key")
    private CountLayoutSubzoneEntity countLayoutSubzoneEntity;

    private String userId;
    private String firstName;
    private String lastName;
    // ... other fields
}

@Entity
@Table(name = "count_layout_location")
public class CountLayoutLocationEntity {
    @Id
    private String countLayoutLocationKey;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "count_layout_subzone_key")
    private CountLayoutSubzoneEntity countLayoutSubzoneEntity;

    private String layoutName;
    private String productId;
    // ... other fields
}
```

### Fetch Types Explained

| Fetch Type | Behavior | SQL Impact |
|------------|----------|------------|
| **LAZY** (default for collections) | Child entities loaded on-demand when accessed | Separate SELECT for each access |
| **EAGER** (default for @ManyToOne) | Child entities loaded immediately with parent | JOIN in single SELECT |

---

## 3. The Classic N+1 Problem {#n1-problem}

### The Problem Explained

**Scenario**: Fetch 10 subzones and access their assigned users

#### Naive Approach
```java
// Query subzones
List<CountLayoutSubzoneEntity> subzones =
    repository.findByCountRequestKey(countRequestKey);

// Later, access users (LAZY loading triggers)
for (CountLayoutSubzoneEntity subzone : subzones) {
    Set<CountAssignedUserEntity> users = subzone.getCountAssignedUserEntities();
    // ← This line triggers a database query!
    System.out.println("Users: " + users.size());
}
```

#### SQL Queries Generated
```sql
-- Query 1: Fetch subzones
SELECT * FROM count_layout_subzone WHERE count_request_key = ?;
-- Returns: 10 rows

-- Query 2: Fetch users for subzone 1
SELECT * FROM count_assigned_user WHERE count_layout_subzone_key = 'subzone1';

-- Query 3: Fetch users for subzone 2
SELECT * FROM count_assigned_user WHERE count_layout_subzone_key = 'subzone2';

-- Query 4: Fetch users for subzone 3
SELECT * FROM count_assigned_user WHERE count_layout_subzone_key = 'subzone3';

-- ... (continues for all 10 subzones)

-- Query 11: Fetch users for subzone 10
SELECT * FROM count_assigned_user WHERE count_layout_subzone_key = 'subzone10';
```

**Total Queries**: **1 + N = 11 queries** (1 for subzones + 10 for users)

#### Performance Impact
- **10 subzones** → 11 queries
- **100 subzones** → 101 queries
- **1,000 subzones** → 1,001 queries 💥

Each query has overhead:
- Network round-trip to database
- Query parsing and execution
- Result set marshalling

### The N+1 Solution: JOIN FETCH

```java
@Query("""
    SELECT DISTINCT s
    FROM CountLayoutSubzoneEntity s
    JOIN FETCH s.countAssignedUserEntities
    WHERE s.countRequestEntity.countRequestKey = :countRequestKey
""")
List<CountLayoutSubzoneEntity> findSubzonesWithUsers(String countRequestKey);
```

#### SQL Generated
```sql
SELECT DISTINCT
    subzone.*,
    user.*
FROM count_layout_subzone subzone
INNER JOIN count_assigned_user user
    ON user.count_layout_subzone_key = subzone.count_layout_subzone_key
WHERE subzone.count_request_key = ?;
```

**Total Queries**: **1 query** ✅

**But wait!** This introduces a NEW problem... 👇

---

## 4. The Cartesian Product Problem {#cartesian-product}

### What is a Cartesian Product?

When you JOIN multiple one-to-many relationships, you get **duplicate rows for the parent entity**.

#### Example Data

**count_layout_subzone table:**
| subzone_key | parent_layout_name |
|-------------|-------------------|
| SZ001 | Subzone A |
| SZ002 | Subzone B |

**count_assigned_user table:**
| assigned_user_key | subzone_key | user_id |
|-------------------|-------------|---------|
| AU001 | SZ001 | john |
| AU002 | SZ001 | jane |
| AU003 | SZ001 | bob |
| AU004 | SZ002 | alice |
| AU005 | SZ002 | charlie |

**count_layout_location table:**
| location_key | subzone_key | layout_name |
|--------------|-------------|-------------|
| LOC001 | SZ001 | Slot-A1 |
| LOC002 | SZ001 | Slot-A2 |
| LOC003 | SZ002 | Slot-B1 |
| LOC004 | SZ002 | Slot-B2 |
| LOC005 | SZ002 | Slot-B3 |

### JOIN FETCH with One Collection

```sql
SELECT DISTINCT subzone.*, user.*
FROM count_layout_subzone subzone
JOIN count_assigned_user user ON user.subzone_key = subzone.subzone_key;
```

**Result Set:**
| subzone_key | parent_layout_name | user_id |
|-------------|-------------------|---------|
| SZ001 | Subzone A | john |
| SZ001 | Subzone A | jane |
| SZ001 | Subzone A | bob |
| SZ002 | Subzone B | alice |
| SZ002 | Subzone B | charlie |

**Rows returned**: 5 (3 for SZ001 + 2 for SZ002)

**Hibernate behavior**:
- Sees 5 rows
- De-duplicates to 2 **unique** subzone entities (SZ001, SZ002)
- Populates `countAssignedUserEntities` collection for each

✅ **Works correctly!**

### JOIN FETCH with TWO Collections (THE PROBLEM!)

```sql
SELECT DISTINCT subzone.*, user.*, location.*
FROM count_layout_subzone subzone
JOIN count_assigned_user user ON user.subzone_key = subzone.subzone_key
JOIN count_layout_location location ON location.subzone_key = subzone.subzone_key;
```

**Result Set (Cartesian Product):**
| subzone_key | parent_layout_name | user_id | layout_name |
|-------------|-------------------|---------|-------------|
| SZ001 | Subzone A | john | Slot-A1 |
| SZ001 | Subzone A | john | Slot-A2 |
| SZ001 | Subzone A | jane | Slot-A1 |
| SZ001 | Subzone A | jane | Slot-A2 |
| SZ001 | Subzone A | bob | Slot-A1 |
| SZ001 | Subzone A | bob | Slot-A2 |
| SZ002 | Subzone B | alice | Slot-B1 |
| SZ002 | Subzone B | alice | Slot-B2 |
| SZ002 | Subzone B | alice | Slot-B3 |
| SZ002 | Subzone B | charlie | Slot-B1 |
| SZ002 | Subzone B | charlie | Slot-B2 |
| SZ002 | Subzone B | charlie | Slot-B3 |

**Rows returned**: **12** 😱

For SZ001:
- 3 users × 2 locations = **6 rows**

For SZ002:
- 2 users × 3 locations = **6 rows**

**Formula**: `rows = users × locations` (Cartesian product)

#### Problems:
1. **Massive result sets**: 100 users × 200 locations = 20,000 rows for ONE subzone!
2. **Network bandwidth**: Transferring huge result sets
3. **Memory usage**: Hibernate processes all rows
4. **Pagination breaks** (see next section)

### Hibernate Warning

```
HHH000104: firstResult/maxResults specified with collection fetch;
applying in memory!
```

Hibernate **cannot paginate** in the database when you use JOIN FETCH with multiple collections. It loads **ALL** results into memory and paginates them in Java! 💥

---

## 5. Pagination Nightmare with JOIN FETCH {#pagination-nightmare}

### The Pagination Problem

**Goal**: Get page 2 (items 11-20) of subzones, sorted by user's name

#### Attempt 1: Naive JOIN FETCH with Pagination

```java
@Query("""
    SELECT DISTINCT s
    FROM CountLayoutSubzoneEntity s
    JOIN FETCH s.countAssignedUserEntities user
    LEFT JOIN FETCH s.countLayoutLocationEntities location
    WHERE s.countRequestEntity.countRequestKey = :countRequestKey
    ORDER BY user.firstName ASC
""")
Page<CountLayoutSubzoneEntity> findSubzones(
    String countRequestKey,
    Pageable pageable  // ← page=1, size=10
);
```

#### SQL Generated
```sql
SELECT DISTINCT
    subzone.*,
    user.*,
    location.*
FROM count_layout_subzone subzone
JOIN count_assigned_user user ON user.subzone_key = subzone.subzone_key
LEFT JOIN count_layout_location location ON location.subzone_key = subzone.subzone_key
WHERE subzone.count_request_key = ?
ORDER BY user.first_name ASC
LIMIT 10 OFFSET 10;  -- Page 2
```

#### Why This BREAKS Pagination

**Database perspective (what SQL does):**
```
Row 1:  SZ001, Subzone A, alice, Slot-A1
Row 2:  SZ001, Subzone A, alice, Slot-A2
Row 3:  SZ001, Subzone A, bob, Slot-A1
Row 4:  SZ001, Subzone A, bob, Slot-A2
Row 5:  SZ001, Subzone A, charlie, Slot-A1
Row 6:  SZ001, Subzone A, charlie, Slot-A2  ← SZ001 has 6 rows total
Row 7:  SZ002, Subzone B, alice, Slot-B1
Row 8:  SZ002, Subzone B, alice, Slot-B2
Row 9:  SZ002, Subzone B, dave, Slot-B1
Row 10: SZ002, Subzone B, dave, Slot-B2     ← LIMIT 10 stops here!
```

**LIMIT 10** returns rows 1-10, which contain:
- ✅ Complete data for SZ001 (rows 1-6)
- ❌ **Partial** data for SZ002 (rows 7-10, but SZ002 has 6 total rows!)

**Hibernate behavior:**
- Receives 10 rows
- De-duplicates to 2 subzone entities (SZ001, SZ002)
- But SZ002's collections are **INCOMPLETE**! 😱

**Result**: Wrong data, incomplete collections, unpredictable behavior

#### Hibernate's "Solution": In-Memory Pagination

```
HHH000104: firstResult/maxResults specified with collection fetch;
applying in memory!
```

**What Hibernate actually does:**
1. **Ignores** LIMIT/OFFSET in SQL
2. Fetches **ALL** matching rows from database (could be 100,000+ rows!)
3. De-duplicates in memory
4. Paginates in Java

```sql
-- What Hibernate actually executes
SELECT DISTINCT
    subzone.*,
    user.*,
    location.*
FROM count_layout_subzone subzone
JOIN count_assigned_user user ON user.subzone_key = subzone.subzone_key
LEFT JOIN count_layout_location location ON location.subzone_key = subzone.subzone_key
WHERE subzone.count_request_key = ?
ORDER BY user.first_name ASC;
-- NO LIMIT! Fetches ALL rows! 💥
```

**Result:**
- **1,000 subzones** × **3 users** × **5 locations** = **15,000 rows** fetched
- All loaded into Java heap
- Paginated in memory: keep rows 11-20, discard 14,980 rows
- **OutOfMemoryError** risk
- Terrible performance

#### The Count Query Problem

```sql
-- To get total count for pagination metadata
SELECT COUNT(DISTINCT subzone.subzone_key)
FROM count_layout_subzone subzone
JOIN count_assigned_user user ON user.subzone_key = subzone.subzone_key
WHERE subzone.count_request_key = ?;
```

❌ **This count is WRONG if you have duplicates from JOINs!**

The count query returns the number of **rows** (including duplicates), not the number of **distinct subzones**.

---

## 6. Lazy Loading vs Eager Loading {#lazy-vs-eager}

### Lazy Loading (Default for Collections)

```java
@OneToMany(mappedBy = "countLayoutSubzoneEntity", fetch = FetchType.LAZY)
private Set<CountAssignedUserEntity> countAssignedUserEntities;
```

#### How It Works
```java
// Step 1: Load subzone
CountLayoutSubzoneEntity subzone = repository.findById("SZ001");
// SQL: SELECT * FROM count_layout_subzone WHERE subzone_key = 'SZ001';

// Step 2: Access users (triggers lazy load)
Set<CountAssignedUserEntity> users = subzone.getCountAssignedUserEntities();
// SQL: SELECT * FROM count_assigned_user WHERE subzone_key = 'SZ001';

// Step 3: Iterate users
for (CountAssignedUserEntity user : users) {
    System.out.println(user.getUserId());  // Already loaded, no additional query
}
```

#### The LazyInitializationException

```java
@Transactional
public List<CountLayoutSubzone> getSubzones() {
    List<CountLayoutSubzoneEntity> entities = repository.findAll();
    return entities.stream()
        .map(this::mapToDto)  // ← Happens inside transaction
        .toList();
}

public CountLayoutSubzone mapToDto(CountLayoutSubzoneEntity entity) {
    CountLayoutSubzone dto = new CountLayoutSubzone();
    dto.setSubzoneName(entity.getParentLayoutName());

    // Access users OUTSIDE transaction
    Set<CountAssignedUserEntity> users = entity.getCountAssignedUserEntities();
    // 💥 LazyInitializationException: could not initialize proxy - no Session

    return dto;
}
```

**Solution**: Use `@Transactional` on the method that accesses lazy collections, or fetch eagerly when needed.

### Eager Loading

```java
@OneToMany(mappedBy = "countLayoutSubzoneEntity", fetch = FetchType.EAGER)
private Set<CountAssignedUserEntity> countAssignedUserEntities;
```

⚠️ **Dangers:**
- **Always** loads child entities, even when not needed
- Cascades to all queries on this entity
- Can cause performance issues

---

## 7. Filtering with JOIN FETCH and WHERE Clauses {#filtering-with-join-fetch}

### The Filtering Confusion

One of the most common mistakes developers make with JPA/Hibernate is **unintentionally filtering parent entities** when they only meant to filter child collections. Understanding how WHERE clauses interact with JOINs is critical.

### Sample Data for Examples

**count_layout_subzone table:**
| subzone_key | parent_layout_name | status |
|-------------|-------------------|--------|
| SZ001 | Subzone A | ACTIVE |
| SZ002 | Subzone B | ACTIVE |
| SZ003 | Subzone C | ACTIVE |

**count_assigned_user table:**
| assigned_user_key | subzone_key | user_id | role |
|-------------------|-------------|---------|------|
| AU001 | SZ001 | john | COUNTER |
| AU002 | SZ001 | jane | SUPERVISOR |
| AU003 | SZ002 | bob | COUNTER |
| AU004 | SZ003 | alice | SUPERVISOR |
| AU005 | SZ003 | charlie | COUNTER |

### Scenario 1: Filtering Parent Entities (CORRECT) ✅

**Goal**: Get only ACTIVE subzones with their assigned users

#### JPQL Query
```java
@Query("""
    SELECT DISTINCT s
    FROM CountLayoutSubzoneEntity s
    JOIN FETCH s.countAssignedUserEntities
    WHERE s.status = 'ACTIVE'
""")
List<CountLayoutSubzoneEntity> findActiveSubzonesWithUsers();
```

#### Generated SQL
```sql
SELECT DISTINCT
    subzone.*,
    user.*
FROM count_layout_subzone subzone
INNER JOIN count_assigned_user user
    ON user.count_layout_subzone_key = subzone.count_layout_subzone_key
WHERE subzone.status = 'ACTIVE';
```

#### Result Set
| subzone_key | parent_layout_name | status | user_id | role |
|-------------|-------------------|--------|---------|------|
| SZ001 | Subzone A | ACTIVE | john | COUNTER |
| SZ001 | Subzone A | ACTIVE | jane | SUPERVISOR |
| SZ002 | Subzone B | ACTIVE | bob | COUNTER |
| SZ003 | Subzone C | ACTIVE | alice | SUPERVISOR |
| SZ003 | Subzone C | ACTIVE | charlie | COUNTER |

**Hibernate Entities Created:**
```
SZ001: { users: [john, jane] }
SZ002: { users: [bob] }
SZ003: { users: [alice, charlie] }
```

✅ **Perfect!** All 3 subzones returned with ALL their users.

**Key Point**: WHERE clause on **parent entity field** (`s.status`) correctly filters parent entities.

---

### Scenario 2: Filtering on Child Entity Field with INNER JOIN (⚠️ PROBLEM!)

**Intended Goal**: Get all subzones, but **only** load users with role='COUNTER'

**What you might write:**
```java
@Query("""
    SELECT DISTINCT s
    FROM CountLayoutSubzoneEntity s
    JOIN FETCH s.countAssignedUserEntities user
    WHERE user.role = 'COUNTER'
""")
List<CountLayoutSubzoneEntity> findSubzonesWithCounterUsers();  // Misleading name!
```

#### Generated SQL
```sql
SELECT DISTINCT
    subzone.*,
    user.*
FROM count_layout_subzone subzone
INNER JOIN count_assigned_user user
    ON user.count_layout_subzone_key = subzone.count_layout_subzone_key
WHERE user.role = 'COUNTER';  -- ⚠️ This filters BOTH parent AND child!
```

#### Result Set
| subzone_key | parent_layout_name | user_id | role |
|-------------|-------------------|---------|------|
| SZ001 | Subzone A | john | COUNTER |
| SZ002 | Subzone B | bob | COUNTER |
| SZ003 | Subzone C | charlie | COUNTER |

**Hibernate Entities Created:**
```
SZ001: { users: [john] }           // ❌ Missing jane (SUPERVISOR)!
SZ002: { users: [bob] }            // ✅ Correct (only had bob)
SZ003: { users: [charlie] }        // ❌ Missing alice (SUPERVISOR)!
```

❌ **PROBLEM**:
- SZ001 appears to have only 1 user (john), but actually has 2 users total
- SZ003 appears to have only 1 user (charlie), but actually has 2 users total
- Users with role='SUPERVISOR' were **completely excluded** from the loaded collections

#### What Actually Happened

The WHERE clause on the child entity field (`user.role = 'COUNTER'`) does **TWO things**:

1. **Filters parent entities**: Only returns subzones that have AT LEAST ONE user with role='COUNTER'
2. **Filters child entities in result**: Only includes user rows matching the condition

**Why SZ002 works correctly?**
- SZ002 only has bob (COUNTER), so filtering by COUNTER still returns the complete user collection

**The Real Problem:**
When Hibernate populates the `countAssignedUserEntities` collection from the result set, it **ONLY sees the users returned by SQL**. It has no idea there are other users that were filtered out by the WHERE clause!

```java
// What Hibernate "thinks"
SZ001.users = [john]  // Hibernate believes this is the COMPLETE collection

// Reality in database
SZ001.users = [john, jane]  // There's actually another user!
```

---

### Scenario 3: Filtering Child Collections Correctly with LEFT JOIN ✅

**Goal**: Get ALL subzones, but only load users with role='COUNTER' (keep subzones even if they have no COUNTER users)

#### Using LEFT JOIN (Preferred)
```java
@Query("""
    SELECT DISTINCT s
    FROM CountLayoutSubzoneEntity s
    LEFT JOIN FETCH s.countAssignedUserEntities user
    WHERE user.role = 'COUNTER' OR user.role IS NULL
""")
List<CountLayoutSubzoneEntity> findAllSubzonesWithOptionalCounterUsers();
```

**Problem**: Still doesn't work perfectly because WHERE clause still filters!

#### The CORRECT Solution: Separate Queries

**Query 1: Get Parent Entities**
```java
@Query("SELECT DISTINCT s FROM CountLayoutSubzoneEntity s WHERE s.status = 'ACTIVE'")
List<CountLayoutSubzoneEntity> findActiveSubzones();
```

**Query 2: Get Filtered Child Entities**
```java
@Query("""
    SELECT DISTINCT user
    FROM CountAssignedUserEntity user
    WHERE user.countLayoutSubzoneEntity.countLayoutSubzoneKey IN :subzoneKeys
      AND user.role = 'COUNTER'
""")
List<CountAssignedUserEntity> findCounterUsersBySubzoneKeys(@Param("subzoneKeys") List<String> subzoneKeys);
```

**Manually Merge:**
```java
// Get subzones
List<CountLayoutSubzoneEntity> subzones = findActiveSubzones();
List<String> subzoneKeys = subzones.stream()
    .map(CountLayoutSubzoneEntity::getCountLayoutSubzoneKey)
    .toList();

// Get filtered users
List<CountAssignedUserEntity> counterUsers = findCounterUsersBySubzoneKeys(subzoneKeys);

// Merge: Populate subzone collections with filtered users
Map<String, List<CountAssignedUserEntity>> usersBySubzone = counterUsers.stream()
    .collect(Collectors.groupingBy(
        user -> user.getCountLayoutSubzoneEntity().getCountLayoutSubzoneKey()
    ));

subzones.forEach(subzone -> {
    String key = subzone.getCountLayoutSubzoneKey();
    List<CountAssignedUserEntity> filteredUsers = usersBySubzone.getOrDefault(key, List.of());

    // Initialize collection to avoid lazy loading
    subzone.setCountAssignedUserEntities(new HashSet<>(filteredUsers));
});
```

**Result:**
```
SZ001: { users: [john] }          // ✅ Only COUNTER users, but we KNOW others exist
SZ002: { users: [bob] }           // ✅ Only COUNTER users
SZ003: { users: [charlie] }       // ✅ Only COUNTER users
```

✅ **Correct!** We intentionally loaded a filtered subset of users, and we're aware of it.

---

### Scenario 4: Complex Filtering - Multiple Conditions

**Goal**: Get subzones with status='ACTIVE' AND have users with role='COUNTER'

#### Using JOIN with Multiple WHERE Conditions
```java
@Query("""
    SELECT DISTINCT s
    FROM CountLayoutSubzoneEntity s
    JOIN FETCH s.countAssignedUserEntities user
    WHERE s.status = 'ACTIVE'
      AND user.role = 'COUNTER'
""")
List<CountLayoutSubzoneEntity> findActiveSubzonesWithCounters();
```

#### Generated SQL
```sql
SELECT DISTINCT
    subzone.*,
    user.*
FROM count_layout_subzone subzone
INNER JOIN count_assigned_user user
    ON user.count_layout_subzone_key = subzone.count_layout_subzone_key
WHERE subzone.status = 'ACTIVE'
  AND user.role = 'COUNTER';
```

#### Result Analysis

**Database Result:**
- Only returns subzones that are ACTIVE **AND** have at least one COUNTER user
- Only includes COUNTER users in the result set
- **Missing**: Subzones that are ACTIVE but have NO COUNTER users
- **Incomplete**: SUPERVISOR users are excluded from collections

**What gets returned:**
```
SZ001: { users: [john] }          // Missing jane!
SZ002: { users: [bob] }           // Complete (only had bob)
SZ003: { users: [charlie] }       // Missing alice!
```

**What if SZ002's only user was a SUPERVISOR?**
- SZ002 would be **completely excluded** from results!
- Even though SZ002 is ACTIVE, it has no COUNTER users

---

### Scenario 5: LEFT JOIN vs INNER JOIN Behavior

**Sample Data Addition:**

| subzone_key | parent_layout_name | status |
|-------------|-------------------|--------|
| SZ004 | Subzone D | ACTIVE |

No users assigned to SZ004.

#### Using INNER JOIN
```java
@Query("""
    SELECT DISTINCT s
    FROM CountLayoutSubzoneEntity s
    JOIN FETCH s.countAssignedUserEntities
    WHERE s.status = 'ACTIVE'
""")
List<CountLayoutSubzoneEntity> findActiveSubzonesWithUsers();
```

```sql
SELECT DISTINCT subzone.*, user.*
FROM count_layout_subzone subzone
INNER JOIN count_assigned_user user ON user.count_layout_subzone_key = subzone.count_layout_subzone_key
WHERE subzone.status = 'ACTIVE';
```

**Result:** SZ001, SZ002, SZ003 (SZ004 **excluded** because it has no users!)

#### Using LEFT JOIN
```java
@Query("""
    SELECT DISTINCT s
    FROM CountLayoutSubzoneEntity s
    LEFT JOIN FETCH s.countAssignedUserEntities
    WHERE s.status = 'ACTIVE'
""")
List<CountLayoutSubzoneEntity> findAllActiveSubzonesWithOptionalUsers();
```

```sql
SELECT DISTINCT subzone.*, user.*
FROM count_layout_subzone subzone
LEFT JOIN count_assigned_user user ON user.count_layout_subzone_key = subzone.count_layout_subzone_key
WHERE subzone.status = 'ACTIVE';
```

**Result:** SZ001, SZ002, SZ003, **SZ004** (SZ004 included with empty users collection!)

**Key Difference:**
- **INNER JOIN**: Only returns parents that have matching children
- **LEFT JOIN**: Returns all parents, even those without children

---

### Scenario 6: Filtering with Subqueries (Advanced)

**Goal**: Get all ACTIVE subzones that have at least one COUNTER user, but load ALL users for those subzones

#### Using EXISTS Subquery
```java
@Query("""
    SELECT DISTINCT s
    FROM CountLayoutSubzoneEntity s
    LEFT JOIN FETCH s.countAssignedUserEntities
    WHERE s.status = 'ACTIVE'
      AND EXISTS (
        SELECT 1
        FROM CountAssignedUserEntity user
        WHERE user.countLayoutSubzoneEntity = s
          AND user.role = 'COUNTER'
      )
""")
List<CountLayoutSubzoneEntity> findActiveSubzonesHavingCountersWithAllUsers();
```

#### Generated SQL
```sql
SELECT DISTINCT
    subzone.*,
    user.*
FROM count_layout_subzone subzone
LEFT JOIN count_assigned_user user ON user.count_layout_subzone_key = subzone.count_layout_subzone_key
WHERE subzone.status = 'ACTIVE'
  AND EXISTS (
    SELECT 1
    FROM count_assigned_user u2
    WHERE u2.count_layout_subzone_key = subzone.count_layout_subzone_key
      AND u2.role = 'COUNTER'
  );
```

**Result:**
```
SZ001: { users: [john, jane] }     // ✅ All users loaded!
SZ002: { users: [bob] }            // ✅ All users loaded!
SZ003: { users: [alice, charlie] } // ✅ All users loaded!
```

✅ **Perfect!**
- Parent filtering: Only ACTIVE subzones that have at least one COUNTER
- Child loading: ALL users for matching subzones (not just COUNTERs)

**Why this works:**
- EXISTS subquery filters parent entities based on child criteria
- LEFT JOIN FETCH loads ALL children (no WHERE filter on child)
- Separation of concerns: filter parents vs load children

---

### Common Pitfalls Summary

| Scenario | Query Pattern | Result | Issue |
|----------|--------------|--------|-------|
| **Filter parents only** | `WHERE parent.field = value` | ✅ Correct | Parent filtering works as expected |
| **Filter by child field with INNER JOIN** | `JOIN FETCH child WHERE child.field = value` | ❌ Wrong | Filters BOTH parents AND children, incomplete collections |
| **Filter by child field with LEFT JOIN** | `LEFT JOIN FETCH child WHERE child.field = value` | ⚠️ Confusing | Still filters parents if child field is NULL |
| **EXISTS subquery + LEFT JOIN** | `LEFT JOIN FETCH child WHERE EXISTS(...)` | ✅ Correct | Filters parents, loads all children |
| **Separate queries** | Query 1: parents, Query 2: filtered children | ✅ Correct | Full control, explicit intent |

---

### Best Practices for Filtering with JOIN FETCH

#### ✅ DO: Filter Parent Entities by Parent Fields
```java
@Query("""
    SELECT DISTINCT s
    FROM CountLayoutSubzoneEntity s
    JOIN FETCH s.countAssignedUserEntities
    WHERE s.status = :status
      AND s.facilityId = :facilityId
""")
List<CountLayoutSubzoneEntity> findByStatusAndFacility(String status, String facilityId);
```

**Safe because**: WHERE conditions only reference parent entity fields.

#### ✅ DO: Use EXISTS for Complex Child-Based Parent Filtering
```java
@Query("""
    SELECT DISTINCT s
    FROM CountLayoutSubzoneEntity s
    LEFT JOIN FETCH s.countAssignedUserEntities
    WHERE s.status = 'ACTIVE'
      AND EXISTS (
        SELECT 1 FROM CountAssignedUserEntity user
        WHERE user.countLayoutSubzoneEntity = s AND user.active = true
      )
""")
List<CountLayoutSubzoneEntity> findActiveSubzonesWithActiveUsers();
```

**Good because**: EXISTS filters parents without affecting child collection loading.

#### ✅ DO: Use Separate Queries for Filtered Child Collections
```java
// Query 1: Get parent keys
List<String> subzoneKeys = getFilteredSubzoneKeys(criteria);

// Query 2: Get full entities
@Query("SELECT DISTINCT s FROM Subzone s WHERE s.id IN :ids")
List<Subzone> getSubzones(@Param("ids") List<String> ids);

// Query 3: Get filtered children
@Query("SELECT u FROM User u WHERE u.subzone.id IN :ids AND u.role = :role")
List<User> getFilteredUsers(@Param("ids") List<String> ids, @Param("role") String role);
```

**Best approach**: Explicit, clear intent, no hidden filtering.

#### ❌ DON'T: Filter by Child Fields with JOIN FETCH
```java
// ❌ BAD - Creates incomplete collections!
@Query("""
    SELECT DISTINCT s
    FROM CountLayoutSubzoneEntity s
    JOIN FETCH s.countAssignedUserEntities user
    WHERE user.role = 'COUNTER'
""")
List<CountLayoutSubzoneEntity> findSubzonesWithCounters();  // Misleading!
```

**Why bad**:
- Developers expect ALL users to be loaded
- Collections appear complete but are actually filtered
- Causes bugs when iterating over "all" users

#### ❌ DON'T: Mix Parent and Child Filtering Without Understanding
```java
// ❌ CONFUSING - What does this return?
@Query("""
    SELECT DISTINCT s
    FROM CountLayoutSubzoneEntity s
    JOIN FETCH s.countAssignedUserEntities user
    WHERE s.status = 'ACTIVE'
      AND user.role = 'COUNTER'
      AND user.firstName LIKE :namePattern
""")
List<CountLayoutSubzoneEntity> findActiveWithCountersByName(String namePattern);
```

**Why confusing**: Mixes three concerns - active subzones, counter role, name search.

---

### How Hibernate Applies WHERE Clauses

**Key Insight**: Hibernate applies WHERE clauses to the **SQL result set**, not to entity relationships.

#### Example: What Hibernate Sees

**SQL Result Set:**
```
| subzone_key | subzone_name | user_id | user_role |
|-------------|--------------|---------|-----------|
| SZ001       | Subzone A    | john    | COUNTER   |
| SZ002       | Subzone B    | bob     | COUNTER   |
```

**Hibernate's Processing:**
1. Groups rows by `subzone_key` (de-duplication)
2. Creates entity: `SZ001` with users collection `[john]`
3. Creates entity: `SZ002` with users collection `[bob]`

**Hibernate does NOT:**
- Query database again to check for other users
- Know that there might be filtered-out children
- Mark collections as "partially loaded"

**The collection appears complete** from Hibernate's perspective!

---

### Debugging Filtered Collections

#### Enable SQL Logging
```yaml
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
logging.level.org.hibernate.SQL=DEBUG
```

#### Check Result Set Size
```java
List<CountLayoutSubzoneEntity> subzones = repository.findSubzonesWithCounters();

for (CountLayoutSubzoneEntity subzone : subzones) {
    int userCount = subzone.getCountAssignedUserEntities().size();

    // Cross-check with separate count query
    long actualUserCount = userRepository.countBySubzoneKey(subzone.getCountLayoutSubzoneKey());

    if (userCount != actualUserCount) {
        log.warn("Subzone {} has incomplete user collection: loaded={}, actual={}",
            subzone.getCountLayoutSubzoneKey(), userCount, actualUserCount);
    }
}
```

---

### Real-World WMS Example: The Projection Solution

From the actual WMS implementation, notice how filtering is handled:

```properties
# Query 1: Projection with filters on ALL entity levels
CountLayoutSubzoneEntity.getSortedSubzoneKeysWithProjection=\
SELECT cls.countLayoutSubzoneKey \
FROM CountLayoutSubzoneEntity cls \
LEFT JOIN cls.countLayoutLocationEntities location \
LEFT JOIN cls.countAssignedUserEntities cau \
WHERE cls.countRequestEntity.countRequestKey = :countRequestKey \
AND (:parentLayoutDistinctName IS NULL OR cls.parentLayoutDistinctName = :parentLayoutDistinctName) \
AND (:userId IS NULL OR cau.userId = :userId) \
AND (:layoutName IS NULL OR location.layoutName = :layoutName) \
GROUP BY cls.countLayoutSubzoneKey
```

**Key Points:**
- ✅ Uses LEFT JOIN (includes subzones even without matching children)
- ✅ Filters applied via WHERE clauses
- ✅ GROUP BY collapses duplicates from JOINs
- ✅ Returns ONLY keys (projection), not full entities
- ✅ Subsequent queries fetch full entities by keys

```java
// Query 2: Fetch full entities WITHOUT filtering
@Query("SELECT DISTINCT cls FROM CountLayoutSubzoneEntity cls WHERE cls.countLayoutSubzoneKey IN :keys")
List<CountLayoutSubzoneEntity> getSubzonesByKeys(@Param("keys") List<String> keys);

// Query 3: Fetch child entities WITH optional filtering
@Query("""
    SELECT DISTINCT user
    FROM CountAssignedUserEntity user
    WHERE user.countLayoutSubzoneEntity.countLayoutSubzoneKey IN :subzoneKeys
      AND (:userId IS NULL OR user.userId = :userId)
""")
List<CountAssignedUserEntity> getUsersBySubzoneKeys(
    @Param("subzoneKeys") List<String> subzoneKeys,
    @Param("userId") String userId
);
```

✅ **Perfect separation**:
- Projection query handles ALL filtering (parent + children)
- Entity fetch queries load based on filtered keys
- Child queries can have their own additional filters
- Clear, explicit, no hidden behavior

---

## 8. Sorting on Child Entity Fields {#sorting-challenges}

### The Sorting Challenge

**Requirement**: Sort subzones by assigned user's first name

#### Problem with Lazy Loading
```java
Page<CountLayoutSubzoneEntity> subzones = repository.findAll(
    PageRequest.of(0, 10, Sort.by("countAssignedUserEntities.firstName"))
);
```

❌ **Fails!** You cannot sort on a lazy collection in JPA.

#### Problem with JOIN FETCH
```sql
SELECT DISTINCT subzone.*
FROM count_layout_subzone subzone
ORDER BY user.first_name ASC;  -- ❌ 'user' is not in FROM clause!
```

You need the child table in the query to sort by its columns.

#### Solution: Include Child Table for Sorting

```sql
SELECT DISTINCT
    subzone.subzone_key,
    subzone.parent_layout_name,
    MIN(user.first_name || ' ' || user.last_name) as user_name  -- For sorting
FROM count_layout_subzone subzone
LEFT JOIN count_assigned_user user ON user.subzone_key = subzone.subzone_key
WHERE subzone.count_request_key = ?
GROUP BY subzone.subzone_key, subzone.parent_layout_name
ORDER BY user_name ASC
LIMIT 10 OFFSET 10;
```

**Key points:**
- LEFT JOIN to include child table
- GROUP BY to collapse duplicates
- MIN/MAX aggregate to get a single value for sorting
- Only fetch columns needed for sorting (projection)

---

## 8. The Two-Query Solution {#two-query-solution}

### The Elegant Solution by Shalinee

The implementation uses a **two-query approach** that solves all the problems:

#### Architecture Overview
```
┌─────────────────────────────────────────────────────┐
│ QUERY 1: Get Sorted & Paginated Keys (Projection)  │
│─────────────────────────────────────────────────────│
│ • JOINs child tables for sorting/filtering         │
│ • Returns ONLY keys + sort columns (minimal data)  │
│ • Database handles DISTINCT, ORDER BY, LIMIT       │
│ • Accurate pagination metadata                     │
└─────────────────────────────────────────────────────┘
                        ↓
            [key1, key2, ..., key10]
                        ↓
┌─────────────────────────────────────────────────────┐
│ QUERY 2: Fetch Full Entities for Page Keys         │
│─────────────────────────────────────────────────────│
│ • WHERE key IN (:keys) ← Only page's keys          │
│ • JOIN FETCH countRequest (needed immediately)     │
│ • No duplicates (fetching by PK)                   │
│ • Fast and memory efficient                        │
└─────────────────────────────────────────────────────┘
                        ↓
        [entity1, entity2, ..., entity10]
                        ↓
┌─────────────────────────────────────────────────────┐
│ Reorder entities to match Query 1 sort order       │
└─────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────┐
│ QUERIES 3-5: Conditionally Fetch Child Entities    │
│─────────────────────────────────────────────────────│
│ • Query 3: Assigned Users (if requested)           │
│ • Query 4: Locations (if requested)                │
│ • Query 5: Perform Actions (if requested)          │
│ • Separate queries avoid Cartesian product         │
│ • Client controls what to load                     │
└─────────────────────────────────────────────────────┘
```

### Query 1: Sorted Keys with Projection

#### JPA Named Query
```properties
# META-INF/jpa-named-queries.properties

CountLayoutSubzoneEntity.getSortedSubzoneKeysWithProjection=\
SELECT \
  cls.countLayoutSubzoneKey as subzoneKey, \
  cls.parentLayoutName as parentLayoutName, \
  cls.startTs as startTs, \
  MIN(CONCAT(COALESCE(cau.firstName, ''), ' ', COALESCE(cau.lastName, ''))) as userName, \
  MIN(CONCAT(COALESCE(userCpa.recounterFirstName, ''), ' ', COALESCE(userCpa.recounterLastName, ''))) as recounterUserName, \
  cls.totalInvAmtAbsAdjusted as totalInvAmtAbsAdjusted, \
  cls.totalInvQtyExceedsVariance as totalInvQtyExceedsVariance, \
  CASE WHEN cls.totalInvQtyCounted > 0 THEN \
    ((CAST(cls.totalInvQtyCounted AS double) - COALESCE(cls.totalInvQtyAbsAdjusted, 0)) * 100.0 / CAST(cls.totalInvQtyCounted AS double)) \
  ELSE 0.0 END as totalInvQtyAbsAccuracy \
FROM CountLayoutSubzoneEntity cls \
LEFT JOIN cls.countLayoutLocationEntities location \
LEFT JOIN cls.countAssignedUserEntities cau \
LEFT JOIN cau.countPerformActionEntities userCpa \
WHERE cls.countRequestEntity.countRequestKey = :countRequestKey \
AND (:#{#searchCriteria.parentLayoutDistinctName} IS NULL OR cls.parentLayoutDistinctName = :#{#searchCriteria.parentLayoutDistinctName}) \
AND (:#{#searchCriteria.userId} IS NULL OR cau.userId = :#{#searchCriteria.userId}) \
AND (:#{#searchCriteria.layoutName} IS NULL OR location.layoutName = :#{#searchCriteria.layoutName}) \
AND (:#{#searchCriteria.productId} IS NULL OR location.productId = :#{#searchCriteria.productId}) \
GROUP BY cls.countLayoutSubzoneKey, cls.parentLayoutName, cls.startTs, cls.totalInvAmtAbsAdjusted, cls.totalInvQtyExceedsVariance
```

#### Generated SQL (Spanner/PostgreSQL)
```sql
SELECT
  subzone.count_layout_subzone_key as subzone_key,
  subzone.parent_layout_name,
  subzone.start_ts,
  MIN(COALESCE(user.first_name, '') || ' ' || COALESCE(user.last_name, '')) as user_name,
  MIN(COALESCE(cpa.recounter_first_name, '') || ' ' || COALESCE(cpa.recounter_last_name, '')) as recounter_user_name,
  subzone.total_inv_amt_abs_adjusted,
  subzone.total_inv_qty_exceeds_variance,
  CASE
    WHEN subzone.total_inv_qty_counted > 0 THEN
      ((CAST(subzone.total_inv_qty_counted AS FLOAT64) - COALESCE(subzone.total_inv_qty_abs_adjusted, 0)) * 100.0 / CAST(subzone.total_inv_qty_counted AS FLOAT64))
    ELSE 0.0
  END as total_inv_qty_abs_accuracy
FROM count_layout_subzone subzone
LEFT JOIN count_layout_location location ON location.count_layout_subzone_key = subzone.count_layout_subzone_key
LEFT JOIN count_assigned_user user ON user.count_layout_subzone_key = subzone.count_layout_subzone_key
LEFT JOIN count_perform_action cpa ON cpa.count_assigned_user_key = user.assigned_user_key
WHERE subzone.count_request_key = ?
  AND (? IS NULL OR subzone.parent_layout_distinct_name = ?)
  AND (? IS NULL OR user.user_id = ?)
  AND (? IS NULL OR location.layout_name = ?)
  AND (? IS NULL OR location.product_id = ?)
GROUP BY
  subzone.count_layout_subzone_key,
  subzone.parent_layout_name,
  subzone.start_ts,
  subzone.total_inv_amt_abs_adjusted,
  subzone.total_inv_qty_exceeds_variance
ORDER BY user_name ASC  -- Dynamic based on sort parameter
LIMIT 10 OFFSET 10;     -- Page 2, size 10
```

**Key Features:**
✅ **Projection**: Only fetch columns needed for sorting and identification
✅ **GROUP BY**: Collapses duplicate rows from JOINs
✅ **Aggregates**: MIN/MAX to get single value for sorting
✅ **Filtering**: All filters applied at database level
✅ **Pagination**: LIMIT/OFFSET works correctly on distinct keys
✅ **Sorting**: Database sorts on child entity fields
✅ **Performance**: Minimal data transfer

**Return Type:**
```java
public interface SubzoneSortProjection {
    String getSubzoneKey();
    String getParentLayoutName();
    OffsetDateTime getStartTs();
    String getUserName();
    String getRecounterUserName();
    Float getTotalInvAmtAbsAdjusted();
    // ... other projection fields
}
```

### Query 2: Fetch Full Entities

```java
@Query("""
    SELECT DISTINCT cls
    FROM CountLayoutSubzoneEntity cls
    JOIN FETCH cls.countRequestEntity
    WHERE cls.countLayoutSubzoneKey IN :subzoneKeys
""")
List<CountLayoutSubzoneEntity> getSubzonesByKeysWithCountRequest(
    @Param("subzoneKeys") List<String> subzoneKeys
);
```

#### Generated SQL
```sql
SELECT DISTINCT
    subzone.*,
    count_request.*  -- Eagerly loaded via JOIN FETCH
FROM count_layout_subzone subzone
INNER JOIN count_request count_request ON count_request.count_request_key = subzone.count_request_key
WHERE subzone.count_layout_subzone_key IN (?, ?, ?, ?, ?, ?, ?, ?, ?, ?);  -- 10 keys from Query 1
```

**Key Features:**
✅ **No duplicates**: Fetching by primary key (IN clause with keys)
✅ **Fast**: Simple PK lookup, typically uses index
✅ **JOIN FETCH countRequest**: Load parent immediately (avoid lazy load later)
✅ **Memory efficient**: Only 10 entities (page size)

### Reordering Entities

Query 2 may not return entities in the same order as Query 1's sorted keys.

```java
// Step 1: Create map for O(1) lookup
Map<String, CountLayoutSubzoneEntity> entityMap = subzoneEntities.stream()
    .collect(Collectors.toMap(
        CountLayoutSubzoneEntity::getCountLayoutSubzoneKey,
        entity -> entity
    ));

// Step 2: Build ordered list by iterating over sorted keys from Query 1
List<CountLayoutSubzoneEntity> orderedEntities = pageSubzoneKeys.stream()
    .map(entityMap::get)
    .filter(Objects::nonNull)
    .toList();
```

**Time Complexity**: O(n) where n = page size (10)

### Queries 3-5: Conditional Child Entities

#### Query 3: Assigned Users (if requested)
```sql
SELECT DISTINCT user
FROM count_assigned_user user
INNER JOIN count_layout_subzone subzone ON subzone.count_layout_subzone_key = user.count_layout_subzone_key
WHERE user.count_layout_subzone_key IN (?, ?, ?, ?, ?, ?, ?, ?, ?, ?)  -- 10 subzone keys
  AND (? IS NULL OR user.user_id = ?);  -- Optional filter
```

**Merging Strategy:**
```java
assignedUserEntities.forEach(assignedUser -> {
    String parentSubzoneKey = assignedUser.getCountLayoutSubzoneEntity().getCountLayoutSubzoneKey();

    // Get or create subzone in enriched map
    CountLayoutSubzoneEntity subzone = enrichedSubzoneMap.computeIfAbsent(
        parentSubzoneKey,
        key -> {
            CountLayoutSubzoneEntity entity = assignedUser.getCountLayoutSubzoneEntity();
            entity.setCountAssignedUserEntities(new HashSet<>());  // Initialize to avoid lazy load
            entity.setCountLayoutLocationEntities(new HashSet<>());
            return entity;
        }
    );

    // Add assigned user to subzone's collection
    subzone.getCountAssignedUserEntities().add(assignedUser);
});
```

#### Query 4: Locations (if requested)
```sql
SELECT DISTINCT location
FROM count_layout_location location
INNER JOIN count_layout_subzone subzone ON subzone.count_layout_subzone_key = location.count_layout_subzone_key
WHERE location.count_layout_subzone_key IN (?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
  AND (? IS NULL OR location.layout_name = ?)
  AND (? IS NULL OR location.product_id = ?);
```

#### Query 5: Perform Actions (if requested)
```sql
SELECT DISTINCT action
FROM count_perform_action action
INNER JOIN count_layout_location location ON location.count_layout_location_key = action.count_layout_location_key
INNER JOIN count_layout_subzone subzone ON subzone.count_layout_subzone_key = location.count_layout_subzone_key
WHERE (? IS NULL OR action.count_assigned_user_key IN (:userKeys))
  AND (? IS NULL OR action.count_layout_location_key IN (:locationKeys));
```

---

## 9. Complete Example Walkthrough {#complete-example}

### Scenario
- **Database**: 47 subzones match criteria
- **Request**: Page 2, size 10, sort by userName ASC
- **Associations**: COUNT_ASSIGNED_USER, COUNT_LAYOUT_LOCATION
- **Filters**: taskId=TASK123, parentLayoutDistinctName=SZ=FER1

### Step-by-Step Execution

#### Step 1: Query 1 - Get Sorted Keys

```sql
SELECT
  subzone.count_layout_subzone_key,
  subzone.parent_layout_name,
  MIN(user.first_name || ' ' || user.last_name) as user_name
FROM count_layout_subzone subzone
LEFT JOIN count_assigned_user user ON user.count_layout_subzone_key = subzone.count_layout_subzone_key
WHERE subzone.count_request_key = 'hash_of_AZUS_DC1_TASK123'
  AND subzone.parent_layout_distinct_name = 'SZ=FER1'
GROUP BY subzone.count_layout_subzone_key, subzone.parent_layout_name
ORDER BY user_name ASC
LIMIT 10 OFFSET 10;
```

**Result** (page 2):
| subzone_key | parent_layout_name | user_name |
|-------------|-------------------|-----------|
| SZ011 | Subzone-K | Alice Brown |
| SZ012 | Subzone-L | Alice Cooper |
| SZ013 | Subzone-M | Bob Anderson |
| SZ014 | Subzone-N | Bob Johnson |
| SZ015 | Subzone-O | Charlie Davis |
| SZ016 | Subzone-P | Charlie Wilson |
| SZ017 | Subzone-Q | David Clark |
| SZ018 | Subzone-R | David Moore |
| SZ019 | Subzone-S | Eve Martinez |
| SZ020 | Subzone-T | Eve Taylor |

**Pagination Metadata**:
- Total elements: 47
- Total pages: 5 (47 / 10 = 4.7, rounded up)
- Current page: 2
- Page size: 10

#### Step 2: Extract Keys

```java
List<String> pageSubzoneKeys = projectionPage.getContent().stream()
    .map(SubzoneSortProjection::getSubzoneKey)
    .distinct()  // Remove any duplicates from JOINs
    .toList();

// Result: [SZ011, SZ012, SZ013, SZ014, SZ015, SZ016, SZ017, SZ018, SZ019, SZ020]
```

#### Step 3: Query 2 - Fetch Full Entities

```sql
SELECT DISTINCT
    subzone.*,
    count_request.*
FROM count_layout_subzone subzone
INNER JOIN count_request count_request ON count_request.count_request_key = subzone.count_request_key
WHERE subzone.count_layout_subzone_key IN (
    'SZ011', 'SZ012', 'SZ013', 'SZ014', 'SZ015',
    'SZ016', 'SZ017', 'SZ018', 'SZ019', 'SZ020'
);
```

**Result**: 10 `CountLayoutSubzoneEntity` objects with `countRequestEntity` populated

#### Step 4: Reorder Entities

Database might return entities in different order than sorted keys:

**Query 2 result order**: [SZ012, SZ011, SZ015, SZ013, ...]

**Reorder to match Query 1**: [SZ011, SZ012, SZ013, SZ014, SZ015, ...]

```java
Map<String, CountLayoutSubzoneEntity> entityMap = new HashMap<>();
for (CountLayoutSubzoneEntity entity : subzoneEntities) {
    entityMap.put(entity.getCountLayoutSubzoneKey(), entity);
}

List<CountLayoutSubzoneEntity> orderedEntities = new ArrayList<>();
for (String key : pageSubzoneKeys) {  // Iterate in sorted order
    CountLayoutSubzoneEntity entity = entityMap.get(key);
    if (entity != null) {
        orderedEntities.add(entity);
    }
}
```

#### Step 5: Query 3 - Fetch Assigned Users

```sql
SELECT DISTINCT user.*
FROM count_assigned_user user
WHERE user.count_layout_subzone_key IN (
    'SZ011', 'SZ012', 'SZ013', 'SZ014', 'SZ015',
    'SZ016', 'SZ017', 'SZ018', 'SZ019', 'SZ020'
);
```

**Result**: 23 assigned users across the 10 subzones

**Data**:
- SZ011: 3 users (Alice Brown, John Doe, Jane Smith)
- SZ012: 2 users (Alice Cooper, Bob Wilson)
- SZ013: 2 users (Bob Anderson, Charlie Lee)
- ... (continues for all 10 subzones)

**Merge into enriched map**:
```java
enrichedSubzoneMap:
  SZ011 -> SubzoneEntity { users: [Alice, John, Jane], locations: [] }
  SZ012 -> SubzoneEntity { users: [Alice, Bob], locations: [] }
  ...
```

#### Step 6: Query 4 - Fetch Locations

```sql
SELECT DISTINCT location.*
FROM count_layout_location location
WHERE location.count_layout_subzone_key IN (
    'SZ011', 'SZ012', 'SZ013', 'SZ014', 'SZ015',
    'SZ016', 'SZ017', 'SZ018', 'SZ019', 'SZ020'
);
```

**Result**: 145 locations across the 10 subzones

**Merge into enriched map**:
```java
enrichedSubzoneMap:
  SZ011 -> SubzoneEntity { users: [Alice, John, Jane], locations: [Slot-A1, Slot-A2, ...] }
  SZ012 -> SubzoneEntity { users: [Alice, Bob], locations: [Slot-B1, Slot-B2, ...] }
  ...
```

#### Step 7: Map to DTOs

```java
Page<CountLayoutSubzone> dtoPage = subzonePage.map(baseEntity -> {
    // Get enriched version with child entities
    CountLayoutSubzoneEntity enrichedEntity = enrichedSubzoneMap.getOrDefault(
        baseEntity.getCountLayoutSubzoneKey(),
        baseEntity
    );

    // Map to DTO
    CountLayoutSubzone dto = mapper.toCountLayoutSubzone(enrichedEntity);

    // Include assigned users (requested)
    if (!enrichedEntity.getCountAssignedUserEntities().isEmpty()) {
        List<AssignedUserDto> userDtos = enrichedEntity.getCountAssignedUserEntities().stream()
            .map(mapper::toAssignedUserDto)
            .toList();
        dto.setAssignedUsers(userDtos);
    }

    // Include locations (requested)
    if (!enrichedEntity.getCountLayoutLocationEntities().isEmpty()) {
        List<LayoutLocation> locationDtos = enrichedEntity.getCountLayoutLocationEntities().stream()
            .map(mapper::toLayoutLocation)
            .toList();
        dto.setLocations(locationDtos);
    }

    return dto;
});
```

#### Step 8: Final Response

```json
{
  "content": [
    {
      "countLayoutSubzoneKey": "SZ011",
      "parentLayoutName": "Subzone-K",
      "totalInvQtyExpected": 5000,
      "assignedUsers": [
        { "userId": "alice.brown", "firstName": "Alice", "lastName": "Brown" },
        { "userId": "john.doe", "firstName": "John", "lastName": "Doe" },
        { "userId": "jane.smith", "firstName": "Jane", "lastName": "Smith" }
      ],
      "locations": [
        { "layoutName": "Slot-A1", "productId": "12345" },
        { "layoutName": "Slot-A2", "productId": "12346" }
      ]
    },
    // ... 9 more subzones
  ],
  "pageable": {
    "pageNumber": 1,
    "pageSize": 10,
    "sort": { "sorted": true }
  },
  "totalElements": 47,
  "totalPages": 5,
  "first": false,
  "last": false,
  "numberOfElements": 10
}
```

### Total Queries: 4
1. Query 1: Get sorted keys (projection)
2. Query 2: Fetch subzone entities
3. Query 3: Fetch assigned users
4. Query 4: Fetch locations

### Performance Metrics
- **Response time**: ~150ms
- **Rows transferred**: 10 + 10 + 23 + 145 = **188 rows**
- **Memory usage**: Minimal (only page's data)
- **Accuracy**: ✅ 100% correct pagination and sorting

---

## 10. Best Practices & Patterns {#best-practices}

### ✅ DO: Use Projections for Sorting

```java
public interface SubzoneSortProjection {
    String getSubzoneKey();
    String getSortField();
}

@Query("""
    SELECT s.id as subzoneKey, u.firstName as sortField
    FROM Subzone s
    JOIN s.users u
    GROUP BY s.id
    ORDER BY sortField
""")
Page<SubzoneSortProjection> findSortedKeys(Pageable pageable);
```

### ✅ DO: Batch Fetch with IN Clause

```java
@Query("SELECT DISTINCT u FROM User u WHERE u.subzone.id IN :ids")
List<User> findBySubzoneIds(@Param("ids") List<String> ids);
```

**Why**: Single query instead of N queries

### ✅ DO: Initialize Collections to Avoid Lazy Loading

```java
entity.setCountAssignedUserEntities(new HashSet<>());  // Prevent lazy load
entity.setCountLayoutLocationEntities(new HashSet<>());
```

### ✅ DO: Use @Transactional for Lazy Collections

```java
@Transactional(readOnly = true)
public List<SubzoneDto> getSubzones() {
    List<SubzoneEntity> entities = repository.findAll();
    return entities.stream()
        .map(entity -> {
            // Can safely access lazy collections inside transaction
            Set<User> users = entity.getUsers();
            return mapToDto(entity, users);
        })
        .toList();
}
```

### ✅ DO: Use DISTINCT with JOIN FETCH

```java
@Query("SELECT DISTINCT s FROM Subzone s JOIN FETCH s.users")
List<Subzone> findWithUsers();
```

**Why**: Removes duplicate parent entities

### ❌ DON'T: JOIN FETCH Multiple Collections

```java
// ❌ BAD - Cartesian product!
@Query("""
    SELECT DISTINCT s
    FROM Subzone s
    JOIN FETCH s.users
    JOIN FETCH s.locations
""")
List<Subzone> findAll();
```

**Solution**: Use separate queries

### ❌ DON'T: Use EAGER Fetch Everywhere

```java
// ❌ BAD - Always loads users, even when not needed
@OneToMany(fetch = FetchType.EAGER)
private Set<User> users;
```

**Why**: Performance penalty on ALL queries

### ❌ DON'T: Paginate with JOIN FETCH

```java
// ❌ BAD - In-memory pagination!
@Query("SELECT DISTINCT s FROM Subzone s JOIN FETCH s.users")
Page<Subzone> findAll(Pageable pageable);
```

**Solution**: Two-query approach

### ✅ DO: Use EntityGraph for Selective Eager Loading

```java
@EntityGraph(attributePaths = {"users", "countRequest"})
@Query("SELECT s FROM Subzone s WHERE s.id IN :ids")
List<Subzone> findByIdsWithUsers(@Param("ids") List<String> ids);
```

**Equivalent to**: JOIN FETCH, but more flexible

### ✅ DO: Use Hibernate Statistics for Debugging

```yaml
# application.properties
spring.jpa.properties.hibernate.generate_statistics=true
logging.level.org.hibernate.stat=DEBUG
```

**Output**:
```
Hibernate: Session Metrics {
    15 nanoseconds spent acquiring 1 JDBC connections;
    0 nanoseconds spent releasing 0 JDBC connections;
    42 nanoseconds spent preparing 4 JDBC statements;
    5000 nanoseconds spent executing 4 JDBC statements;
    0 nanoseconds spent executing 0 JDBC batches;
}
```

### ✅ DO: Profile with SQL Logging

```yaml
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
logging.level.org.hibernate.type.descriptor.sql.BasicBinder=TRACE
```

**Output**:
```sql
Hibernate:
    select
        subzone0_.subzone_key as subzone_key1_0_,
        subzone0_.parent_layout_name as parent_layout_name2_0_
    from
        count_layout_subzone subzone0_
    where
        subzone0_.count_request_key=?
2024-03-12 10:15:23.456 TRACE 12345 --- [nio-8080-exec-1] o.h.type.descriptor.sql.BasicBinder      : binding parameter [1] as [VARCHAR] - [hash_value]
```

---

## 📊 Performance Comparison

### Scenario: Fetch 10 subzones with users and locations

| Approach | Queries | Rows Transferred | Memory | Pagination Correct? |
|----------|---------|------------------|--------|---------------------|
| **Naive (Lazy)** | 21 | 10 + 10×2 + 10×15 = 180 | Low | ✅ |
| **JOIN FETCH (1 collection)** | 1 | 10 + 20 duplicates = 30 | Low | ✅ |
| **JOIN FETCH (2 collections)** | 1 | 10×2×15 = 300 | High | ❌ In-memory! |
| **Two-Query Approach** | 4 | 10 + 10 + 20 + 150 = 190 | Low | ✅ |
| **Two-Query + Projection** | 4 | 50 + 10 + 20 + 150 = 230 | Low | ✅ |

**Winner**: Two-Query Approach with Projection
- ✅ Minimal queries (4)
- ✅ Correct pagination
- ✅ Efficient memory usage
- ✅ No Cartesian product
- ✅ Client controls associations

---

## 🎓 Key Takeaways

1. **N+1 Problem**: Use JOIN FETCH or batch fetching to avoid multiple queries
2. **Cartesian Product**: Never JOIN FETCH multiple collections in one query
3. **Pagination**: Two-query approach for paginating with child entity sorting
4. **Filtering Confusion**: WHERE clauses on child fields filter BOTH parents AND children, creating incomplete collections
5. **Lazy vs Eager**: Default to LAZY, use JOIN FETCH selectively
6. **Projections**: Fetch only needed columns for sorting/filtering
7. **Conditional Loading**: Let client control which associations to fetch
8. **Collections Initialization**: Initialize collections to avoid lazy loading exceptions
9. **Transactions**: Access lazy collections inside @Transactional methods
10. **Profiling**: Always profile with SQL logging and Hibernate statistics
11. **Multi-Query is OK**: 4-5 targeted queries better than 1 massive query or N+1 queries
12. **Separate Filtering from Loading**: Use EXISTS subqueries or separate queries to filter parents without affecting child collections

---

## 🔗 References

- [Hibernate Documentation](https://hibernate.org/orm/documentation/)
- [Vlad Mihalcea's Blog](https://vladmihalcea.com/) - JPA/Hibernate expert
- [JPA Specification](https://jakarta.ee/specifications/persistence/)
- [Spring Data JPA Reference](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/)

---

**Author**: Deep Dive Analysis based on Shalinee Chauhan's Implementation
**Date**: 2026-03-12
**Version**: 2.0 (Added comprehensive filtering section)

