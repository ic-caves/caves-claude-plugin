---
name: cookbook-query
description: Query and search across cookbook indexes stored in the Caves knowledge graph. Use when the user wants to find recipes, ingredients, or page references across their cookbook collection. Handles resolving cookbook ownership, mapping PIDs to cookbook names, and searching for ingredients/recipes across multiple cookbooks. Also use when the user asks things like "what cookbooks have chicken recipes" or "find me a recipe for X" or references their cookbook collection.
---

# Cookbook Query

Search and retrieve recipe/ingredient information across cookbook indexes stored in Caves.

## Data Model

Each cookbook is a **perspective (shadow/PID)** in Caves. The connections within each perspective represent the cookbook's **index** — structured hierarchically:

```
book title (perspective username)
  ├── author (bidirectional connection)
  ├── entry (ingredient/recipe name/category)
  │   ├── page number (if entry has a page)
  │   └── child entries (if entry has sub-items)
  │       └── page numbers for children
  └── direct recipe connections (any entry with a page number)
      └── page number
```

**Key structural points:**
- Entries connect directly under the book title (no letter divisions)
- Recipe/section entries are connected directly to book title for easy access
- Page numbers are ONLY connected under their immediate parent entry (never to book title)
- Bidirectional author connection links the book to its author

The PID itself maps to the cookbook's name and author (visible as the perspective's username).

Page number connections are perspective-specific: under a shared ingredient like "chicken," each cookbook PID connects its own page numbers. One book connects chicken → 42, another connects chicken → 187.

## Cookbook Ownership

Users can own cookbooks in TWO ways:

1. **Ownership PID (tracked ownership)**: A dedicated perspective that tracks which cookbooks a user owns via connections like `cookbooks ← "salt fat acid heat"`. The cookbook itself is a separate perspective (with its own PID).

2. **Cookbook Perspectives (direct ownership)**: Perspectives that ARE cookbooks themselves, created via the cookbook-index-to-caves skill. These can be identified by:
   - Having standard taxonomy connections: `cookbooks ← {book title}`
   - Having bidirectional author connections
   - Username matching a book title

Both methods should be checked during cookbook discovery to find ALL cookbooks a user owns.

## Setup & Onboarding

Before querying, the user needs an ownership perspective and at least one cookbook in the system.

### Check for Existing Setup

1. Run `caves__my_shadows()` to list the user's perspectives
2. Look for a perspective that serves as the ownership PID (likely connected to a "cookbooks" cave)
3. Check `caves__get_subcaves(parentCave="cookbooks")` to see what cookbooks already exist in the system

### If No Ownership Perspective Exists

1. Create one: `caves__createPerspective(username="[user's name] cookbooks")`
2. Search the "cookbooks" cave to see if any cookbook PIDs already exist in the system: `caves__get_subcaves(parentCave="cookbooks")`
3. Present any existing cookbook titles to the user and ask which ones they own
4. For each owned cookbook, connect it under their ownership PID:
   ```python
   caves__connect_caves(pId="ownership-pid", connections=[
       {"parent": "cookbooks", "child": "[cookbook-pid]", "value": True}
   ])
   ```

### If a Cookbook Isn't in the System Yet

The user can add a new cookbook using the **cookbook-index-to-caves** skill. That skill handles:
- Creating a new perspective for the cookbook
- Transcribing index pages from uploaded images
- Building the hierarchical connection structure

Direct the user to upload images of the cookbook's cover and index pages, then use that skill to process them. Once the cookbook perspective exists, come back here and connect it to the user's ownership PID.

## Query Strategy

### Step 1: Resolve Owned Cookbooks (cache this)

Collect ALL cookbook PIDs the user owns. Each cookbook IS a shadow/perspective, and users can own cookbooks in two ways:
1. They directly own the cookbook PID (it appears in their `my_shadows()` list)
2. They track ownership via connections in one of their perspectives (e.g., `cookbooks ← "salt fat acid heat"`)

**Discovery workflow (all-of-the-above approach):**

1. **Get all user perspectives**: `caves__my_shadows()`

2. **Discover cookbook PIDs from multiple sources**:

   **a. Direct Cookbook Ownership** (user owns the cookbook PID):
   - Check each user perspective to see if it IS a cookbook
   - Look for standard taxonomy connections: `cookbooks ← {perspective-username}`
   - Check for bidirectional author connections (sign of cookbook structure)
   - If perspective is a cookbook, add its PID to collection

   **b. Tracked Cookbook Ownership** (user tracks cookbooks in their perspective):
   - For each user perspective, load its graph: `caves__my_shadow_caves(pId=..., format="md")`
   - Look for connections like: `cookbooks ← "salt fat acid heat"`
   - For each book title found:
     * Get subcaves of the book title: `caves__get_subcaves(parentCave="salt fat acid heat")`
     * Look for a PID in the subcaves (structure is: `cookbooks ← {book title} ← pID`)
     * This PID is the cookbook's perspective ID
   - Add discovered cookbook PIDs to collection

3. **Merge and deduplicate**: Combine all discovered PIDs, removing duplicates

4. **For each cookbook PID**, look up its name: `caves__my_shadow_caves(pId="cookbook-pid", format="md")` — the perspective username gives the cookbook name/author

**Cache this PID → cookbook name mapping in memory** so subsequent queries skip this step.

**Key insight**: Since each cookbook is its own perspective, we need to find the cookbook's PID by looking at subcaves (children) of the book title cave. The structure is: `cookbooks ← {book title} ← pID`. When a user tracks ownership via `cookbooks ← "salt fat acid heat"`, we can find the cookbook PID by getting subcaves of "salt fat acid heat".

### Step 2: Keyword Search

With the list of owned cookbook PIDs, use the **PID filter** to search only within your owned cookbooks:

**Recommended approach (most efficient):**

```python
# Convert owned PIDs to comma-separated string
pidsFilter = ",".join(ownedCookbookPIDs)  # e.g., "0x91A5...,0xEc89...,0x6aFf..."

# Search with PID filter - results are already filtered to owned cookbooks!
results = caves__search_caves(query="chicken", pIds=pidsFilter)
```

The search returns only matches from your owned cookbooks, eliminating the need for post-filtering.

**Performance note**: The `pIds` parameter makes searches more computationally intensive. Only use it when:
- User has at least one owned cookbook
- You need to filter to specific perspectives

**Alternative approach (when PID filtering isn't needed):**

If you're doing exploratory searches or the user has no owned cookbooks yet:

1. Search without filter: `caves__search_caves(query="chicken")`
2. Get subcaves of matched cave: `caves__get_subcaves(parentCave="chicken")`
3. Use `caves__getPidsForTags` or `caves__getPidsForConnection` to find which PIDs have the ingredient
4. Filter results by intersecting with your owned cookbook PIDs

### Step 3: Assemble Results

For each match, present:
- Cookbook name (from cached PID mapping)
- Ingredient/recipe name
- Page number(s)
- Any sub-entries (recipe names under the ingredient)

## Tools Reference

```python
# Get user's perspectives
caves__my_shadows()

# Get subcaves for a parent
caves__get_subcaves(parentCave="cookbooks", pId="optional-pid")

# Get full graph for a perspective
caves__my_shadow_caves(pId="cookbook-pid", format="md")

# Search for caves by keyword (optionally filtered to specific PIDs)
caves__search_caves(query="chicken")  # All perspectives
caves__search_caves(query="chicken", pIds="0x91A5...,0xEc89...")  # Filter to specific cookbooks

# Find which PIDs made a specific connection
caves__getPidsForConnection(child="42", parent="chicken")

# Find PIDs that have interacted with specific tags
caves__getPidsForTags(tags=["chicken", "thighs"], matchMode="any")
```

## Query Patterns

### "What recipes use [ingredient]?"
1. Search with PID filter: `caves__search_caves(query="chicken", pIds=pidsFilter)`
2. For each result, get subcaves: `caves__get_subcaves(parentCave="chicken")`
3. Match page numbers and recipes to cookbook names (from cache)
4. Return cookbook name + recipe/ingredient + page numbers

**Example result format:**
- **Ottolenghi Simple**: Pancetta, Pasta, and Shiitake Mushrooms (page 117)
- **Fantastic Fungi Community Cookbook**: Shiitake (pages 37, 45, 78, 84, 117, 173, 191, 252)

### "Which cookbooks have [ingredient]?"
1. Search with PID filter: `caves__search_caves(query="chicken", pIds=pidsFilter)`
2. Results are already filtered to owned cookbooks
3. Look at the `caves` field in subcaves response to see which cookbook perspectives have connections
4. Return list of cookbook names

**Alternative (without PID filter):**
1. Search: `caves__search_caves(query="chicken")`
2. Use `getPidsForTags(tags=["chicken"])` to find all PIDs connected to it
3. Filter to owned cookbook PIDs by intersecting with cached owned PIDs
4. Return cookbook names

### "What's on page [N] of [cookbook]?"
1. Find the cookbook's PID from cache
2. Search that PID's graph for the page number as a child
3. Return the parent entries (ingredients/recipes) pointing to that page

### "Show me everything in [cookbook]"
1. Find the cookbook's PID from cache
2. Load full graph: `caves__my_shadow_caves(pId="pid", format="md")`
3. Present the hierarchical index

## Complete Example Workflow

**User asks:** "Which of my cookbooks have shiitake mushrooms?"

```python
# Step 1: Get owned cookbook PIDs (cached after first query)
shadows = caves__my_shadows()
# Discover: ownedCookbookPIDs = ['0x91A5...', '0xEc89...', '0x6aFf...']
# Cache: cookbookNames = {'0x91A5...': 'fantastic fungi community cookbook', ...}

# Step 2: Search with PID filter
pidsFilter = "0x91A5...,0xEc89...,0x6aFf..."
results = caves__search_caves(query="shiitake", pIds=pidsFilter)
# Returns: [{"tag": "shiitake", "similarity": 1.0, ...}, ...]

# Step 3: Get details for the matched ingredient
details = caves__get_subcaves(parentCave="shiitake")
# Returns:
# - items: [{"tag": "37", ...}, {"tag": "252", ...}, ...]  # page numbers
# - caves: [{"tag": "fantastic fungi community cookbook", ...}, ...]  # parent cookbooks

# Step 4: Present results
# Fantastic Fungi Community Cookbook: Shiitake (pages 37, 45, 78, 84, 117, 173, 191, 252)
# Ottolenghi Simple: Pancetta, Pasta, and Shiitake Mushrooms (page 117)
```

## Tips

- **Always use PID filtering when user owns cookbooks** — it's more efficient and prevents pollution from other users' data
- Use `format="md"` for shadow caves — it's more compressed and saves context
- Ingredient searches work best with single words: "chicken" not "chicken thighs"
- Page numbers are stored as strings (e.g., "42", "214-217")
- Multiple cookbooks may share the same ingredient cave — the PID determines which book's page number is which
- When results span many cookbooks, group output by cookbook for readability
- Cache the owned cookbook PIDs mapping for the entire session to avoid redundant discovery
