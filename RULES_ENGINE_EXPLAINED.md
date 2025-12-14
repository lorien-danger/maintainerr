# Rules Engine: Current Problems & Proposed Solution

## The Core Problem: One Field, Two Different Meanings

The current rules engine uses the `operator` field for **two completely different purposes**, which creates massive confusion.

---

## How It Currently Works (Confusingly)

### The Dual-Purpose Operator

```typescript
// Inside a rule object:
{
  operator: "AND",  // But what does this mean???
  firstValue: "plex_addDate",
  action: "BEFORE",
  customValue: "2023-01-01"
}
```

**The `operator` field means different things depending on WHERE the rule appears:**

1. **If it's NOT the first rule in a section** → Combines this rule WITH previous rule in same section
2. **If it's the first rule in a NEW section** → Combines this ENTIRE section with previous section

---

## Counter-Intuitive Example #1: The "Operator Theft"

Let's say you want to find movies that are:
- **(Section 0)** Added before 2023 AND never watched
- **(Section 1)** OR are larger than 50GB

**Your intuitive YAML might look like:**

```yaml
mediaType: movies
rules:
  - 0:
      - firstValue: plex_addDate
        action: BEFORE
        customValue: { type: date, value: "2023-01-01" }
      - operator: AND  # ← Combine with previous rule
        firstValue: plex_viewCount
        action: EQUALS
        customValue: { type: number, value: 0 }
  - 1:
      - operator: OR  # ← I want this section OR'd with section 0!
        firstValue: radarr_sizeOnDisk
        action: BIGGER
        customValue: { type: number, value: 50000000000 }
```

**What you EXPECT:**
```
(addDate before 2023 AND viewCount = 0) OR (size > 50GB)
```

**What ACTUALLY HAPPENS:**

```typescript
// Line 116: sectionActionAnd = +parsedRule.operator === 0;
// The system STEALS the operator from the first rule of Section 1!
// Then Line 118: parsedRule.operator = null;  // WIPES IT OUT!
```

1. Section 1's first rule's `operator: OR` gets **stolen** to mean "combine Section 1 with Section 0 using OR"
2. The operator is then **deleted** from that rule
3. So the actual first rule of Section 1 has `operator: null`!

**Actual result:**
```
Section 0: (addDate before 2023 AND viewCount = 0)
Section 1: (size > 50GB)  ← No operator! Just evaluates standalone
Combined: Section 0 OR Section 1  ← Uses the stolen operator
```

**The logic ends up correct by accident**, but look at how confusing the YAML is!

---

## Counter-Intuitive Example #2: Can't Express Simple Logic

You want to find shows where:
- They have 3 or more seasons
- AND (watched count < 5 OR last watched > 6 months ago)

**In normal programming:**
```javascript
seasons >= 3 && (watchCount < 5 || lastWatched > 6MonthsAgo)
```

**In current YAML, you CANNOT express this directly!**

You'd have to do:

```yaml
rules:
  - 0:  # seasons >= 3
      - firstValue: plex_seasonCount
        action: BIGGER_OR_EQUAL
        customValue: { type: number, value: 3 }
  - 1:  # AND with next section
      - operator: AND  # ← STOLEN for section combination!
        firstValue: tautulli_watchCount
        action: SMALLER
        customValue: { type: number, value: 5 }
      - operator: OR  # Within section
        firstValue: tautulli_lastWatched
        action: BEFORE
        customValue: { type: date, value: "6_months_ago" }
```

**The confusion:**
1. The `operator: AND` on line 1 of Section 1 **doesn't mean** "AND this rule with previous"
2. It means "AND Section 1 with Section 0"
3. But **visually** it looks like it's ANDing the watchCount rule with something
4. Extremely unintuitive!

---

## Counter-Intuitive Example #3: The First Rule Trap

**Scenario:** Find movies that are:
- Added in last 30 days OR have high ratings

```yaml
rules:
  - 0:
      - operator: OR  # ❌ THIS GETS IGNORED!
        firstValue: plex_addDate
        action: IN_LAST
        customValue: { type: number, value: 2592000 }  # 30 days
      - operator: OR
        firstValue: plex_rating
        action: BIGGER
        customValue: { type: number, value: 8.0 }
```

**What you EXPECT:**
```
addDate in last 30 days OR rating > 8.0
```

**What ACTUALLY HAPPENS:**

```typescript
// Line 100-102: Force first rule's operator to null
if ((rule as RuleDbDto)?.id === (rulegroup.rules[0] as RuleDbDto)?.id) {
  parsedRule.operator = null;  // ← YOUR OR GETS DELETED!
}
```

1. **The very first rule's operator is ALWAYS forced to `null`**
2. Your `operator: OR` on the first rule **gets deleted**
3. System treats it as the "starting point" (correct) but your explicit operator is ignored
4. **You can't tell from the YAML** that this rule is special!

**Actual result:**
```
(First rule stands alone, operator ignored)
OR (second rule)
```

It works, but **your first OR was silently ignored** - confusing!

---

## Counter-Intuitive Example #4: Section Numbers Are Opaque

```yaml
rules:
  - 0:
      - firstValue: plex_year
        action: SMALLER
        customValue: { type: number, value: 2000 }
  - 1:  # ← What does "1" mean? Why not "A" and "B"?
      - operator: AND
        firstValue: plex_viewCount
        action: EQUALS
        customValue: { type: number, value: 0 }
  - 2:  # ← Is this AND or OR with sections 0 and 1?
      - operator: OR  # ← You'd think this is OR, but...
        firstValue: radarr_sizeOnDisk
        action: BIGGER
        customValue: { type: number, value: 10000000000 }
```

**Questions users can't answer from the YAML:**
1. Why are sections numbered 0, 1, 2? What do the numbers mean?
2. Is Section 2 OR'd or AND'd with the previous result?
3. The `operator: OR` on Section 2's first rule - is that for combining the section or the rule?

**You have to understand the implementation to know:**
- Numbers are just sequential IDs (meaningless)
- Section 2's `operator: OR` means "Section 2 OR (Section 1 AND Section 0)"
- The structure is **not self-documenting**

---

## The Implementation Issues

### Issue 1: Stateful, Mutable Arrays

```typescript
// Three arrays being juggled:
this.plexData = [/* all media */];
this.workerData = [/* current section matches */];
this.resultData = [/* final combined results */];

// Within section - mutating workerData:
if (rule.operator === OR) {
  this.workerData.push(item);  // Add matches
} else {
  this.workerData.splice(i, 1);  // Remove non-matches
}

// Between sections - mutating resultData:
if (sectionActionAnd) {
  this.resultData = this.resultData.filter(...);  // Complex filtering
} else {
  this.resultData.push(...this.workerData);  // Append
}
```

**Problems:**
- Hard to debug (state changes throughout execution)
- Can't easily replay or test logic
- Performance issues with large arrays (splice in loop!)

### Issue 2: Complex Section Combination Logic

```typescript
// Line 382-391: AND between sections
this.resultData = this.resultData.filter((el) => {
  const plexId = +el.ratingKey;
  // If in current data.. Otherwise we're removing previously added media
  if (plexIdsInCurrentData.has(plexId)) {
    return this.workerPlexIds.has(plexId);
  } else {
    // If not in current data, skip check
    return true;  // ← Why true? To preserve media from other libraries?
  }
});
```

**This comment says it all:** "Otherwise we're removing previously added media"

The logic is trying to handle edge cases that arise from the mutable state design. It's defensive programming because the state management is fragile.

### Issue 3: Operator Overloading Creates Bugs

```typescript
// Line 116: Read operator to determine section combination
sectionActionAnd = +parsedRule.operator === 0;

// Line 118: Then IMMEDIATELY delete it
parsedRule.operator = null;

// This creates a temporal coupling:
// - The order of operations matters
// - If you refactor and accidentally swap these lines, everything breaks
// - The operator field is being mutated, making it hard to trace
```

---

## How Users Actually Think About Rules

When users create rules, they think in **logical expressions**, not sections:

### Mental Model:
```
Find media where:
  (year < 2000 AND viewCount = 0)
  OR
  (sizeOnDisk > 50GB AND rating < 5)
```

### What they want to write:
```json
{
  "match": "any",  // OR at top level
  "conditions": [
    {
      "match": "all",  // AND within group
      "conditions": [
        { "field": "plex.year", "operator": "<", "value": 2000 },
        { "field": "plex.viewCount", "operator": "=", "value": 0 }
      ]
    },
    {
      "match": "all",  // AND within group
      "conditions": [
        { "field": "radarr.sizeOnDisk", "operator": ">", "value": 50000000000 },
        { "field": "plex.rating", "operator": "<", "value": 5 }
      ]
    }
  ]
}
```

**This is:**
- ✅ Self-documenting
- ✅ Unambiguous
- ✅ Arbitrarily nestable
- ✅ Matches how people think
- ✅ Easy to validate

---

## The Better Solution: AST-Based Rules

### 1. **Clear Structure**

```typescript
interface RuleGroup {
  operator: "AND" | "OR";
  conditions: (RuleCondition | RuleGroup)[];  // Recursive!
}

interface RuleCondition {
  field: string;      // "plex.addDate"
  operator: string;   // "before", "equals", "greater_than"
  value: any;         // Actual value to compare
}
```

### 2. **Example: Complex Rule Made Simple**

**Goal:** Find shows where:
- (seasons >= 3 AND episodes >= 20)
- AND (viewCount < 5 OR lastWatched > 6 months ago)
- AND size > 10GB

**Current YAML (confusing):**
```yaml
rules:
  - 0:
      - firstValue: plex_seasonCount
        action: BIGGER_OR_EQUAL
        customValue: { type: number, value: 3 }
      - operator: AND
        firstValue: plex_episodeCount
        action: BIGGER_OR_EQUAL
        customValue: { type: number, value: 20 }
  - 1:
      - operator: AND  # ← Actually means "AND section 1 with section 0"
        firstValue: tautulli_watchCount
        action: SMALLER
        customValue: { type: number, value: 5 }
      - operator: OR
        firstValue: tautulli_lastWatched
        action: BEFORE
        customValue: { type: number, value: 15552000 }
  - 2:
      - operator: AND  # ← Actually means "AND section 2 with previous"
        firstValue: plex_sizeOnDisk
        action: BIGGER
        customValue: { type: number, value: 10000000000 }
```

**New format (clear):**
```json
{
  "operator": "AND",
  "conditions": [
    {
      "operator": "AND",
      "conditions": [
        { "field": "plex.seasonCount", "operator": ">=", "value": 3 },
        { "field": "plex.episodeCount", "operator": ">=", "value": 20 }
      ]
    },
    {
      "operator": "OR",
      "conditions": [
        { "field": "tautulli.watchCount", "operator": "<", "value": 5 },
        { "field": "tautulli.lastWatched", "operator": "before", "value": "6_months" }
      ]
    },
    {
      "field": "plex.sizeOnDisk",
      "operator": ">",
      "value": 10000000000
    }
  ]
}
```

**Visual representation:**
```
AND
├─ AND
│  ├─ seasonCount >= 3
│  └─ episodeCount >= 20
├─ OR
│  ├─ watchCount < 5
│  └─ lastWatched before 6_months
└─ sizeOnDisk > 10GB
```

### 3. **Immutable Evaluation**

```typescript
function evaluateRule(rule: RuleGroup | RuleCondition, media: MediaItem[]): MediaItem[] {
  // Base case: simple condition
  if ('field' in rule) {
    return media.filter(item => {
      const value = getValue(item, rule.field);
      return compare(value, rule.operator, rule.value);
    });
  }

  // Recursive case: group of conditions
  if (rule.operator === 'AND') {
    // Start with all media, progressively filter
    return rule.conditions.reduce((result, condition) => {
      return evaluateRule(condition, result);  // Recursion!
    }, media);
  } else {
    // OR: evaluate each condition separately, then combine
    const results = rule.conditions.map(condition =>
      evaluateRule(condition, media)  // Recursion!
    );
    return uniqueBy(results.flat(), item => item.id);
  }
}
```

**Benefits:**
- ✅ Pure function (no side effects)
- ✅ Immutable (original media array never changes)
- ✅ Testable (easy to write unit tests)
- ✅ Composable (rules can be infinitely nested)
- ✅ Readable (logic is straightforward)

---

## Migration Path

### Phase 1: Add New Format (Backward Compatible)

1. **Accept both formats** in the API
2. **Convert old YAML** to new AST internally
3. **New UI** uses new format
4. Keep old YAML export for backward compatibility

### Phase 2: Deprecate Old Format

1. Show warnings when using old format
2. Add migration tool to convert old rules
3. Documentation updates

### Phase 3: Remove Old Code

1. Delete `rule.comparator.service.ts` (615 lines)
2. Delete section-based logic
3. Keep only AST evaluator (~150 lines)

---

## Real-World Impact

### Before (Current System):

**User posts on Discord:**
> "Why doesn't my rule work? I have Section 0 with OR, then Section 1 with AND, but it's not finding the movies I expect..."

**Response time:** 2-3 hours (need to understand their mental model, explain sections, debug YAML)

### After (New System):

**User posts:**
> "Why doesn't my rule work? Here's my JSON..."

**Response:**
```json
// Oh I see, you have:
{
  "operator": "OR",  // ← This should be "AND"
  "conditions": [...]
}
```

**Response time:** 5 minutes (visual inspection, clear fix)

---

## Code Reduction

| Component | Current | Proposed | Savings |
|-----------|---------|----------|---------|
| Rule comparator | 615 lines | 150 lines | **75% reduction** |
| YAML service | 176 lines | 0 lines* | **100% reduction** |
| State management | 3 arrays + Sets | 0 | **Stateless** |
| Test complexity | High | Low | **Much easier** |

*YAML can still be supported via a thin conversion layer if needed

---

## Summary

### Current System is Broken Because:

1. **Operator overloading** - same field, two meanings
2. **Hidden state** - three arrays being mutated
3. **Non-intuitive** - sections with numbers, stolen operators
4. **Not self-documenting** - requires implementation knowledge
5. **Hard to test** - 615 lines of complex state management
6. **Repetitive** - can't express simple nested logic without multiple sections

### New System Would:

1. **Match user mental model** - direct expression of logic
2. **Be self-documenting** - clear hierarchy visible in JSON
3. **Enable nesting** - arbitrary complexity without sections
4. **Reduce bugs** - immutable, pure functions
5. **Improve performance** - can optimize/parallelize easily
6. **Easier testing** - simple input/output tests

### Bottom Line:

The current system tries to be clever with sections and operator reuse. The new system would be **obvious and straightforward** - which is exactly what you want in a rule engine.
