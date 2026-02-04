---
name: cookbook-index-to-caves
description: Convert cookbook index pages into Caves knowledge graph connections. Use when user uploads cookbook index images and wants to organize recipe/ingredient relationships hierarchically in Caves. Handles transcription, structure normalization, and batch connection creation.
---

# Cookbook Index to Caves

Convert cookbook index pages into structured Caves connections, preserving hierarchical relationships between ingredients, recipes, techniques, and page numbers.

## Prerequisites

- **Caves account with MCP access required**
  - Sign up at [itscaves.com](https://itscaves.com)
  - To get MCP access, contact on Discord: https://discord.com/invite/8bW2xwv7kc
- **Front cover image required** - Take a clear photo of the book's front cover for metadata extraction
- Use `caves__my_shadows` to verify you have perspective access before starting

## Accessing Caves

Caves can be accessed via URLs using the schema:
```
https://itscaves.com/c/{url-encoded-cave-name}
```

**Examples:**
- `https://itscaves.com/c/smoke%20and%20pickles` - Access the "smoke and pickles" perspective (book title)
- `https://itscaves.com/c/edward%20lee` - Access the "edward lee" cave (author)
- `https://itscaves.com/c/kimchi` - Access the "kimchi" cave (ingredient/recipe)

When sharing caves with users, provide the URL-encoded link for direct navigation.

## Workflow

### 1. Extract Metadata from Cover
- Identify the front cover image (user should provide this first)
- If HEIC format, convert to JPEG: `sips -s format jpeg cover.HEIC --out cover.jpg`
- Resize if needed: `sips -Z 1200 cover.jpg`
- Extract from the cover image:
  - Book title (for perspective username)
  - Author's full name (for bidirectional connection)
- Confirm extracted metadata with user if uncertain

### 2. Check for Existing Cookbook

**CRITICAL: Always check if this cookbook already exists before creating a new perspective.**

**Search process:**

1. **Search for the book title**: `caves__search_caves(query="[book title]")`
   - Look for exact or close matches to the book title
   - Example: Search for "salt fat acid heat"

2. **Check subcaves of "cookbooks"**: `caves__get_subcaves(parentCave="cookbooks")`
   - This shows all cookbooks currently in the system
   - Look for the book title in the caves list

3. **If a match is found**:
   - Check if it's a perspective: Look for a PID in the subcaves
   - Verify it's the same book by checking the author connection
   - Get the perspective details: `caves__my_shadow_caves(pId="found-pid", format="md")`
   - **Present findings to the user**:
     - "I found an existing cookbook perspective for '[book title]' by [author]"
     - "It already has [X] index entries"
     - "Would you like to: (a) Use this existing perspective and add to it, (b) Create a new separate perspective with a different title?"

4. **If no match is found**:
   - Confirm with user: "I didn't find '[book title]' in the system. Proceeding to create new perspective."

5. **If creating a new perspective when similar one exists**:
   - Suggest adding a distinguishing suffix
   - Example: "salt fat acid heat (2nd edition)" or "salt fat acid heat (personal copy)"
   - This prevents confusion when multiple versions exist

**Why this matters:**
- Prevents duplicate cookbooks in the system
- Avoids confusion for users who already added this book
- Allows adding to existing incomplete indexes
- Maintains data consistency across the Caves ecosystem

### 3. Create New Perspective
- Create a new Caves perspective using `caves__createPerspective` with book title as username
- Store the returned pId for all subsequent connections
- Create bidirectional connection between book title and author: `book title ← author` AND `book title → author`
- Connect the pId to title, author, and thing as parents: `{book title} ← {pId}`, `{author} ← {pId}`, AND `thing ← {pId}`
- Create standard taxonomy connections:
  - `books ← cookbooks` (establishes cookbooks category)
  - `cookbooks ← {book title}` (places this book in cookbooks)
  - `indexes ← {book title}` (marks this book as having an index)

### 4. Process Index Images
- Convert HEIC images to JPEG if needed
- Resize large images for readability: `sips -Z 1200 image.jpg`
- Transcribe each index page from the images

### 5. Create Index Connections
- Check existing connections with `caves__my_shadow_caves` to avoid duplicates
- Process all index entries in large batches (API handles 50-100+ connections well)
- Maintain all hierarchical relationships from the index (ingredients → recipes)
- Create direct connections from book title to recipe/section entries (but NOT to page numbers)
- Page numbers should ONLY be connected under their parent recipe/section
- **Stop and consult the user on any new edge cases**

## Structure Rules

### Hierarchy
```
book title (perspective username, e.g., "salt fat acid heat")
  ├── author (bidirectional connection, e.g., "samin nosrat")
  ├── entry (ingredient/recipe name/category)
  │   ├── page number (if entry has a page)
  │   └── child entries (if entry has sub-items)
  │       └── page numbers for children
  └── direct recipe connections (any entry with a page number)
      └── page number
```

**Note**: The book title is the perspective itself (created via `createPerspective`). The author is bidirectionally connected to the book title as the first connections made within that perspective.

**Direct Access Pattern**: Recipe and section entries (not page numbers) should be connected directly to the book title for easy access. Page numbers are ONLY connected under their parent recipe/section, never directly to the book title.

**Example hierarchy + direct access:**
```
book title ← aioli (direct access - it's a recipe)
aioli ← 236 (page number under recipe)
aioli ← romano beans with aioli (child recipe)
romano beans with aioli ← 236 (child's page number)
book title ← romano beans with aioli (direct access - it's also a recipe)
```

### Formatting
- Lowercase everything
- Expand page ranges: "214–17" → "214-217"
- Strip prefixes: "in", "with", "on" from sub-items (treat as regular children)

### Entry Types

**Simple entry with page:**
```
# Direct connection to book title (recipe level):
book title ← adas polo

# Page number under the recipe:
adas polo ← 158
```
**Note:** Recipe entries are connected directly to book title. Page numbers are only connected under their recipe parent.

**Entry with sub-items (no page on parent):**
```
# Parent without page (category level):
book title ← agave syrup
agave syrup ← cucumber-chia agua fresca

# Page number under the child recipe:
cucumber-chia agua fresca ← 222

# Direct connection for the child recipe:
book title ← cucumber-chia agua fresca
```

**Entry with both page and sub-items:**
```
# Direct connections to book title (both are recipes):
book title ← aioli
book title ← romano beans with aioli

# Hierarchical structure:
aioli ← 236  # page number under parent
aioli ← romano beans with aioli  # child recipe
romano beans with aioli ← 236  # page number under child
```

**Qualifier entries (comma-separated become parent/child):**
```
# Category and qualifiers:
book title ← apricots
apricots ← dried
apricots ← fresh

# Recipe under qualifier:
dried ← chicken braised with apricots and harissa
chicken braised with apricots and harissa ← 89  # page number under recipe

# Direct connection (recipe level):
book title ← chicken braised with apricots and harissa
```

**Technique references (combine with parent context):**
```
# Category:
book title ← beans

# Technique under category:
beans ← cooking beans
cooking beans ← 159  # page number under technique

# Direct connection (technique/section level):
book title ← cooking beans
```

**"See also" references (treat as regular children):**
```
book title ← beans
beans ← chickpeas
beans ← shell beans
beans ← string beans
```

**Qualifier with page number (e.g., "about", "see"):**
```
# Direct connection to book title (section level):
book title ← peas

# Hierarchical structure with page number:
peas ← about
about ← 12  # page number under qualifier only
```
**Note:** When a qualifier word (like "about", "see", etc.) appears before a page number, create the hierarchical connection (entry ← qualifier ← page). The page number is ONLY connected under its immediate parent (the qualifier).

## Edge Case Handling

**Critical: Stop and consult the user when encountering:**
- New prefix types beyond "in", "with", "on"
- Ambiguous hierarchy structures
- Cross-reference formats not yet discussed
- Any pattern that doesn't clearly fit existing rules

**Page Number Rule**: Page numbers are ONLY connected under their immediate parent (recipe/section/qualifier). They are NEVER connected directly to the book title. Recipe and section entries (not page numbers) are connected directly to the book title for easy access.

**Known patterns:**
| Pattern | Action |
|---------|--------|
| "in XXX" | Strip "in", link as child |
| "with XXX" | Strip "with", link as child |
| "on XXX" | Strip "on", link as child |
| "See also X; Y; Z" | Create children: parent ← X, parent ← Y, parent ← Z |
| "See XXX" | Create child: parent ← XXX |
| "entry, qualifier" | Create: entry ← qualifier |
| "technique, page" under parent | Create: parent ← "technique parent", "technique parent" ← page, book title ← "technique parent" (page only under technique) |
| "entry, qualifier, page" (e.g., "peas, about, 12") | Create: entry ← qualifier, qualifier ← page, book title ← entry (page only under qualifier) |

## Connection Format

Use Caves connect_caves tool with batched connections:
```python
# Step 1: Create the perspective with book title
caves__createPerspective(username="salt fat acid heat")
# Returns: { pId: "abc123...", username: "salt fat acid heat", ... }

# Step 2: Create connections starting with bidirectional book title ↔ author and taxonomy
caves__connect_caves(
    pId="abc123...",
    connections=[
        # Bidirectional connection: book title ↔ author
        {"parent": "salt fat acid heat", "child": "samin nosrat", "value": True},
        {"parent": "samin nosrat", "child": "salt fat acid heat", "value": True},

        # Connect pId as child of title, author, and thing
        {"parent": "salt fat acid heat", "child": "abc123...", "value": True},
        {"parent": "samin nosrat", "child": "abc123...", "value": True},
        {"parent": "thing", "child": "abc123...", "value": True},

        # Standard taxonomy connections
        {"parent": "books", "child": "cookbooks", "value": True},
        {"parent": "cookbooks", "child": "salt fat acid heat", "value": True},
        {"parent": "indexes", "child": "salt fat acid heat", "value": True},

        # Recipe-level connections (direct to book title)
        {"parent": "salt fat acid heat", "child": "aioli", "value": True},
        {"parent": "salt fat acid heat", "child": "romano beans with aioli", "value": True},
        {"parent": "salt fat acid heat", "child": "adas polo", "value": True},

        # Hierarchical structure with page numbers
        {"parent": "aioli", "child": "236", "value": True},  # page under recipe
        {"parent": "aioli", "child": "romano beans with aioli", "value": True},  # child recipe
        {"parent": "romano beans with aioli", "child": "236", "value": True},  # page under child
        {"parent": "adas polo", "child": "158", "value": True},  # page under recipe

        # ... more entries
    ]
)
```

### Adding to an Existing Cookbook

If the cookbook already exists in Caves (found during Step 2 check):

```python
# Use the existing perspective PID (from search results)
existingPId = "0x6aFf..."  # Found from caves__get_subcaves(parentCave="cookbooks")

# Check what's already there
caves__my_shadow_caves(pId=existingPId, format="md")
# Review existing entries to avoid duplicates

# Add new index entries to the existing perspective
caves__connect_caves(
    pId=existingPId,  # Use existing PID, not a new one
    connections=[
        # Add new recipe entries that don't exist yet
        {"parent": "salt fat acid heat", "child": "new recipe name", "value": True},
        {"parent": "new recipe name", "child": "145", "value": True},

        # ... more new entries
    ]
)
```

**Important when adding to existing:**
- Always check existing connections first with `caves__my_shadow_caves` to avoid duplicates
- Don't recreate the author connection or taxonomy connections (they already exist)
- Only add new index entries that aren't already present
- Use the same lowercase formatting and structure as existing entries

## Quality Checklist

Before creating connections:
- [ ] Extracted book title and author name from cover image
- [ ] Confirmed metadata accuracy with user if uncertain
- [ ] **CHECKED if cookbook already exists in Caves** (searched book title, checked cookbooks subcaves)
- [ ] If existing cookbook found, consulted user about using existing vs creating new
- [ ] If creating new when similar exists, added distinguishing suffix to title
- [ ] Created new perspective with book title using `caves__createPerspective`
- [ ] Stored pId from perspective creation
- [ ] Created bidirectional connection: book title ↔ author as first connections
- [ ] Connected pId as child of title, author, and thing: {title} ← {pId}, {author} ← {pId}, thing ← {pId}
- [ ] Created standard taxonomy connections: books ← cookbooks, cookbooks ← {book title}, indexes ← {book title}
- [ ] Checked for existing connections with `caves__my_shadow_caves` to avoid duplicates
- [ ] All text lowercased
- [ ] Page ranges expanded (e.g., "214–17" → "214-217")
- [ ] Prefixes stripped from sub-items
- [ ] Technique references contextualized with parent
- [ ] "See also" items converted to children
- [ ] All hierarchical relationships from index preserved
- [ ] Recipe and section entries connected directly to book title
- [ ] Page numbers ONLY connected under their immediate parent (never to book title)
- [ ] New edge cases flagged for user review
- [ ] Batching connections efficiently (50-100+ per batch)
