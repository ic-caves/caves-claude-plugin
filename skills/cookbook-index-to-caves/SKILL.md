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

### 2. Resolve or Create Perspective

**Use the `perspective-resolver` skill to handle perspective management.**

This cookbook perspective represents the book itself. Invoke the perspective-resolver skill with these parameters:

```javascript
// Invoke: perspective-resolver skill
{
  primary_tag: book_title,  // e.g., "salt fat acid heat"
  identifying_tags: [book_title, author_name],  // e.g., ["salt fat acid heat", "samin nosrat"]
  bidirectional_pairs: [[book_title, author_name]],
  taxonomy: [
    {parent: "books", child: "cookbooks"},
    {parent: "cookbooks", child: book_title},
    {parent: "indexes", child: book_title},
    {parent: "thing", child: "{pId}"}
  ],
  entity_type: "cookbook",
  entity_description: `${book_title} by ${author_name}`
}
```

The perspective-resolver skill will:
- Search for existing perspectives using the identifying tags
- Present findings to user if cookbook already exists
- Handle user choice (reuse existing vs create new)
- Suggest distinguishing suffixes if needed
- Create all initial connections (bidirectional, identifying, taxonomy)
- Return the pId to use

**Store the returned pId** for use in Step 3 (index connections).

### 3. Process Index Images
- Convert HEIC images to JPEG if needed
- Resize large images for readability: `sips -Z 1200 image.jpg`
- Transcribe each index page from the images

### 4. Create Index Connections
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

The perspective-resolver skill (Step 2) handles initial setup. You only create index-specific connections (Step 4):

```python
# Step 2: Invoke perspective-resolver skill
# Returns pId and creates:
# - Bidirectional connection: salt fat acid heat ↔ samin nosrat
# - Identifying connections: salt fat acid heat ← pId, samin nosrat ← pId
# - Taxonomy connections: books ← cookbooks ← salt fat acid heat, indexes ← salt fat acid heat, thing ← pId

# Step 4: Create index-specific connections
caves__connect_caves(
    pId="abc123...",  # pId returned from perspective-resolver
    connections=[
        # Recipe-level connections (direct to book title)
        {"parent": "salt fat acid heat", "child": "aioli", "value": True},
        {"parent": "salt fat acid heat", "child": "romano beans with aioli", "value": True},
        {"parent": "salt fat acid heat", "child": "adas polo", "value": True},

        # Hierarchical structure with page numbers
        {"parent": "aioli", "child": "236", "value": True},  # page under recipe
        {"parent": "aioli", "child": "romano beans with aioli", "value": True},  # child recipe
        {"parent": "romano beans with aioli", "child": "236", "value": True},  # page under child
        {"parent": "adas polo", "child": "158", "value": True},  # page under recipe

        # ... more index entries
    ]
)
```

### Adding to an Existing Cookbook

If the cookbook already exists in Caves, the perspective-resolver skill (Step 2) will find it and ask the user:

```python
# Step 2: Invoke perspective-resolver skill
# - Finds existing cookbook
# - User chooses option (a) to use existing perspective
# - Returns: pId = "0x6aFf..." (existing, is_new: false)

# Check what's already there
caves__my_shadow_caves(pId=pId, format="md")
# Review existing entries to avoid duplicates

# Add new index entries to the existing perspective
caves__connect_caves(
    pId=pId,  # Use existing pId returned from perspective-resolver
    connections=[
        # Add new recipe entries that don't exist yet
        {"parent": "salt fat acid heat", "child": "new recipe name", "value": True},
        {"parent": "new recipe name", "child": "145", "value": True},

        # ... more new entries
    ]
)
```

**Important when adding to existing:**
- The perspective-resolver skill handles finding and confirming with user
- Always check existing connections first with `caves__my_shadow_caves` to avoid duplicates
- Initial connections (author, taxonomy) already exist - don't recreate them
- Only add new index entries that aren't already present
- Use the same lowercase formatting and structure as existing entries

## Quality Checklist

Before creating connections:
- [ ] Extracted book title and author name from cover image
- [ ] Confirmed metadata accuracy with user if uncertain
- [ ] Used perspective-resolver skill (Step 2):
  - [ ] Invoked with cookbook parameters (title, author, taxonomy)
  - [ ] Stored returned pId for index connections
- [ ] Checked for existing index connections with `caves__my_shadow_caves` to avoid duplicates
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
