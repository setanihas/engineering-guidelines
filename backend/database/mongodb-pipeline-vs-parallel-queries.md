# MongoDB Performance Comparison: Aggregation Pipeline vs Parallel Queries

## Introduction

When fetching a document with multiple related collections in MongoDB, there are two common approaches:

1. **Aggregation Pipeline with `$lookup`** - Join data at the database level in a single query
2. **Parallel Queries with `Promise.all()`** - Fetch related data separately and combine in application code

This document presents our findings from performance testing both approaches to help you make informed decisions for your use cases.

---

## The Problem

We needed to fetch a resource document along with its related data:
- **Direct relations:** pages, owner, members, teams, organization, modals
- **Nested relations:** member role assignments, team role actions, parent pages, variant rules
- **Deeply nested relations:** member roles, team roles

This creates a complex data structure with 3+ levels of nesting.

---

## Approaches Tested

### Approach 1: Aggregation Pipeline

Using MongoDB's `$lookup` operator to join collections at the database level:

```javascript
async function getResourceWithPipeline(id) {
    const pipeline = [
        { $match: { _id: ObjectId(id) } },
        
        // Direct lookups
        { $lookup: { from: 'modals', localField: 'modals', foreignField: '_id', as: 'modals' } },
        { $lookup: { from: 'users', localField: 'owner', foreignField: '_id', as: 'owner' } },
        { $unwind: { path: '$owner', preserveNullAndEmptyArrays: true } },
        { $lookup: { from: 'organizations', localField: 'organization', foreignField: '_id', as: 'organization' } },
        { $unwind: { path: '$organization', preserveNullAndEmptyArrays: true } },
        
        // Nested pipeline lookups
        {
            $lookup: {
                from: 'pages',
                let: { pageIds: '$pages' },
                pipeline: [
                    { $match: { $expr: { $in: ['$_id', '$$pageIds'] } } },
                    {
                        $lookup: {
                            from: 'pages',
                            localField: 'parent_page',
                            foreignField: '_id',
                            as: 'parent_page'
                        }
                    },
                    { $unwind: { path: '$parent_page', preserveNullAndEmptyArrays: true } },
                    {
                        $lookup: {
                            from: 'variant_rules',
                            localField: 'variant_rules',
                            foreignField: '_id',
                            as: 'variant_rules'
                        }
                    }
                ],
                as: 'pages'
            }
        },
        
        // Members with nested role lookups
        {
            $lookup: {
                from: 'users',
                let: { memberIds: '$members' },
                pipeline: [
                    { $match: { $expr: { $in: ['$_id', '$$memberIds'] } } },
                    {
                        $lookup: {
                            from: 'role_assignments',
                            localField: 'role_assignment',
                            foreignField: '_id',
                            as: 'role_assignment',
                            pipeline: [
                                {
                                    $lookup: {
                                        from: 'role_actions',
                                        localField: 'role_actions',
                                        foreignField: '_id',
                                        as: 'role_actions',
                                        pipeline: [
                                            {
                                                $lookup: {
                                                    from: 'roles',
                                                    localField: 'role',
                                                    foreignField: '_id',
                                                    as: 'role'
                                                }
                                            },
                                            { $unwind: { path: '$role', preserveNullAndEmptyArrays: true } }
                                        ]
                                    }
                                }
                            ]
                        }
                    }
                ],
                as: 'members'
            }
        },
        
        // Teams with nested role lookups (similar structure)
        {
            $lookup: {
                from: 'teams',
                let: { teamIds: '$teams' },
                pipeline: [
                    { $match: { $expr: { $in: ['$_id', '$$teamIds'] } } },
                    {
                        $lookup: {
                            from: 'role_actions',
                            localField: 'role_actions',
                            foreignField: '_id',
                            as: 'role_actions',
                            pipeline: [
                                {
                                    $lookup: {
                                        from: 'roles',
                                        localField: 'role',
                                        foreignField: '_id',
                                        as: 'role'
                                    }
                                },
                                { $unwind: { path: '$role', preserveNullAndEmptyArrays: true } }
                            ]
                        }
                    }
                ],
                as: 'teams'
            }
        }
    ];

    const [resource] = await db.collection('resources').aggregate(pipeline).toArray();
    return resource;
}
```

### Approach 2: Parallel Queries

Fetching data in stages using `Promise.all()` for parallel execution:

```javascript
async function getResourceWithParallelQueries(id) {
    // Step 1: Fetch main resource
    const resource = await db.collection('resources').findOne({ _id: ObjectId(id) });
    
    if (!resource) return null;

    // Step 2: Fetch all direct relations in parallel
    const [pages, owner, members, teams, organization, modals] = await Promise.all([
        db.collection('pages').find({ 
            _id: { $in: resource.pages.map(id => ObjectId(id)) } 
        }).toArray(),
        
        db.collection('users').findOne({ _id: ObjectId(resource.owner) }),
        
        db.collection('users').find({ 
            _id: { $in: (resource.members || []).map(id => ObjectId(id)) } 
        }).toArray(),
        
        db.collection('teams').find({ 
            _id: { $in: (resource.teams || []).map(id => ObjectId(id)) } 
        }).toArray(),
        
        db.collection('organizations').findOne({ _id: ObjectId(resource.organization) }),
        
        db.collection('modals').find({ 
            _id: { $in: (resource.modals || []).map(id => ObjectId(id)) } 
        }).toArray(),
    ]);

    // Step 3: Fetch nested relations in parallel
    const [memberRoleAssignments, teamRoleActions, parentPages, variantRules] = await Promise.all([
        members.length > 0
            ? db.collection('role_assignments').find({
                _id: { $in: members.flatMap(m => m.role_assignment || []).map(id => ObjectId(id)) }
            }).toArray()
            : [],

        teams.length > 0
            ? db.collection('role_actions').find({
                _id: { $in: teams.flatMap(t => t.role_actions || []).map(id => ObjectId(id)) }
            }).toArray()
            : [],

        pages.length > 0
            ? db.collection('pages').find({
                _id: { $in: pages.map(p => p.parent_page).filter(Boolean).map(id => ObjectId(id)) }
            }).toArray()
            : [],

        pages.length > 0
            ? db.collection('variant_rules').find({
                _id: { $in: pages.flatMap(p => p.variant_rules || []).map(id => ObjectId(id)) }
            }).toArray()
            : [],
    ]);

    // Step 4: Fetch deeply nested relations in parallel
    const [memberRoleActions, teamRoles] = await Promise.all([
        memberRoleAssignments.length > 0
            ? db.collection('role_actions').find({
                _id: { $in: memberRoleAssignments.flatMap(ra => ra.role_actions || []).map(id => ObjectId(id)) }
            }).toArray()
            : [],

        teamRoleActions.length > 0
            ? db.collection('roles').find({
                _id: { $in: teamRoleActions.map(ra => ra.role).filter(Boolean).map(id => ObjectId(id)) }
            }).toArray()
            : [],
    ]);

    // Step 5: Fetch final nested relation
    const memberRoles = memberRoleActions.length > 0
        ? await db.collection('roles').find({
            _id: { $in: memberRoleActions.map(rac => rac.role).filter(Boolean).map(id => ObjectId(id)) }
        }).toArray()
        : [];

    // Step 6: Build and return response object
    return {
        ...resource,
        modals,
        pages: pages.map(page => ({
            ...page,
            parent_page: parentPages.find(pp => pp._id.equals(page.parent_page)) || null,
            variant_rules: variantRules.filter(vr =>
                (page.variant_rules || []).some(vrId => vr._id.equals(vrId))
            ),
        })),
        owner,
        members: members.map(member => ({
            ...member,
            role_assignment: memberRoleAssignments
                .filter(ra => (member.role_assignment || []).some(raId => ra._id.equals(raId)))
                .map(ra => ({
                    ...ra,
                    role_actions: memberRoleActions
                        .filter(rac => (ra.role_actions || []).some(racId => rac._id.equals(racId)))
                        .map(rac => ({
                            ...rac,
                            role: memberRoles.find(r => r._id.equals(rac.role)) || null,
                        })),
                })),
        })),
        teams: teams.map(team => ({
            ...team,
            role_actions: teamRoleActions
                .filter(ra => (team.role_actions || []).some(raId => ra._id.equals(raId)))
                .map(ra => ({
                    ...ra,
                    role: teamRoles.find(r => r._id.equals(ra.role)) || null,
                })),
        })),
        organization,
    };
}
```

---

## Test Results

### Aggregation Pipeline Performance

| Metric | Time |
|--------|------|
| Aggregate Execution | 86 - 297ms |
| **Total** | **133 - 907ms** |

### Parallel Queries Performance

| Metric | Time |
|--------|------|
| Fetch Resource | 5 - 53ms |
| Fetch Direct Relations | 8 - 95ms |
| Fetch Nested Relations | 4 - 62ms |
| Build Response | 0.3 - 1.6ms |
| **Total** | **24 - 226ms** |

### Overall Comparison

| Metric | Pipeline | Parallel Queries | Improvement |
|--------|----------|------------------|-------------|
| **Average** | ~317ms | ~75ms | **4.2x faster** |
| **Best Case** | ~133ms | ~24ms | **5.5x faster** |
| **Worst Case** | ~907ms | ~226ms | **4x faster** |

---

## Analysis

### Why is the Pipeline approach slower?

1. **Nested `$lookup` Complexity**
   - Each nested `$lookup` adds computational overhead
   - MongoDB executes nested pipelines sequentially within each document
   - With 3+ levels of nesting, this compounds significantly

2. **Memory Usage**
   - Complex pipelines require more working memory
   - Large intermediate result sets are held in memory during processing

3. **Index Utilization**
   - Nested `$lookup` pipelines may not utilize indexes as efficiently
   - Separate queries can better leverage indexed fields

### Why are Parallel Queries faster?

1. **True Parallelism**
   - `Promise.all()` executes queries concurrently
   - Network round-trips overlap instead of being sequential

2. **Simpler Queries**
   - Each query is a simple `find()` operation
   - MongoDB can optimize these more effectively

3. **Conditional Execution**
   - Skip unnecessary queries when arrays are empty
   - No wasted computation on non-existent relations

---

## Pros & Cons Summary

### Aggregation Pipeline

| Pros | Cons |
|------|------|
| Single database round-trip | Complex nested `$lookup` adds overhead |
| Atomic operation | Hard to debug and maintain |
| All processing on database server | High variability in response times |
| Less data transfer for filtered results | Poor performance with deep nesting |

### Parallel Queries

| Pros | Cons |
|------|------|
| Significantly faster in practice | Multiple database round-trips |
| Predictable response times | More network requests |
| Easier to read, debug, and maintain | Response assembly in application code |
| Better error isolation | Requires careful null/empty handling |
| Conditional query execution | Not atomic (potential consistency issues) |
| Scales well with proper indexing | |

---

## Recommendations

### Use Parallel Queries when:
- You have complex nested relations (3+ levels deep)
- Individual collections have proper indexes
- You need conditional fetching based on data presence
- Code maintainability is a priority
- Response time predictability is important

### Consider Aggregation Pipeline when:
- You have simple joins (1-2 `$lookup` stages without nesting)
- You need to filter large datasets before joining
- Atomic read consistency is required
- You're performing aggregations (grouping, counting, etc.)
- Data reduction happens early in the pipeline

---

## Implementation Tips

### For Parallel Queries:

1. **Always handle empty arrays:**
   ```javascript
   const results = ids.length > 0 
       ? await collection.find({ _id: { $in: ids } }).toArray()
       : [];
   ```

2. **Use proper indexing:**
   Ensure `_id` fields and frequently queried fields are indexed.

3. **Consider connection pooling:**
   Multiple parallel queries benefit from a properly configured connection pool.

4. **Add error handling:**
   ```javascript
   const [result1, result2] = await Promise.all([
       query1().catch(err => { console.error(err); return []; }),
       query2().catch(err => { console.error(err); return null; }),
   ]);
   ```

### For Aggregation Pipelines:

1. **Place `$match` early:**
   Filter documents as early as possible to reduce processing.

2. **Limit nested depth:**
   Avoid more than 2 levels of nested `$lookup` when possible.

3. **Use `allowDiskUse` for large datasets:**
   ```javascript
   collection.aggregate(pipeline, { allowDiskUse: true })
   ```

---

## Conclusion

For our use case involving complex nested relations, **parallel queries with `Promise.all()` outperformed aggregation pipelines by 4-5x**. While aggregation pipelines have their place, they become a performance bottleneck when dealing with deeply nested `$lookup` operations.

We recommend evaluating both approaches for your specific use case, but lean towards parallel queries when dealing with complex document relationships.

---

*Last updated: December 2025*
