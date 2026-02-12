---
name: perspective-resolver
description: Internal skill for resolving or creating Caves perspectives. Searches for existing perspectives by identifying tags, prompts user for reuse vs creation, and sets up initial connections. Used by other skills that need perspective management.
---

# Perspective Resolver

Internal skill for managing Caves perspectives. Each perspective represents an entity (cookbook, article, person, etc.) and is identified by key tags that the pId is a child of.

## Core Concept

A perspective **IS** an entity. The pId should be a **child** of its key identifiers:
- **Cookbook**: `{book title} ← {pId}` and `{author} ← {pId}`
- **Article**: `{author} ← {pId}` and `{url} ← {pId}`
- **Other entities**: Define identifying tags that uniquely describe what this perspective represents

These connections define what the perspective represents. To find if it already exists, search for a pId that's already a child of those exact identifying tags.

## Input Parameters

```javascript
{
  // Primary identifier used as perspective username (required)
  primary_tag: "salt fat acid heat",

  // Tags that uniquely identify this entity (required)
  // The pId will be created as a child of ALL these tags
  identifying_tags: ["salt fat acid heat", "samin nosrat"],

  // Optional: Pairs of tags to connect bidirectionally
  // Format: [[parent, child], ...] creates both parent←child and child←parent
  bidirectional_pairs: [
    ["salt fat acid heat", "samin nosrat"]
  ],

  // Optional: Taxonomy and other structural connections
  // Use "{pId}" as placeholder for the resolved/created perspective ID
  taxonomy: [
    {parent: "books", child: "cookbooks"},
    {parent: "cookbooks", child: "salt fat acid heat"},
    {parent: "indexes", child: "salt fat acid heat"},
    {parent: "thing", child: "{pId}"}
  ],

  // Optional: Context description for user communication
  entity_type: "cookbook",  // e.g., "cookbook", "article", "transcript"
  entity_description: "salt fat acid heat by samin nosrat"
}
```

## Workflow

### Step 1: Search for Existing Perspective

Use `caves__getPidsForTags` to find perspectives that are children of ALL identifying tags:

```javascript
caves__getPidsForTags(
  tags: identifying_tags,
  matchMode: 'all'
)
```

**matchMode='all'** ensures the pId is a child of ALL identifying tags (intersection), which means it represents exactly this entity.

### Step 2: Handle Search Results

**If matching pId(s) found:**

1. Fetch the perspective details:
   ```javascript
   caves__my_shadow_caves(pId: found_pId, format: 'md')
   ```

2. Present to user:
   ```
   Found existing perspective for {entity_description}:
   - Username: {username}
   - Has {X} existing connections

   Would you like to:
   a) Use this existing perspective
   b) Create a new separate perspective
   ```

3. If user chooses (a):
   - Return existing pId
   - Skip connection creation (already exists)

4. If user chooses (b):
   - Suggest distinguishing suffix for primary_tag
   - Example: "salt fat acid heat (2nd edition)"
   - Proceed to Step 3 with modified primary_tag

**If no matching pId found:**

1. Confirm with user:
   ```
   No existing perspective found for {entity_description}.
   Creating new perspective with username: {primary_tag}

   Proceed?
   ```

2. Proceed to Step 3

### Step 3: Create New Perspective

```javascript
result = caves__createPerspective(username: primary_tag)
pId = result.pId
```

### Step 4: Create Initial Connections

Build connection array in this order:

1. **Bidirectional connections** (if provided):
   ```javascript
   // For each pair [tag1, tag2]:
   {parent: tag1, child: tag2, value: true}
   {parent: tag2, child: tag1, value: true}
   ```

2. **Identifying connections** (pId as child of identifying tags):
   ```javascript
   // For each identifying_tag:
   {parent: identifying_tag, child: pId, value: true}
   ```

3. **Taxonomy connections** (if provided):
   ```javascript
   // Replace "{pId}" placeholder with actual pId
   // Then add each connection
   {parent: parent_tag, child: child_tag, value: true}
   ```

Execute batch connection:
```javascript
caves__connect_caves(
  pId: pId,
  connections: all_connections
)
```

### Step 5: Return Result

```javascript
{
  pId: "0x...",
  username: "salt fat acid heat",
  is_new: true,  // or false if reused existing
  connections_created: 12
}
```

## Usage Examples

### Example 1: Cookbook

```javascript
perspective_resolver({
  primary_tag: "salt fat acid heat",
  identifying_tags: ["salt fat acid heat", "samin nosrat"],
  bidirectional_pairs: [
    ["salt fat acid heat", "samin nosrat"]
  ],
  taxonomy: [
    {parent: "books", child: "cookbooks"},
    {parent: "cookbooks", child: "salt fat acid heat"},
    {parent: "indexes", child: "salt fat acid heat"},
    {parent: "thing", child: "{pId}"}
  ],
  entity_type: "cookbook",
  entity_description: "salt fat acid heat by samin nosrat"
})
```

Result: pId that represents this cookbook, with:
- Bidirectional: `salt fat acid heat ↔ samin nosrat`
- Identifying: `salt fat acid heat ← pId`, `samin nosrat ← pId`
- Taxonomy: `books ← cookbooks ← salt fat acid heat`, `indexes ← salt fat acid heat`, `thing ← pId`

### Example 2: Article/Transcript

```javascript
perspective_resolver({
  primary_tag: "david graeber",
  identifying_tags: ["david graeber", "https://theanarchistlibrary.org/article"],
  taxonomy: [
    {parent: "articles", child: "https://theanarchistlibrary.org/article"},
    {parent: "thing", child: "{pId}"}
  ],
  entity_type: "article",
  entity_description: "article by david graeber"
})
```

Result: pId that represents this article, with:
- Identifying: `david graeber ← pId`, `https://theanarchistlibrary.org/article ← pId`
- Taxonomy: `articles ← https://theanarchistlibrary.org/article`, `thing ← pId`

### Example 3: Minimal (Just Entity Identification)

```javascript
perspective_resolver({
  primary_tag: "my research notes",
  identifying_tags: ["my research notes", "economics"],
  entity_type: "research collection",
  entity_description: "economics research notes"
})
```

Result: pId that represents this collection, with:
- Identifying: `my research notes ← pId`, `economics ← pId`
- No additional taxonomy

## Important Notes

### Tag Formatting
- **CRITICAL**: All tags must be lowercase EXCEPT perspective IDs (0x...)
- Primary tag should be descriptive and unique
- Identifying tags should be the minimal set that uniquely identifies this entity

### When to Reuse vs Create New
- **Reuse**: Same entity (same book, same article, same person)
- **Create new**: Different editions, different contexts, user preference

### Connection Order Matters
The order in Step 4 ensures:
1. Bidirectional connections establish relationships between identifying entities
2. Identifying connections define what the pId represents
3. Taxonomy connections organize the entity in the broader knowledge graph

### Error Handling

**caves__getPidsForTags fails:**
- May mean no existing perspectives (proceed to create new)
- Or API error (report to user)

**caves__createPerspective fails:**
- Username may be taken (suggest alternative)
- User may lack permissions (report error)

**caves__connect_caves fails:**
- May be partial success (some connections created)
- Report which connections failed
- Return pId anyway (it was created successfully)

## Best Practices

1. **Identifying Tags**: Choose tags that together uniquely identify the entity
   - Too few: May match unrelated perspectives
   - Too many: May miss valid matches
   - Sweet spot: 2-3 core identifiers

2. **Primary Tag**: Should be the most human-readable identifier
   - Cookbook: Book title
   - Article: Author name or article title
   - Collection: Collection name

3. **Taxonomy**: Use to organize entities in your knowledge structure
   - Not required for basic functionality
   - Helpful for browsing and discovery
   - Consider standard parents like "thing", "books", "articles", etc.

4. **User Communication**: Always explain what you're doing
   - Show what was found or will be created
   - Give users choice when existing perspective found
   - Confirm before creating new perspectives

## Integration with Other Skills

Skills that need perspective management should:

1. **Prepare parameters** based on their entity type
2. **Invoke this skill** (or call the logic directly)
3. **Receive pId** and connection metadata
4. **Continue** with their specific work using the resolved pId

This skill does NOT:
- Create content-specific connections (recipes, index entries, article tags, etc.)
- Upload content to IPFS
- Analyze content

Those are the responsibility of the calling skill.
