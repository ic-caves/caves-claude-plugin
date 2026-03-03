---
name: recipe-to-caves
description: Extract recipe content from photos and upload to Caves with ingredient connections. Use when user uploads recipe page images and wants to preserve recipe text in IPFS while building knowledge graph connections between recipes, cookbooks, and ingredients.
---

# Recipe Photo to Caves

Convert recipe page photos into structured knowledge graph connections with full recipe text preserved in IPFS. This skill extracts recipe details, links them to cookbooks, creates hierarchical ingredient relationships (generic and specific), and preserves the original recipe text permanently.

## Prerequisites

Before starting, verify these requirements are met:

1. **Caves Account**: User must have an account at itscaves.com
2. **MCP Access**: Verify with `caves__my_shadows()` - should return user's perspectives
3. **Recipe Image**: User has recipe page photo(s) ready
4. **Fire Balance**: Warn user about connection costs (especially for first-time users)

If MCP tools are not accessible, inform the user they need to set up Caves MCP integration first.

## When to Invoke This Skill

Use this skill when the user wants to:
- Upload recipe page photos to their knowledge graph
- Extract and preserve recipe text from cookbook pages
- Link recipes to cookbooks and ingredients
- Build searchable recipe collections
- Organize recipes with ingredient relationships

**Keywords that should trigger this skill:**
- "add recipe photo to caves"
- "extract recipe from image"
- "upload recipe page to knowledge graph"
- "process recipe photo"
- "preserve recipe in caves"

## Workflow Instructions for Claude

Follow these 7 phases to process recipe photos:

### Phase 0: Prerequisites Check

**Objective**: Verify Caves access and warn about costs.

**Instructions**:
1. Check Caves access:
   ```
   Call: caves__my_shadows()
   ```
   - If fails: User needs MCP setup, provide instructions
   - If succeeds: Store perspectives for later use

2. Warn about fire balance:
   ```
   Claude: "Recipe processing creates connections in your Caves knowledge graph.
           Each connection requires fire balance.

           Typical recipe creates:
           - 1 cookbook connection
           - 5-15 ingredient connections (10-30 with bidirectional)
           - 3-5 ingredient bridge connections
           - 1 IPFS content connection
           Total: ~15-50 connections per recipe

           Would you like to proceed?"
   ```

3. Accept image(s) from user:
   - Single recipe: 1 image
   - Multi-page recipe: Multiple images (will concatenate)
   - Multiple recipes: Process individually (ask first)

### Phase 1: Image Classification

**Objective**: Determine if this is a recipe page or index page.

**Instructions**:
1. Use Read tool to analyze the image(s)

2. Look for recipe indicators:
   - **Recipe page**:
     - Prominent title at top
     - Ingredient list (bulleted/numbered)
     - Paragraph instructions
     - Usually 1-2 recipes per page

   - **Index page**:
     - Multi-column layout
     - Page numbers after entries
     - Alphabetical ordering
     - Many entries (20+ items)
     - Minimal paragraph text

3. Classification decision:
   ```python
   if index_page:
       # Redirect to cookbook-index-to-caves skill
       message = "This looks like an index page, not a recipe page.
                  I'll redirect to the cookbook-index-to-caves skill instead."
       # Invoke: cookbook-index-to-caves skill

   elif ambiguous:
       # Ask user
       message = "I'm not certain if this is a recipe page or index page.
                  Which is it?"
       # Wait for user response

   else:
       # Continue to Phase 2
       pass
   ```

**Example**:
```
Claude: "Analyzing image...
        This looks like a recipe page for 'Focaccia' with an ingredient
        list and instructions. Proceeding with recipe extraction."
```

### Phase 2: Recipe Extraction

**Objective**: Parse structured recipe data from image(s).

**Instructions**:
1. Extract recipe components:

   **Required fields**:
   - **Title**: Main heading (usually largest text or first heading)
   - **Ingredients**: List with quantities, units, and preparations
   - **Instructions**: Step-by-step cooking directions

   **Optional fields** (extract if present):
   - **Yield/Servings**: "Serves 4", "Makes 12", etc.
   - **Time**: Prep time, cook time, total time
   - **Temperature**: Oven temperatures, heat levels
   - **Source notes**: Recipe origin, attribution, headnotes

2. Format extraction as JSON:
   ```json
   {
     "title": "focaccia",
     "ingredients": [
       "4 cups all-purpose flour",
       "2 tsp salt",
       "1 packet active dry yeast",
       "2 cups warm water",
       "1/4 cup olive oil",
       "2 tbsp rosemary, fresh",
       "coarse sea salt for topping"
     ],
     "instructions": "In a large bowl, combine flour and salt. In another bowl, dissolve yeast in warm water and let stand 5 minutes. Add yeast mixture and olive oil to flour, stirring until combined. Knead on floured surface for 10 minutes until smooth. Place in oiled bowl, cover, and let rise 1 hour. Punch down, shape into rectangle on baking sheet. Dimple with fingers, brush with olive oil, top with rosemary and salt. Let rise 30 minutes. Bake at 425°F for 25 minutes until golden.",
     "metadata": {
       "yield": "1 large loaf (serves 8)",
       "time": "2 hours (including rising)",
       "temperature": "425°F"
     }
   }
   ```

3. Validation:
   - Must have: title AND ingredients AND instructions
   - If missing any required field:
     ```
     Claude: "I'm having trouble extracting the complete recipe.
             I found:
             - Title: [yes/no]
             - Ingredients: [yes/no]
             - Instructions: [yes/no]

             Could you provide the missing information?"
     ```

4. Multiple recipes per page:
   - If image contains 2+ recipes: Ask user which to process
   - Or offer to process all separately

5. Multi-page recipes:
   - If user provides multiple images for one recipe:
     - Concatenate ingredients/instructions from all pages
     - Note in metadata: "multi_page: true"

**Quality check**: Show extracted recipe summary to user for confirmation:
```
Claude: "Extracted recipe: Focaccia
        - 7 ingredients
        - Instructions: [first 100 chars]...
        - Yield: 1 large loaf

        Does this look correct?"
```

### Phase 3: Duplicate Check

**Objective**: Prevent unintentional duplicate recipes in knowledge graph.

**Instructions**:
1. Search for existing recipe:
   ```
   Call: caves__search_caves(query=recipe_title)
   ```

2. Evaluate search results:
   - **Score 10**: Exact match (same recipe name)
   - **Score 8-9**: Very similar (likely same recipe)
   - **Score 5-7**: Related (different recipe, similar name)
   - **Score 0-4**: Unrelated

3. Handle high-confidence matches (score ≥ 8):
   ```
   Claude: "Found existing recipe in Caves:
           - Name: focaccia
           - Connected to: salt fat acid heat (cookbook)
           - Similarity score: 9/10

           Options:
           a) Skip this recipe (already exists)
           b) Create with distinguishing suffix (e.g., 'focaccia (rosemary)')
           c) Replace existing (disconnect old, create new)

           What would you like to do?"
   ```

4. User choice handling:
   - **Skip**: Exit gracefully, offer to process next image
   - **Suffix**: Ask for distinguishing detail, update recipe_title
   - **Replace**: Note old recipe tag for disconnection in Phase 7

5. Low confidence or no match (score < 8):
   - Proceed to Phase 4
   - No user interaction needed

### Phase 4: Cookbook Perspective Resolution

**Objective**: Determine which cookbook this recipe belongs to.

**Instructions**:

#### Step 4a: Determine Cookbook Context

Look for cookbook information in these places (in order):

1. **User explicitly stated**: "This is from Salt Fat Acid Heat by Samin Nosrat"
   - Extract: cookbook_title = "salt fat acid heat", author = "samin nosrat"

2. **Image shows book metadata**:
   - Header/footer with book title
   - Page attribution text
   - Copyright notice
   - Extract if present

3. **No clear context**:
   - Ask user (see Step 4b)

#### Step 4b: Ask User for Cookbook Information (if needed)

If cookbook context is unclear:

```
Claude: "Which cookbook is this recipe from?

        Options:
        a) Enter cookbook title and author
        b) Create as standalone recipe (no cookbook)
        c) Show me the extracted recipe details first

        Note: Cookbook connections help organize and discover
              related recipes in your knowledge graph."
```

**Handle response**:
- **Option a**: Ask follow-up: "What is the cookbook title and author?"
- **Option b**: Skip to Phase 4d (standalone handling)
- **Option c**: Show full recipe JSON, then re-ask

#### Step 4c: Use perspective-resolver Skill

**For cookbook recipes** (when user provides cookbook info):

Invoke the **perspective-resolver** skill with cookbook parameters:

```javascript
// Invoke: perspective-resolver skill
{
  primary_tag: cookbook_title,  // e.g., "salt fat acid heat"
  identifying_tags: [cookbook_title, author_name],  // e.g., ["salt fat acid heat", "samin nosrat"]
  bidirectional_pairs: [[cookbook_title, author_name]],
  taxonomy: [
    {parent: "books", child: "cookbooks"},
    {parent: "cookbooks", child: cookbook_title},
    {parent: "thing", child: "{pId}"}
  ],
  entity_type: "cookbook",
  entity_description: `${cookbook_title} by ${author_name}`
}
```

The perspective-resolver skill will:
- Search for existing cookbook perspectives
- Ask user to reuse or create new
- Create perspective with `caves__createPerspective()` if needed
- Set up initial connections (bidirectional, identifying, taxonomy)
- Return pId to use

**Store the returned pId** for Phase 7 (connection creation).

#### Step 4d: Standalone Recipe Handling

**For standalone recipes** (when user selects option b):

Two approaches:

1. **Create temporary "my recipes" perspective** (recommended for multiple standalone recipes):
   ```javascript
   // Invoke: perspective-resolver skill (first standalone recipe only)
   {
     primary_tag: "my recipes",
     identifying_tags: ["my recipes"],
     taxonomy: [
       {parent: "recipes", child: "my recipes"},
       {parent: "thing", child: "{pId}"}
     ],
     entity_type: "recipe collection",
     entity_description: "standalone recipes collection"
   }
   ```
   - Reuse this perspective for subsequent standalone recipes
   - Store pId for Phase 7

2. **Skip cookbook connection** (simpler for one-off recipes):
   - Use user's primary perspective (from `caves__my_shadows()`)
   - Recipe won't be connected to any cookbook
   - Still create ingredient connections and IPFS upload

**Store the pId** (or note to skip cookbook connection) for Phase 7.

### Phase 5: Ingredient Entity Resolution

**Objective**: Create both generic and specific ingredient tags with bridge connections.

**Instructions**:

#### Step 5a: Parse Each Ingredient

For each ingredient in the extracted list:

1. **Parse ingredient string components**:
   ```python
   # Example: "2 cups all-purpose flour, sifted"

   components = {
     "quantity": "2 cups",
     "base": "flour",           # Generic ingredient
     "specific": "all-purpose flour",  # Specific variant
     "preparation": "sifted"    # Prep method
   }
   ```

2. **Extraction patterns**:
   - **Quantity**: Numbers + units (cups, tsp, oz, grams, etc.)
   - **Base**: Core ingredient (last significant noun)
   - **Specific**: Qualifier + base (adjective + noun)
   - **Preparation**: After comma (chopped, diced, minced, etc.)

3. **Common patterns**:
   ```
   "2 cups all-purpose flour" →
     base: "flour"
     specific: "all-purpose flour"

   "1 tsp salt" →
     base: "salt"
     specific: null (no qualifier)

   "1 packet active dry yeast" →
     base: "yeast"
     specific: "active dry yeast"

   "2 tbsp fresh rosemary" →
     base: "rosemary"
     specific: "fresh rosemary"

   "1/4 cup olive oil" →
     base: "oil"
     specific: "olive oil"

   "coarse sea salt for topping" →
     base: "salt"
     specific: "coarse sea salt"
   ```

#### Step 5b: Search and Resolve Generic Ingredients

For each unique base ingredient:

1. **Search Caves**:
   ```
   Call: caves__search_caves(query=base_ingredient)
   ```

2. **Evaluate results** (top 5):
   - **Score 10**: Exact match → use that tag name
   - **Score 7-9**: Very related → consider bridge connection
   - **Score 4-6**: Somewhat related → maybe bridge
   - **Score 0-3**: Unrelated → use our extracted name

3. **Decision logic**:
   ```python
   if exact_match (score == 10):
       canonical_generic = existing_tag_name
   elif close_match (score >= 7):
       # Consider creating bridge to related concept
       # Example: "flour" search finds "wheat"
       canonical_generic = our_base_ingredient
       bridge_to = existing_tag_name
   else:
       canonical_generic = our_base_ingredient
   ```

4. **Cache results** to avoid duplicate searches

#### Step 5c: Search and Resolve Specific Ingredients

For each specific ingredient (where qualifier exists):

1. **Search Caves**:
   ```
   Call: caves__search_caves(query=specific_ingredient)
   ```

2. **Same scoring logic** as Step 5b

3. **Decision**:
   - Exact match (score 10): Use existing tag
   - Otherwise: Use our extracted specific name

#### Step 5d: Build Connection Pairs

For each ingredient, create connection specification:

```python
ingredients_resolved = [
  {
    "generic": "flour",           # Canonical generic name
    "specific": "all-purpose flour",  # Specific variant name
    "bridge": True,               # Create generic ← specific
    "related_bridges": []         # Related concepts to bridge
  },
  {
    "generic": "salt",
    "specific": "coarse sea salt",
    "bridge": True
  },
  {
    "generic": "yeast",
    "specific": "active dry yeast",
    "bridge": True
  },
  {
    "generic": "rosemary",
    "specific": "fresh rosemary",
    "bridge": True
  },
  {
    "generic": "oil",
    "specific": "olive oil",
    "bridge": True
  }
]
```

**Example with related bridge**:
```python
# Search for "flour" found related concept "wheat" (score 8)
{
  "generic": "flour",
  "specific": "all-purpose flour",
  "bridge": True,
  "related_bridges": ["wheat"]  # Create flour ← wheat bridge
}
```

#### Step 5e: Summary Report

After resolution, show summary to user:

```
Claude: "Ingredient Resolution Complete

        Generic ingredients: 5
        - flour (found in Caves)
        - salt (found in Caves)
        - yeast (new)
        - rosemary (new)
        - oil (found in Caves)

        Specific variants: 5
        - all-purpose flour (new)
        - coarse sea salt (new)
        - active dry yeast (new)
        - fresh rosemary (new)
        - olive oil (found in Caves)

        Bridges to create: 6
        - flour ← all-purpose flour
        - salt ← coarse sea salt
        - yeast ← active dry yeast
        - rosemary ← fresh rosemary
        - oil ← olive oil
        - flour ← wheat (related concept)

        Ready to proceed?"
```

### Phase 6: Format Recipe Text & Upload to IPFS

**Objective**: Create clean markdown and upload to IPFS for permanent storage.

**Instructions**:

#### Step 6a: Format Recipe as Markdown

Create structured markdown:

```markdown
# {Recipe Title}

{Source attribution - see below}

## Ingredients

{Ingredient list - one per line}

## Instructions

{Full instructions paragraph(s)}

{Metadata section - see below}

---
*Extracted {date} via Caves recipe-to-caves skill*
```

**Source attribution** (first section after title):
```markdown
# Focaccia

From: **Salt Fat Acid Heat** by Samin Nosrat

## Ingredients
...
```

Or for standalone:
```markdown
# Focaccia

*Standalone recipe*

## Ingredients
...
```

**Metadata section** (optional, only if metadata exists):
```markdown
## Notes

- **Yield**: 1 large loaf (serves 8)
- **Time**: 2 hours (including rising)
- **Temperature**: 425°F
```

**Full example**:
```markdown
# Focaccia

From: **Salt Fat Acid Heat** by Samin Nosrat

## Ingredients

- 4 cups all-purpose flour
- 2 tsp salt
- 1 packet active dry yeast
- 2 cups warm water
- 1/4 cup olive oil
- 2 tbsp fresh rosemary
- Coarse sea salt for topping

## Instructions

In a large bowl, combine flour and salt. In another bowl, dissolve yeast in warm water and let stand 5 minutes. Add yeast mixture and olive oil to flour, stirring until combined. Knead on floured surface for 10 minutes until smooth. Place in oiled bowl, cover, and let rise 1 hour. Punch down, shape into rectangle on baking sheet. Dimple with fingers, brush with olive oil, top with rosemary and salt. Let rise 30 minutes. Bake at 425°F for 25 minutes until golden.

## Notes

- **Yield**: 1 large loaf (serves 8)
- **Time**: 2 hours (including rising)
- **Temperature**: 425°F

---
*Extracted 2026-03-03 via Caves recipe-to-caves skill*
```

#### Step 6b: Upload to IPFS

Upload the formatted markdown:

```javascript
Call: caves__upload_text(
  text: formatted_markdown,
  pId: cookbook_pId,  // Or standalone pId
  metadata: {
    title: recipe_title,
    source: cookbook_title || "standalone",
    content_type: "recipe",
    extracted_date: current_date_iso,
    author: author_name || null
  }
)
```

**Returns**: IPFS response with CID

```json
{
  "ipfsCid": "bafkreif...",
  "size": 1247,
  "url": "https://ipfs.itscaves.com/ipfs/bafkreif..."
}
```

**Store the CID** for Phase 7 (connection creation).

**Error handling**:
- If upload fails: Continue anyway, create connections without IPFS
- Warn user: "IPFS upload failed, recipe connections will be created without preserved text"

### Phase 7: Connection Creation

**Objective**: Create all knowledge graph connections in efficient batches.

**Instructions**:

#### Step 7a: Build Connection Batches

Organize connections into logical batches:

**Batch 1: Recipe → Cookbook** (if not standalone):
```javascript
batch1_cookbook = [
  {
    parent: cookbook_title,
    child: recipe_title,
    value: true
  }
]
```

**Batch 2: Recipe → Generic Ingredients (bidirectional)**:
```javascript
batch2_ingredients = []

for each ingredient in ingredients_resolved:
  # Recipe → Ingredient
  batch2_ingredients.push({
    parent: recipe_title,
    child: ingredient.generic,
    value: true
  })

  # Ingredient → Recipe (backlink)
  batch2_ingredients.push({
    parent: ingredient.generic,
    child: recipe_title,
    value: true
  })
```

**Batch 3: Generic → Specific Ingredient Bridges**:
```javascript
batch3_bridges = []

for each ingredient in ingredients_resolved:
  if ingredient.specific:
    # Generic ← Specific
    batch3_bridges.push({
      parent: ingredient.generic,
      child: ingredient.specific,
      value: true
    })

  if ingredient.related_bridges:
    for related in ingredient.related_bridges:
      # Generic ← Related
      batch3_bridges.push({
        parent: ingredient.generic,
        child: related,
        value: true
      })
```

**Batch 4: Recipe → IPFS Content**:
```javascript
batch4_ipfs = [
  {
    parent: recipe_title,
    child: `/ipfs/${ipfs_cid}`,
    value: true
  }
]
```

#### Step 7b: Combine All Batches

```javascript
all_connections = [
  ...batch1_cookbook,     // 1 connection
  ...batch2_ingredients,  // 2 * N connections (bidirectional)
  ...batch3_bridges,      // M connections
  ...batch4_ipfs         // 1 connection
]
```

**Example count** (5 ingredients):
- Cookbook: 1
- Ingredients bidirectional: 2 × 5 = 10
- Bridges: 5
- IPFS: 1
- **Total: 17 connections**

#### Step 7c: Execute Batch Creation

Create all connections in single API call:

```javascript
Call: caves__connect_caves(
  pId: cookbook_pId,  // Or standalone pId
  connections: all_connections
)
```

**Response**:
```json
{
  "success": true,
  "created": 15,
  "already_existed": 2,
  "total": 17
}
```

#### Step 7d: Handle Duplicate Recipe (if replacing)

If user chose "replace" in Phase 3:

1. **Disconnect old recipe** from cookbook:
   ```javascript
   Call: caves__disconnect_caves(
     pId: cookbook_pId,
     connection: {
       parent: cookbook_title,
       child: old_recipe_tag,
       value: true
     }
   )
   ```

2. Note in success message: "Replaced old recipe connection"

#### Step 7e: Report Results to User

Show comprehensive success message:

```
✓ Recipe Added to Caves!

Recipe: focaccia
Cookbook: salt fat acid heat by samin nosrat

Connections Created:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Recipe → Cookbook:                    1
Recipe ↔ Ingredients (bidirectional): 10 (5 pairs)
  - flour ↔ focaccia
  - salt ↔ focaccia
  - yeast ↔ focaccia
  - rosemary ↔ focaccia
  - oil ↔ focaccia

Ingredient Bridges:                   5
  - flour ← all-purpose flour
  - salt ← coarse sea salt
  - yeast ← active dry yeast
  - rosemary ← fresh rosemary
  - oil ← olive oil

Recipe → IPFS:                        1
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Total: 17 connections (15 new, 2 existed)

IPFS Content: /ipfs/bafkreif...
Size: 1.2 KB

View Your Recipe:
- Recipe: https://itscaves.com/c/focaccia
- Cookbook: https://itscaves.com/c/salt%20fat%20acid%20heat
- Ingredients: https://itscaves.com/c/flour

Next Steps:
- Upload more recipes from this cookbook
- Search recipes by ingredient: Use cookbook-query skill
- Explore ingredient connections in Caves
```

**If multiple recipes processed**:
```
✓ All Recipes Processed!

Processed: 3 recipes
Total connections: 52 (45 new, 7 existed)

Recipes added:
1. focaccia (17 connections)
2. roasted chicken (18 connections)
3. chocolate cake (17 connections)

View cookbook: https://itscaves.com/c/salt%20fat%20acid%20heat
```

## Edge Cases

### Multiple Recipes Per Page

**Detection**: Image analysis finds 2+ titles with separate ingredient lists

**Handling**:
```
Claude: "This page contains multiple recipes:
        1. Focaccia
        2. Ciabatta

        Options:
        a) Process both recipes separately
        b) Select specific recipe to process
        c) Skip this page

        What would you like to do?"
```

- **Option a**: Run Phases 2-7 for each recipe sequentially
- **Option b**: Ask "Which recipe number?" then process that one
- **Option c**: Exit gracefully

### Recipe Spans Multiple Images

**Detection**: User uploads 2+ images and states they're for one recipe

**Handling**:
1. Read all images in sequence
2. Extract ingredients from each image (may be split across pages)
3. Concatenate instructions from each image (preserve order)
4. Combine into single recipe JSON
5. Note in metadata: `"multi_page": true`
6. Single IPFS upload with combined content
7. Create connections as normal (one recipe, not multiple)

**Example**:
```
Claude: "Processing 2-page recipe...
        Page 1: Found ingredients (10 items)
        Page 2: Found instructions (continued from page 1)

        Combined recipe: Beef Wellington (1 recipe from 2 pages)"
```

### Ingredient Parsing Fails

**Detection**: Cannot extract ingredient list structure from image

**Causes**:
- Handwritten recipe
- Poor image quality
- Non-standard format
- OCR failure

**Handling**:
1. Show what was extracted:
   ```
   Claude: "Having trouble parsing ingredient list.
           I found text but couldn't identify the list structure.

           Raw text:
           [show extracted text]

           Could you help me identify the ingredients?
           Please list one per line."
   ```

2. Accept manual input from user
3. Continue with user-provided ingredient list
4. Flag in metadata: `"manual_ingredients": true`

### Non-English Recipe

**Detection**: Character analysis detects non-Latin alphabet or common non-English words

**Handling**:
1. Process as-is (don't attempt translation)
2. Note language in metadata if identifiable:
   ```javascript
   metadata: {
     ...
     "language": "es"  // ISO code if detected
   }
   ```
3. Ingredient resolution may find fewer matches (expected)
4. Continue normally - connections still valuable

**Example**:
```
Claude: "Detected recipe in Spanish: Paella
        Processing in original language.
        Ingredient matching may be limited for non-English terms."
```

### Handwritten Recipe

**Detection**: Image shows handwritten text rather than printed

**Handling**:
1. Best-effort OCR extraction
2. Expect lower accuracy
3. Show extracted recipe to user for verification:
   ```
   Claude: "Extracted from handwritten recipe (OCR may have errors):

           Title: [extracted title]
           Ingredients:
           - [ingredient 1]
           - [ingredient 2]
           ...

           Please review and correct any errors."
   ```
4. Accept corrections from user
5. Flag in metadata: `"handwritten": true, "ocr_verified": true`

### Unclear Cookbook Context

**Detection**: No book metadata visible, user doesn't know source

**Handling**: Already covered in Phase 4b - ask user for options:
- Enter cookbook info if they remember
- Create as standalone
- Skip cookbook association

### IPFS Upload Fails

**Detection**: `caves__upload_text()` returns error

**Causes**:
- Network issues
- File too large
- API timeout

**Handling**:
1. Log error
2. Continue with connection creation (skip IPFS connection)
3. Warn user:
   ```
   Claude: "⚠ IPFS upload failed (error: [message])

           Recipe connections will still be created, but full recipe
           text won't be preserved in IPFS.

           Proceed with connections only?"
   ```
4. If user approves: Create connections without batch4_ipfs
5. If user declines: Exit, offer to retry later

## Integration with Other Skills

### cookbook-index-to-caves

**Relationship**: Complementary skills for cookbook digitization

**Workflow**:
1. User processes index pages with cookbook-index-to-caves
2. Index creates: cookbook ← recipe_name ← page_number
3. User then processes recipe pages with recipe-to-caves
4. Recipe skill finds existing recipe_name tag (from index)
5. Recipe adds: recipe_name ← ingredients, recipe_name ← /ipfs/...

**Result**: Integrated graph where recipes have both index page numbers AND full content + ingredients

**Example**:
```
salt fat acid heat ← focaccia (from index, has page 127)
focaccia ← 127 (from index)
focaccia ← flour (from recipe photo)
focaccia ← /ipfs/bafk... (from recipe photo)
```

### cookbook-query

**Relationship**: Search/discovery uses recipe-to-caves data

**Example queries**:
- "Which recipes use olive oil?" → Finds recipes via ingredient connections
- "Show me recipes from Salt Fat Acid Heat" → Finds recipes via cookbook connection
- "Recipes with both flour and yeast" → Finds recipes via multiple ingredient connections

Recipe data enriches query results with full ingredient lists and recipe text.

### perspective-resolver

**Relationship**: Core dependency for perspective management

Recipe-to-caves invokes perspective-resolver in Phase 4 to:
- Find or create cookbook perspectives
- Handle duplicate cookbook detection
- Set up initial cookbook connections

### content-to-caves

**Relationship**: Parallel skills for different content types

- content-to-caves: Articles, essays, transcripts → concept connections
- recipe-to-caves: Recipe photos → recipe/ingredient connections

Both share patterns:
- Entity resolution with search
- Bridge connections to related concepts
- IPFS upload for content preservation
- Perspective management via perspective-resolver

## Verification Examples

### Example 1: Simple Recipe from Known Cookbook

**Input**: 1 image, "Focaccia" from Salt Fat Acid Heat

**Process**:
1. Phase 0: ✓ Caves access verified
2. Phase 1: ✓ Classified as recipe page
3. Phase 2: ✓ Extracted title, 7 ingredients, instructions
4. Phase 3: ✓ No duplicate found
5. Phase 4: ✓ Found existing "salt fat acid heat" perspective (reused)
6. Phase 5: ✓ Resolved 5 generic + 5 specific ingredients, 5 bridges
7. Phase 6: ✓ Uploaded 1.2 KB to IPFS
8. Phase 7: ✓ Created 17 connections (1 cookbook + 10 ingredients + 5 bridges + 1 IPFS)

**Outcome**: Recipe fully integrated into existing cookbook graph

### Example 2: Multi-Ingredient Recipe with Novel Ingredients

**Input**: 1 image, "Thai Green Curry" from unknown cookbook

**Process**:
1. Phase 0: ✓ Warned about fire costs
2. Phase 1: ✓ Recipe page confirmed
3. Phase 2: ✓ Extracted 15 ingredients (many Thai-specific)
4. Phase 3: ✓ No duplicate
5. Phase 4: ✓ User chose standalone (no cookbook)
6. Phase 5:
   - 15 generic ingredients (10 new to Caves, 5 found)
   - 12 specific variants (all new)
   - 12 bridges created
7. Phase 6: ✓ Uploaded 2.1 KB to IPFS
8. Phase 7: ✓ Created 44 connections (30 ingredients + 12 bridges + 1 IPFS + 1 "my recipes")

**Outcome**: Rich ingredient graph added to Caves, discoverable by future recipes

### Example 3: Duplicate Recipe with Variation

**Input**: 1 image, "Focaccia" (again) with different ingredients

**Process**:
1. Phases 0-2: Standard processing
2. Phase 3: ⚠ Found existing "focaccia" (score 10)
   - User chose: "Create with suffix"
   - New title: "focaccia (herbed)"
3. Phases 4-7: Proceeded with "focaccia (herbed)" as title

**Outcome**: Both recipes preserved, distinguished by suffix

### Example 4: Handwritten Recipe Card

**Input**: 1 image, handwritten "Grandma's Apple Pie"

**Process**:
1. Phase 2: ⚠ OCR struggled with handwriting
   - Showed extracted text to user
   - User corrected 3 ingredient names
2. Phase 3: ✓ Novel recipe
3. Phase 4: ✓ Standalone (recipe card, no cookbook)
4. Phase 5-7: Standard processing with corrected data
5. Metadata flagged: `"handwritten": true`

**Outcome**: Personal recipe preserved despite handwriting challenges

## Best Practices

### Image Quality
- **Resolution**: 1200px+ width recommended for clear OCR
- **Lighting**: Avoid shadows, glare, or extreme angles
- **Format**: JPEG preferred (convert HEIC first)
- **Focus**: Ensure text is sharp and readable

### Recipe Organization
- Process index pages first (cookbook-index-to-caves)
- Then process recipe pages in order
- Results in integrated graph with page numbers + full content

### Ingredient Naming
- Trust the resolution process - generic/specific pattern works well
- Don't worry about exact matches - bridges handle variations
- Novel ingredients are good (enriches the network)

### Batch Processing
- Process multiple recipes from same cookbook in one session
- Cookbook perspective is reused automatically
- More efficient (single perspective setup)

### Standalone Recipes
- Use "my recipes" perspective for consistency
- Or use primary perspective for personal recipes
- Can always connect to cookbook later if you identify source

## Troubleshooting

### MCP Tool Errors

**Problem**: `caves__my_shadows()` fails
- **Solution**: User needs Caves MCP setup
- **Fix**: Share setup instructions for MCP integration

**Problem**: `caves__search_caves()` returns empty
- **Solution**: Normal for novel ingredients
- **Action**: Continue with extracted names (adds new knowledge)

**Problem**: `caves__connect_caves()` fails
- **Common causes**: Insufficient fire, invalid pId, network error
- **Fix**: Check error message, retry once, report specific failure

### Extraction Issues

**Problem**: Can't extract title
- **Cause**: Non-standard page layout
- **Fix**: Ask user "What is the recipe title?"

**Problem**: Can't find ingredient list
- **Cause**: Handwritten, paragraph format, poor image
- **Fix**: Ask user to provide ingredient list manually

**Problem**: Instructions unclear
- **Cause**: Multi-page, cut off, poor OCR
- **Fix**: Ask user to provide or clarify

### Classification Issues

**Problem**: Ambiguous page type (recipe vs index)
- **Cause**: Minimal text, unusual layout
- **Fix**: Ask user to clarify

**Problem**: Multiple recipes, unclear which to process
- **Cause**: Recipe collection page
- **Fix**: Show detected recipes, ask user to choose

### Performance Issues

**Problem**: Slow ingredient resolution (many searches)
- **Cause**: 15+ ingredients with searches
- **Fix**: Normal, inform user of progress

**Problem**: Large IPFS upload fails
- **Cause**: Multi-page recipe with lots of text
- **Fix**: Proceed without IPFS, offer to retry later

## Summary

This skill provides end-to-end recipe photo processing:

1. ✓ Intelligent classification (recipe vs index)
2. ✓ Structured extraction (title, ingredients, instructions)
3. ✓ Duplicate detection (prevents unwanted duplicates)
4. ✓ Cookbook integration (via perspective-resolver)
5. ✓ Rich ingredient modeling (generic + specific + bridges)
6. ✓ Permanent preservation (IPFS upload)
7. ✓ Knowledge graph connections (recipes ↔ cookbooks ↔ ingredients)

Use this skill to:
- Digitize cookbook recipes with full text preservation
- Build searchable recipe collections
- Organize recipes by ingredients and cookbooks
- Discover recipe connections through shared ingredients
- Preserve family recipes and recipe cards

The result is a rich, interconnected recipe knowledge graph that grows more valuable with each recipe added.
