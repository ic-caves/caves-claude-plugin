---
name: content-to-caves
description: Analyze text content and generate knowledge graph connections using AI-powered 4-phase analysis. Extracts entities, validates against existing knowledge, and creates structured connections in Caves. Use when user wants to analyze transcripts, articles, essays, or text files to build knowledge graphs.
---

# Content to Caves: Auto-Tagging Skill

## Overview

This skill transforms text content (transcripts, articles, essays, long-form writing) into structured knowledge graph connections using a sophisticated 4-phase analysis methodology:

1. **Phase 1: Extract Connections** - Deep analysis to identify 30-40 key entities and relationships
2. **Phase 2: Entity Resolution & Validation** - Search existing knowledge, create bridges to related concepts
3. **Phase 3: Iterative Discovery** - Run 2 additional analysis rounds with different analytical lenses
4. **Phase 4: Cross-Perspective Integration** - Find reinforcements and bridges from related perspectives

The result is a rich, interconnected knowledge graph that builds on existing knowledge while introducing new concepts.

## When to Invoke This Skill

Use this skill when the user wants to:
- Analyze a transcript (podcast, interview, lecture, speech)
- Process an article or essay into knowledge graph connections
- Build their knowledge graph from long-form text content
- Extract structured knowledge from documents or text files
- Connect new content to their existing Caves knowledge network

**Keywords that should trigger this skill:**
- "analyze this transcript/article/content"
- "add to caves" (with text content)
- "build knowledge graph from this text"
- "extract connections from this content"
- "process this into caves"

## Prerequisites

Before starting, verify these requirements are met:

1. **Caves Account**: User must have an account at itscaves.com
2. **MCP Access**: Verify with `caves__my_shadows()` - should return user's perspectives
3. **Content Ready**: User has text content via file path (use Read tool) or direct paste
4. **Fire Balance**: User needs fire balance to create connections (warn if this is their first use)

If MCP tools are not accessible, inform the user they need to set up Caves MCP integration first.

## Workflow Instructions for Claude

Follow these steps to execute the 4-phase analysis:

### Step 1: Content Acquisition

**Objective**: Get the text content to analyze.

**Instructions**:
1. Accept text content in these ways:
   - File path: Use Read tool to load the file
   - Direct paste: User provides text directly in chat
   - URL: Guide user to use WebFetch first to extract content, then provide the text
   - Caves IPFS: Content retrieved from Caves using caves__read_ipfs tool

2. Validate content length:
   - Minimum: ~500 words (too short won't yield good results)
   - Maximum: ~50,000 characters (beyond Claude context limit)
   - Sweet spot: 2,000-20,000 words

3. Optional IPFS upload:
   - **SKIP this step if content was retrieved from Caves** (via caves__read_ipfs or similar - it's already uploaded!)
   - **Only ask** if content came from file path, direct paste, or URL
   - Ask: "Would you like to upload this to IPFS for permanent storage?"
   - If yes: Note for Step 10 (execute after creating connections)
   - Benefits: Content becomes permanent part of knowledge graph

**Example (when content came from file/paste/URL)**:
```
Claude: "I'll analyze this content for your Caves knowledge graph.
        I see you have a 5,000-word transcript.

        Would you like to upload this to IPFS for permanent storage?
        This makes it a permanent part of your knowledge graph."
```

**Example (when content came from Caves IPFS)**:
```
Claude: "I'll analyze this content for your Caves knowledge graph.
        I see you have a 5,000-word article (already stored in IPFS)."

        [Skip IPFS upload question - content is already uploaded]
```

### Step 2: Execution Mode Selection

**Objective**: Determine how autonomous the analysis should be.

**Instructions**:
1. Present two options to the user:
   - **Analyze-only**: Show all suggestions first, get approval before executing (safer, recommended for first use)
   - **Analyze-and-execute**: Run full analysis and create connections autonomously (faster, for experienced users)

2. Set expectations:
   - Analysis may take 2-3 minutes for long content
   - Analyze-and-execute is faster but creates connections immediately
   - Analyze-only gives user control over what gets created

3. Store the user's choice for later steps

**Example**:
```
Claude: "How would you like to proceed?

        a) Analyze-only (recommended): I'll show you all suggestions and
           get your approval before creating any connections.

        b) Analyze-and-execute: I'll run the full analysis and create
           connections automatically (faster, but less control).

        Note: Analysis may take 2-3 minutes for this content length."
```

### Step 3: Resolve or Create Perspective

**Objective**: Determine which perspective (pId) to use for these connections.

**Use the `perspective-resolver` skill to handle perspective management.**

This perspective represents the content/article itself. The pId should be a child of key identifiers like author name and/or source URL.

#### Extract Identifying Information

1. **Analyze first 2,000 characters** to identify:
   - **Author/speaker name**: Look for attribution patterns
     - "by [name]", "- [name]", "written by [name]"
     - Self-references: "I am [name]", "my name is [name]"
     - Interview format: "Interviewer: ... Guest: [name]"
   - **Source URL**: If content has a canonical source URL
   - Convert names to lowercase for tagging

2. **Determine identifying tags** (choose appropriate combination):
   - **With author + URL**: `[author_name, source_url]`
   - **With just author**: `[author_name]`
   - **With just URL**: `[source_url]`
   - Use the most specific combination available

#### Invoke perspective-resolver Skill

**Example with author and URL:**
```javascript
// Invoke: perspective-resolver skill
{
  primary_tag: author_name,  // e.g., "david graeber"
  identifying_tags: [author_name, source_url],  // e.g., ["david graeber", "https://theanarchistlibrary.org/article"]
  taxonomy: [
    {parent: "articles", child: source_url},
    {parent: "thing", child: "{pId}"}
  ],
  entity_type: "article",
  entity_description: `article by ${author_name}`
}
```

**Example with just author (no URL):**
```javascript
// Invoke: perspective-resolver skill
{
  primary_tag: author_name,  // e.g., "david graeber"
  identifying_tags: [author_name],
  taxonomy: [
    {parent: "thing", child: "{pId}"}
  ],
  entity_type: "content",
  entity_description: `content by ${author_name}`
}
```

The perspective-resolver skill will:
- Search for existing perspectives using the identifying tags
- Present findings to user if matching perspective exists
- Handle user choice (reuse existing vs create new)
- Create all initial connections (identifying tags, taxonomy)
- Return the pId to use

**Store the returned pId** for subsequent analysis phases (Step 4 onwards).

### Step 4: Phase 1 - Extract Connections

**Objective**: Perform deep analysis to extract 30-40 initial connections.

**Analysis Prompt (for yourself)**:
```
Task: Analyze this text to extract key entities and their relationships.

Instructions:
1. Read the entire text carefully, noting key themes and concepts

2. Identify entities in these categories:
   - People: Authors, speakers, historical figures, referenced individuals
   - Concepts: Theories, frameworks, ideas, movements, schools of thought
   - Events: Historical events, contemporary situations, case studies
   - Organizations: Institutions, groups, movements, companies

3. For each pair of related entities, determine the relationship:
   - Which entity is MORE GENERAL (parent)?
   - Which entity is MORE SPECIFIC (child)?
   - Format: "parent ← child" (general ← specific)

4. Classify each connection by pattern:
   - THEORY_APPLICATION: theory ← example/application
   - THINKER_CONCEPT: thinker ← their concept/idea
   - HISTORICAL_CONTEMPORARY: historical event ← modern parallel
   - ORGANIZATION_MEMBER: organization ← member/part
   - METAPHOR_TARGET: metaphor ← what it describes
   - CAUSE_EFFECT: cause ← effect/result
   - CATEGORY_INSTANCE: category ← specific instance

5. Include evidence: Direct quote from text (100-200 characters)

6. Target: Extract 30-40 high-quality connections

Output format for each connection:
{
  parent: "capitalism",
  child: "gig economy",
  pattern: "CATEGORY_INSTANCE",
  evidence: "The gig economy represents a new phase of capitalist labor relations..."
}
```

**Quality Criteria**:
- Parent should be more general/abstract than child
- Connection should be clearly supported by text evidence
- Avoid obvious relationships (e.g., "philosophy ← philosophical concept")
- Prefer specific, meaningful connections over generic ones

**Store results** in a structured list for subsequent phases.

### Step 5: Phase 2 - Entity Resolution & Validation

**Objective**: Search for each entity in existing Caves knowledge, create bridges where relevant.

**Instructions for each unique entity from Phase 1**:

1. **Search Caves**:
   ```
   Call: caves__search_caves(query=entity_name)
   ```

2. **Evaluate search results** (top 5):
   - Are these results semantically related to our entity?
   - Score each result (0-10):
     - 10: Exact match (same concept, different wording)
     - 7-9: Very related (should bridge)
     - 4-6: Somewhat related (maybe bridge)
     - 0-3: Unrelated

3. **Decide on canonical name**:
   - If exact match found (score 10): Use that tag name instead
   - Otherwise: Keep our extracted entity name

4. **Create bridge connections**:
   - For results with score ≥ 7:
     - Validate: Does this bridge make sense given source text?
     - Create: `our_entity ← search_result_tag`
     - Add to suggestions as Phase 2 connection
     - Mark with "bridge" metadata

5. **Track validation statistics**:
   - Entities searched: X
   - Entities with results: Y
   - Canonical names used: Z
   - Bridge connections created: W

**Example**:
```
Entity: "gig economy"
Search results:
  1. "platform economy" (score 9) → Create bridge: "gig economy ← platform economy"
  2. "uber" (score 8) → Create bridge: "gig economy ← uber"
  3. "freelance work" (score 7) → Create bridge: "gig economy ← freelance work"
  4. "economy" (score 3) → Skip (too general)
```

**Performance optimization**:
- Cache search results during this phase
- Batch similar entity searches if possible

### Step 6: Phase 3 - Iterative Discovery

**Objective**: Run 2 additional analysis rounds with different analytical lenses to discover 40-60 more connections.

**Round 2 Analysis Prompt**:
```
Task: Re-analyze the text with focus on supporting details and implications.

Instructions:
1. Look for connections you missed in Phase 1:
   - What examples illustrate the main concepts?
   - What supporting evidence or case studies are mentioned?
   - What implications or consequences are discussed?
   - What nuanced relationships exist between entities?

2. Extract 20-30 new connections (avoid duplicating Phase 1)

3. Use the same format and patterns as Phase 1

4. Include evidence quotes for each connection
```

**Round 3 Analysis Prompt**:
```
Task: Re-analyze the text with focus on broader context and applications.

Instructions:
1. Look for higher-level connections:
   - What related domains or fields connect to these ideas?
   - What historical or contemporary parallels are relevant?
   - How do these concepts apply in different contexts?
   - What cross-domain relationships exist?

2. Extract 20-30 new connections (avoid duplicating Phase 1 and Round 2)

3. Use the same format and patterns as Phase 1

4. Include evidence quotes for each connection
```

**Track metadata** for each connection:
- Phase: 3
- Round: 2 or 3
- Discovery: "iterative"

**Quality check**:
- Ensure no duplicates across all rounds
- Verify each connection is supported by text
- Aim for 40-60 additional connections total from both rounds

### Step 7: Phase 4 - Cross-Perspective Integration

**Objective**: Find reinforcements and bridges from related perspectives' knowledge graphs.

**Instructions**:

1. **Collect all unique entities** from Phases 1-3 (both parents and children)

2. **Find related perspectives**:
   ```
   Call: caves__getPidsForTags(tags=[entity_list], matchMode='any')
   ```
   - This returns perspectives that have worked with similar concepts
   - Limit to top 5 related perspectives (by connection count)

3. **For each related perspective**:

   a. **Fetch their knowledge graph**:
      ```
      Call: caves__my_shadow_caves(pId=related_pId, format='md')
      ```
      - Use 'md' format for efficiency (compressed hierarchical structure)

   b. **Analyze their connections**:
      - Parse the markdown structure into parent-child pairs

      - **Look for REINFORCEMENTS** (exact matches):
        - Do they have the same parent ← child connection?
        - If yes: Mark our suggestion as "reinforced"
        - Track: Which pId reinforced it, increment reinforcement count

      - **Look for BRIDGES** (semantically related):
        - Do they have connections to entities related to ours?
        - For each potential bridge:
          - Score semantic similarity (0-10)
          - If score ≥ 7: Validate against source text
          - Validation: Does this bridge connect meaningfully to our content?
          - If valid: Add as Phase 4 suggestion
          - Include provenance: `bridge_from_pId: related_pId`

4. **Track cross-perspective statistics**:
   - Perspectives analyzed: X
   - Reinforcements found: Y (count of suggestions reinforced)
   - Reinforcement instances: Z (total reinforcement count)
   - Bridges found: W
   - Bridges validated: V

**Example**:
```
Related perspective: "alice_economics" (25 shared entities)

Their graph includes:
- "capitalism ← gig economy" → REINFORCEMENT! (matches our suggestion)
- "gig economy ← airbnb" → BRIDGE candidate (airbnb not in our text)
  - Validate: Our text discusses "sharing economy platforms"
  - Result: Valid bridge! Add "gig economy ← airbnb" as Phase 4 suggestion

Metadata for reinforced suggestion:
{
  parent: "capitalism",
  child: "gig economy",
  pattern: "CATEGORY_INSTANCE",
  phase: 1,
  reinforced_by: ["alice_economics"],
  reinforcement_count: 1
}
```

**Performance considerations**:
- Limit to top 5 related perspectives (more is diminishing returns)
- Use markdown format for faster parsing
- Cache parsed graphs during this phase

### Step 8: Present Results

**Objective**: Show user comprehensive analysis results before execution.

**Format**:
```
Analysis Complete!

Summary:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Total Connections: 127

By Phase:
  Phase 1 (Extracted):           38 connections
  Phase 2 (Validated):           15 connections
  Phase 3 (Iterative):           42 connections
    - Round 2:                   24 connections
    - Round 3:                   18 connections
  Phase 4 (Cross-Perspective):   32 connections

By Pattern:
  THEORY_APPLICATION:            28 connections
  THINKER_CONCEPT:               19 connections
  CATEGORY_INSTANCE:             34 connections
  HISTORICAL_CONTEMPORARY:       12 connections
  ORGANIZATION_MEMBER:           11 connections
  METAPHOR_TARGET:                8 connections
  CAUSE_EFFECT:                  15 connections

Cross-Perspective Insights:
  Perspectives analyzed:          3 related perspectives
  Reinforcements:                18 connections validated by others
  Bridges:                        32 new connections discovered
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Top Connections (showing 15 of 127):

1. capitalism ← gig economy
   Pattern: CATEGORY_INSTANCE | Phase: 1
   Evidence: "The gig economy represents a new phase of capitalist labor relations..."
   ⭐ Reinforced by 2 perspectives (alice_economics, bob_politics)

2. david graeber ← debt the first 5000 years
   Pattern: THINKER_CONCEPT | Phase: 1
   Evidence: "In his groundbreaking work Debt: The First 5,000 Years, Graeber argues..."
   ⭐ Reinforced by 1 perspective (alice_economics)

3. neoliberalism ← privatization
   Pattern: THEORY_APPLICATION | Phase: 1
   Evidence: "Privatization became the hallmark of neoliberal economic policy..."

4. sharing economy ← airbnb
   Pattern: CATEGORY_INSTANCE | Phase: 4 (bridge from alice_economics)
   Evidence: [Bridge connection from related perspective]

[... 11 more connections ...]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

[Full list available upon request]
```

**Key elements to include**:
1. Total connection count
2. Breakdown by phase (show work done in each phase)
3. Breakdown by pattern (show which types dominate)
4. Cross-perspective insights (reinforcements and bridges)
5. Top 10-15 connections with evidence
6. Note if full list is available

**Visual enhancements**:
- Use emojis sparingly (⭐ for reinforced connections)
- Use box drawing characters for separation
- Indent hierarchically for readability
- Bold important numbers

### Step 9: Execution Decision (if analyze-only mode)

**Objective**: Get user approval and optional filtering before creating connections.

**Instructions**:

1. **Initial prompt**:
   ```
   Claude: "Ready to create these connections in Caves?

           Options:
           a) Execute all connections (recommended)
           b) Filter before executing
           c) Save suggestions without executing (export to file)
           d) Cancel

           Note: Creating connections requires fire balance.
                 You'll use approximately [X] fire."
   ```

2. **If user chooses filtering** (option b):

   **Filtering options**:
   ```
   How would you like to filter?

   1. By Phase:
      - Exclude Phase 4 (cross-perspective bridges)
      - Exclude Phase 3 (iterative discovery)
      - Include only Phase 1+2 (core extractions)

   2. By Pattern:
      - Include only specific patterns (select from list)
      - Exclude specific patterns

   3. By Reinforcement:
      - Include only reinforced connections (higher confidence)
      - Exclude unreinforced connections

   4. Manual Selection:
      - Review full list and select specific connections
   ```

   **After filtering**:
   - Show updated count
   - Show summary of what was filtered out
   - Confirm before proceeding

3. **If user chooses save** (option c):
   - Format connections as JSON
   - Save to file: `caves-analysis-[timestamp].json`
   - User can load and execute later

4. **Final confirmation**:
   ```
   Claude: "Ready to create [X] connections in Caves using perspective '[username]'.

           This will:
           - Create [X] new connections in your knowledge graph
           - Cost approximately [X] fire
           - Be immediately visible at https://itscaves.com/c/[username]

           Proceed?"
   ```

### Step 10: Connection Creation

**Objective**: Create all approved connections in Caves.

**Instructions**:

1. **Prepare batch connections**:
   ```javascript
   // Transform suggestions into connection format
   connections = approved_suggestions.map(suggestion => ({
     parent: suggestion.parent,
     child: suggestion.child,
     value: true  // or 1 for YES vote
   }))
   ```

2. **Execute batch creation**:
   ```
   Call: caves__connect_caves(pId=selected_pId, connections=connections)
   ```
   - This creates all connections in a single API call
   - Efficient even for 100+ connections
   - Returns count of new vs existing connections

3. **Optional: IPFS Upload** (if requested in Step 1 AND content not already from Caves):

   **IMPORTANT**: Skip this entire step if content was retrieved from Caves (e.g., via caves__read_ipfs) - it's already uploaded!

   a. Upload content:
      ```
      Call: caves__upload_text(
        text=original_content,
        pId=selected_pId,
        metadata={
          title: extracted_title,
          source: "content-to-caves skill",
          author: speaker_name,
          analysis_date: current_date
        }
      )
      ```

   b. Get IPFS CID from response

   c. Create metadata connection:
      ```
      Call: caves__connect_caves(
        pId=selected_pId,
        connections=[{
          parent: selected_pId,
          child: `/ipfs/${ipfsCid}`,
          value: true
        }]
      )
      ```

4. **Report results**:
   ```
   ✓ Analysis Complete!

   Results:
   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   Connections Created:        127 total
     - New connections:        119
     - Already existed:          8

   [If IPFS upload]
   Content Uploaded:           /ipfs/bafkreif...
     - Size:                   15.2 KB
     - Metadata:               Connected to perspective
   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

   View your knowledge graph:
   https://itscaves.com/c/[url-encoded-username]

   Your knowledge graph now includes:
   - [X] entities from this content
   - [Y] connections to existing knowledge
   - [Z] reinforcements from [N] other perspectives
   ```

5. **Follow-up suggestions**:
   ```
   Next steps:
   - Explore your graph at Caves to see connections
   - Analyze related content to build on this topic
   - Review cross-perspective bridges for insights
   ```

## Connection Pattern Reference

### 1. THEORY_APPLICATION
**Definition**: General theory or framework ← specific application or example

**Examples**:
- `supply and demand ← uber surge pricing`
- `game theory ← prisoner's dilemma`
- `behavioral economics ← nudge theory`

**When to use**: When content discusses how abstract theories apply in concrete situations.

### 2. THINKER_CONCEPT
**Definition**: Person (author, thinker, researcher) ← concept they created or are known for

**Examples**:
- `karl marx ← dialectical materialism`
- `daniel kahneman ← thinking fast and slow`
- `david graeber ← debt the first 5000 years`

**When to use**: When attributing ideas, theories, or works to their originators.

### 3. HISTORICAL_CONTEMPORARY
**Definition**: Historical event or concept ← modern parallel or contemporary example

**Examples**:
- `enclosure movement ← gentrification`
- `tulip mania ← cryptocurrency bubble`
- `guilds ← professional associations`

**When to use**: When content draws parallels between past and present.

### 4. ORGANIZATION_MEMBER
**Definition**: Organization, institution, or group ← member or component

**Examples**:
- `european union ← france`
- `anarchist movement ← david graeber`
- `chicago school ← milton friedman`

**When to use**: When discussing membership, affiliation, or organizational structure.

### 5. METAPHOR_TARGET
**Definition**: Metaphor or analogy ← what it describes or refers to

**Examples**:
- `invisible hand ← market self-regulation`
- `tragedy of the commons ← resource depletion`
- `creative destruction ← innovation disrupting industries`

**When to use**: When content uses metaphorical language to explain concepts.

### 6. CAUSE_EFFECT
**Definition**: Cause or condition ← effect or result

**Examples**:
- `inflation ← purchasing power decline`
- `automation ← job displacement`
- `deregulation ← financial crisis`

**When to use**: When content describes causal relationships or consequences.

### 7. CATEGORY_INSTANCE
**Definition**: General category ← specific instance or member

**Examples**:
- `economic systems ← capitalism`
- `labor arrangements ← gig economy`
- `financial instruments ← derivatives`

**When to use**: When content discusses taxonomies, classifications, or "is-a" relationships.

## Best Practices

### Content Selection
- **Ideal length**: 2,000-20,000 words (5,000-10,000 is sweet spot)
- **Content type**: Structured content works best (articles, transcripts, essays)
- **Avoid**: Chat logs, unstructured notes, lists without context
- **Multiple documents**: Run separately for better topic coherence

### Perspective Management
- **Reuse when possible**: Prefer reusing existing perspectives over creating new ones
- **Topic-based**: Create new perspectives for distinct topic areas or speakers
- **Author-based**: When analyzing works by specific thinkers, create perspective per author
- **Project-based**: Consider creating perspectives for research projects or themes

### Analysis Quality
- **Phase 1**: Aim for depth over breadth (30-40 high-quality connections)
- **Phase 2**: Be conservative with bridges (only clear semantic matches)
- **Phase 3**: Use distinct analytical lenses (don't just repeat Phase 1)
- **Phase 4**: Validate bridges rigorously against source text

### Execution Strategy
- **First time**: Use analyze-only mode to understand output quality
- **Experienced**: Use analyze-and-execute for speed
- **Experimental content**: Use analyze-only to filter before committing
- **Trusted content**: Use analyze-and-execute with full confidence

### Cross-Perspective Integration
- **Review carefully**: Phase 4 bridges may not always be relevant
- **Reinforcements**: High-confidence signals (prioritize these)
- **Bridges**: Validate these against your source text
- **Filtering**: Consider excluding Phase 4 for very specialized content

## Troubleshooting

### MCP Tool Errors

**Problem**: `caves__my_shadows()` fails or returns error
- **Solution**: User needs to set up Caves MCP integration
- **Check**: Verify user has Caves account and MCP configured
- **Workaround**: None - MCP access is required

**Problem**: `caves__search_caves()` returns empty results
- **Solution**: This is not an error - entity may be novel to the network
- **Action**: Continue with original entity name
- **Note**: Document as potential new concept for the network

**Problem**: `caves__connect_caves()` fails
- **Common causes**:
  - Insufficient fire balance
  - Invalid pId
  - Malformed connection data
- **Solution**:
  - Check error message for specific cause
  - Retry once with corrected data
  - If still fails, report which connections failed and continue with rest

### Analysis Issues

**Problem**: Too few connections extracted (< 20 in Phase 1)
- **Cause**: Content may be too short or lack conceptual depth
- **Solution**:
  - Verify content length (should be 1,000+ words)
  - Try with longer or more substantive content
  - Lower expectations for short content

**Problem**: Too many connections extracted (> 100 in Phase 1)
- **Cause**: Content is very dense or analysis was too liberal
- **Solution**:
  - This is okay! Proceed with all phases
  - User can filter before execution
  - Document high-density content for future reference

**Problem**: Phase 4 finds no related perspectives
- **Cause**: Content covers novel topics or user has small network
- **Solution**:
  - This is normal for new topics
  - Skip Phase 4 gracefully
  - Note that future analyses will benefit from this baseline

### Perspective Matching Issues

**Problem**: Cannot identify speaker/author
- **Cause**: Content lacks clear attribution
- **Solution**:
  - Ask user: "Who is the author/speaker?"
  - Use generic username: "imported_content_[date]"
  - Or create topic-based username: "[topic]_collection"
  - Invoke perspective-resolver skill with the chosen identifier

**Problem**: Perspective creation fails
- **Cause**: Username already taken or invalid format
- **Solution**: The perspective-resolver skill handles this automatically by:
  - Suggesting alternative usernames
  - Offering to use existing perspective
  - Getting user input for resolution

### Performance Issues

**Problem**: Analysis takes too long (> 5 minutes)
- **Cause**: Content is very long or Phase 4 is slow
- **Solution**:
  - Inform user of progress at each phase
  - Consider reducing Phase 4 perspectives (limit to top 3)
  - For very long content, suggest splitting into sections

**Problem**: Rate limit errors
- **Cause**: Too many API calls in short period
- **Solution**:
  - Implement exponential backoff
  - Reduce Phase 4 perspectives analyzed
  - Wait and retry

### Validation Issues

**Problem**: Bridge connections don't validate well
- **Cause**: Semantic similarity doesn't equal textual relevance
- **Solution**:
  - Be more conservative with bridge validation
  - Require explicit textual support
  - When in doubt, exclude the bridge

**Problem**: Reinforcements seem incorrect
- **Cause**: Tag name matching is too loose
- **Solution**:
  - Use exact string matching for reinforcements
  - Normalize whitespace and case before comparing
  - Document false positives for improvement

## Advanced Usage

### Batch Processing
To analyze multiple related documents:
1. Run skill separately for each document
2. Use same perspective for related content
3. Phase 4 will automatically discover connections between documents
4. Result: Integrated knowledge graph across all content

### Custom Filtering
For specialized use cases:
1. Use analyze-only mode
2. Export suggestions to JSON file
3. Apply custom filters (scripts, manual review)
4. Load filtered suggestions back
5. Execute via MCP tools directly

### Integration with Other Skills
This skill can work with other Caves skills:
- Use cookbook-index-to-caves for recipe indexing
- Use content-to-caves for article analysis
- Results integrate in unified knowledge graph

## Examples

### Example 1: Podcast Transcript

**Input**: 8,000-word transcript of economics podcast

**Process**:
1. Content acquisition: Read from file
2. Execution mode: Analyze-only (first use)
3. Perspective: Used perspective-resolver skill → found existing "david graeber" perspective
4. Phase 1: Extracted 42 connections (theories, concepts, examples)
5. Phase 2: Found 18 related entities, created 12 bridges
6. Phase 3: Discovered 38 additional connections (rounds 2 & 3)
7. Phase 4: Analyzed 4 related perspectives, found 15 reinforcements, 22 bridges
8. Results: 112 total connections
9. Execution: User approved all, created successfully

**Outcome**: Rich knowledge graph connecting economic theories to contemporary examples, reinforced by existing network knowledge.

### Example 2: Research Article

**Input**: 12,000-word academic article on labor economics

**Process**:
1. Content acquisition: Pasted directly
2. Execution mode: Analyze-and-execute (experienced user)
3. Perspective: Used perspective-resolver skill → created new "labor economics research"
4. Phase 1: Extracted 38 connections (academic concepts, studies, theories)
5. Phase 2: Found 24 related entities, created 15 bridges
6. Phase 3: Discovered 44 additional connections
7. Phase 4: Analyzed 5 related perspectives, found 28 reinforcements, 31 bridges
8. Execution: Automatically created 157 connections

**Outcome**: Comprehensive academic knowledge graph integrated with existing perspectives' economics knowledge.

### Example 3: Historical Essay

**Input**: 5,000-word essay on medieval economics

**Process**:
1. Content acquisition: Read from file, uploaded to IPFS
2. Execution mode: Analyze-only
3. Perspective: Used perspective-resolver skill → created new "medieval economics"
4. Phase 1: Extracted 31 connections (historical events, institutions, practices)
5. Phase 2: Found 8 related entities (many historical terms were novel)
6. Phase 3: Discovered 35 additional connections (historical parallels)
7. Phase 4: Found 0 related perspectives (novel topic area)
8. Execution: User filtered to exclude Phase 3 round 3, created 58 connections

**Outcome**: Historical knowledge graph established as foundation; future analyses will benefit from Phase 4 integration.

## Summary

This skill implements a sophisticated 4-phase knowledge graph construction methodology:

1. **Phase 1**: Deep extraction of core concepts and relationships
2. **Phase 2**: Validation and bridge-building with existing knowledge
3. **Phase 3**: Iterative discovery with multiple analytical lenses
4. **Phase 4**: Cross-perspective integration for reinforcement and discovery

The result is a rich, interconnected knowledge graph that:
- ✓ Captures deep conceptual relationships from source text
- ✓ Connects to existing knowledge network
- ✓ Discovers novel connections through iteration
- ✓ Leverages collective intelligence via cross-perspective analysis

Use this skill to transform unstructured text into structured knowledge, building a comprehensive understanding that grows more valuable with each analysis.
