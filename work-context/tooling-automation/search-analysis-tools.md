# Search Analysis Tools

> **Type**: Developer Tooling / Analysis
> **Stack**: Java/Spring Boot/Vaadin (Lucene), MongoDB Atlas Search, Go
> **Purpose**: Full-text search quality analysis and debugging

---

## Overview

I've built and used tools for analyzing and debugging full-text search systems — primarily Apache Lucene (directly and via Spring Boot) and MongoDB Atlas Search. These tools help me understand why search returns specific results, tune relevance, and debug scoring anomalies.

Search quality is one of those areas where intuition fails: you need tooling to see how analyzers tokenize text, how scoring formulas weight terms, and where relevance breaks down.

---

## Lucene Search Analysis Tool

### Stack
- **Java 17 / Spring Boot 3**: Backend
- **Apache Lucene 9**: Search engine
- **Vaadin 24**: Web UI (pure Java, no JavaScript)
- **Maven**: Build system

### Features

#### 1. Analyzer Comparison View

Side-by-side comparison of how different Lucene analyzers process the same text:

| Analyzer | Input: "The Quick Brown Fox" | Tokens |
|----------|---------------------------|--------|
| Standard | the quick brown fox | [quick, brown, fox] (stops "the") |
| Whitespace | The Quick Brown Fox | [The, Quick, Brown, Fox] |
| Keyword | The Quick Brown Fox | [The Quick Brown Fox] |
| English | The Quick Brown Fox | [quick, brown, fox] (stemmed) |
| Custom | The Quick Brown Fox | (user-defined pipeline) |

This view was critical for debugging relevance issues where I expected a match but the analyzer dropped or transformed the term.

#### 2. Token Stream Inspector

Step-by-step visualization of the Lucene analysis pipeline:

```
Input: "running quickly"
         │
    CharFilter (HTMLStripCharFilter)
         │ → "running quickly"
    Tokenizer (StandardTokenizer)
         │ → ["running", "quickly"]
    TokenFilter (LowerCaseFilter)
         │ → ["running", "quickly"]
    TokenFilter (PorterStemFilter)
         │ → ["run", "quickli"]
    TokenFilter (StopFilter)
         │ → ["run", "quickli"]
```

Each step shows the token stream transformation, so I can see exactly where a token gets modified or dropped.

#### 3. Query Explanation / Score Breakdown

Visual decomposition of BM25 scoring for a query against a document:

```
Score: 2.847 for query "quick fox" against doc #42

  Term: "quick"
    TF (term frequency in doc): 2 → sqrt(2) = 1.414
    IDF (inverse doc frequency): log(1 + (N - n + 0.5) / (n + 0.5)) = 1.234
    Field length norm: 1 / sqrt(fieldLength) = 0.316
    Boost: 1.0
    Term score: 1.414 × 1.234 × 0.316 = 0.551

  Term: "fox"
    TF: 1 → sqrt(1) = 1.0
    IDF: 2.456
    Field length norm: 0.316
    Term score: 1.0 × 2.456 × 0.316 = 0.776

  Combined: 0.551 + 0.776 = 1.327
  Coord factor: 2/2 = 1.0
  Final: 1.327
```

This is invaluable for understanding why document A ranks above document B.

#### 4. Custom Analyzer Builder

A drag-and-drop interface for building custom analyzer pipelines:
- Select CharFilters (HTML strip, pattern replace)
- Select Tokenizer (standard, whitespace, keyword, edge n-gram)
- Add TokenFilters (lowercase, stop words, stemmer, synonym)
- Test with sample text and see results immediately

### Why Vaadin?

I chose Vaadin because:
1. **Pure Java**: No JavaScript, no REST API, no frontend build step
2. **Server-side**: All rendering happens on the server — security is simpler
3. **Rich components**: Data grids, charts, drag-and-drop out of the box
4. **Rapid development**: For an internal tool, Vaadin eliminates an entire layer of complexity

[KEY_POINTS]
- Search analysis requires tooling — intuition alone fails for relevance tuning
- BM25 scoring decomposition reveals why results rank the way they do
- Analyzer comparison catches tokenization issues before they reach production
- Vaadin is ideal for data-heavy internal tools: no frontend complexity
[/KEY_POINTS]

---

## MongoDB Atlas Search Analysis

### Context

On the Rave platform, we use MongoDB Atlas Search for full-text search. I built analysis tools to debug and tune Atlas Search indexes.

### Atlas Search vs. Lucene

Atlas Search is built on Lucene under the hood, but the abstraction layer changes how you interact with it:

| Aspect | Raw Lucene | Atlas Search |
|--------|-----------|--------------|
| Analyzer config | Java code | JSON index definition |
| Scoring | Full BM25 control | Limited score modification |
| Debugging | explain() API | $searchMeta aggregation |
| Custom analyzers | Java classes | JSON analyzer definitions |

### Atlas Search Debugging Tool

I built a Go CLI tool for debugging Atlas Search:

```bash
# Analyze how a query matches against a specific document
atlas-debug explain --index "search_index" --query "quick fox" --doc-id "abc123"

# Compare analyzer output for different index definitions
atlas-debug analyze --text "Running quickly" --analyzer "standard" --analyzer "english"

# Score distribution for a query
atlas-debug scores --index "search_index" --query "nimble storage" --top 100
```

### Index Definition Patterns

I developed patterns for common Atlas Search use cases:

**Multi-field search with boosting**:
```json
{
  "mappings": {
    "dynamic": false,
    "fields": {
      "title": {
        "type": "string",
        "analyzer": "lucene.english",
        "searchAnalyzer": "lucene.english"
      },
      "description": {
        "type": "string",
        "analyzer": "lucene.standard"
      }
    }
  }
}
```

**Autocomplete with edge n-grams**:
```json
{
  "mappings": {
    "fields": {
      "name": [
        { "type": "string", "analyzer": "lucene.standard" },
        { "type": "autocomplete", "tokenization": "edgeGram", "minGrams": 2, "maxGrams": 10 }
      ]
    }
  }
}
```

[COMMON_MISTAKES]
- Assuming Atlas Search analyzer names map 1:1 to Lucene analyzers — they don't always
- Not using `searchAnalyzer` separately from `analyzer` — search-time analysis should differ from index-time
- Ignoring score normalization: Atlas Search scores aren't comparable across different queries
- Not testing with production-like data: Analyzer behavior changes with corpus statistics (IDF depends on document count)
[/COMMON_MISTAKES]

---

## Search Relevance Tuning Workflow

My approach to tuning search relevance:

1. **Gather failure cases**: Collect queries where users didn't find what they expected
2. **Analyze tokenization**: Run both the query and expected document through the analyzer comparison tool
3. **Check scoring**: Use explain/score breakdown to understand ranking
4. **Hypothesize**: Identify the gap (missing synonym, over-aggressive stemming, field boost imbalance)
5. **Test fix**: Modify analyzer/index definition, re-test failure cases
6. **Regression check**: Verify fix doesn't degrade other queries
7. **Deploy**: Roll out index changes (Atlas Search supports zero-downtime index updates)

---

## Interview Talking Points

**"Tell me about debugging a search quality issue."**
I built a Lucene analysis tool that decomposes BM25 scoring and visualizes analyzer tokenization. When a customer reported missing search results, I traced the issue to the English analyzer stemming "running" to "run" but the query analyzer not stemming "runner" to the same root. The fix was adding a synonym filter to map related forms.

**"How do you approach search relevance?"**
I use a systematic workflow: collect failure cases, analyze tokenization, decompose scoring, hypothesize, test, and regression check. The key insight is that search quality is measurable — you can build test suites of expected query-result pairs and run them against index changes.

**"What's BM25 and why does it matter?"**
BM25 is the scoring algorithm used by Lucene (and by extension, Elasticsearch, Atlas Search, etc.). It balances term frequency (how often the term appears in the document) with inverse document frequency (how rare the term is across all documents) and field length normalization. Understanding BM25 components is essential for explaining why document A ranks above document B.

[FOLLOW_UP]
- How would you implement semantic search (vector similarity) alongside keyword search?
- What's the difference between BM25 and TF-IDF?
- How do you handle multilingual search?
- When would you use Elasticsearch vs. Atlas Search vs. raw Lucene?
[/FOLLOW_UP]

---

## Cross-References

- See [Side Projects → Other Projects](../side-projects/other-projects.md) for the Lucene Search Analysis project details
- See [Side Projects → DiceDB](../side-projects/dicedb.md) for another database internals project
- See [Tooling → Utility Scripts](utility-scripts.md) for the Atlas Search CLI tool context
- See [Nimble Download → Global Trade Compliance](../nimble-download/global-trade-compliance.md) for Go tooling patterns

---

*Last updated: Auto-generated from work-context-refresh skill*
