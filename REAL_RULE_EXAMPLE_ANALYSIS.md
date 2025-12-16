# Analysis of Real-World Rule: The Duplication Problem

## Your Original Rule

```yaml
mediaType: SEASONS
rules:
  - "0":
      - firstValue: Sonarr.seasonNumber
        action: NOT_EQUALS
        customValue:
          type: number
          value: 1
      - operator: OR
        firstValue: Sonarr.rating
        action: SMALLER
        customValue:
          type: number
          value: 7.5
  - "1":
      - operator: AND
        firstValue: Tautulli.addDate
        action: BEFORE
        customValue:
          type: custom_days
          value: "30"
      - operator: AND
        firstValue: Tautulli.sw_watchers
        action: COUNT_EQUALS
        customValue:
          type: number
          value: 0
      - operator: OR
        firstValue: Tautulli.lastViewedAt
        action: BEFORE
        customValue:
          type: custom_days
          value: "59"
  - "2":
      - operator: OR   # ← Section combiner (OR with previous)
        firstValue: Sonarr.seasonNumber
        action: NOT_EQUALS
        customValue:
          type: number
          value: 1
      - operator: AND
        firstValue: Tautulli.sw_watchers
        action: CONTAINS
        lastValue: Tautulli.sw_allEpisodesSeenBy
      - operator: OR
        firstValue: Sonarr.rating
        action: SMALLER
        customValue:
          type: number
          value: 7.5
      - operator: AND
        firstValue: Tautulli.sw_allEpisodesSeenBy
        action: CONTAINS_ALL
        lastValue: Tautulli.sw_watchers
      - operator: AND
        firstValue: Sonarr.unaired_episodes
        action: EQUALS
        customValue:
          type: boolean
          value: "false"
```

## 🔴 The Smoking Gun: DUPLICATION

**Notice these conditions appear TWICE:**

```yaml
# In Section 0:
- seasonNumber != 1
- OR rating < 7.5

# In Section 2:
- seasonNumber != 1    # ← DUPLICATED!
- ...
- OR rating < 7.5      # ← DUPLICATED!
```

**This duplication is a RED FLAG that you're fighting the system!**

---

## What The System Actually Evaluates

Let me trace through the execution step-by-step:

### Section 0 (First section, no combining)
```
Result: seasonNumber != 1 OR rating < 7.5
```
Matches: Any season except season 1, OR any season with rating < 7.5

### Section 1 (AND with Section 0)
```
Within section: (addDate < 30 days ago AND watchers = 0) OR lastViewed < 59 days ago
```

**Combined with Section 0 using AND:**
```
(seasonNumber != 1 OR rating < 7.5)
AND
((addDate < 30 days AND watchers = 0) OR lastViewed < 59 days)
```

Let's call this **Result A**.

### Section 2 (OR with Result A)

**Within Section 2:**
```
(seasonNumber != 1 AND watchers ⊇ allEpisodesSeenBy)
OR
(rating < 7.5 AND allEpisodesSeenBy ⊇ watchers AND unaired = false)
```

Let's call this **Result B**.

### Final Result
```
Result A OR Result B
```

Which expands to:
```
[
  (seasonNumber != 1 OR rating < 7.5)
  AND
  ((addDate < 30 days AND watchers = 0) OR lastViewed < 59 days)
]
OR
[
  (seasonNumber != 1 AND watchers ⊇ allEpisodesSeenBy)
  OR
  (rating < 7.5 AND allEpisodesSeenBy ⊇ watchers AND unaired = false)
]
```

---

## Why You Had To Structure It This Way

### The Problem You Faced

I suspect you wanted to express something like:

**"Find seasons that match these watching patterns, but prioritize excluding season 1 and low-rated seasons"**

But you **couldn't express nested conditions** in a natural way, so you had to:

1. ✅ Use Section 0 to filter by season number and rating
2. ✅ AND that with watching criteria in Section 1
3. ✅ BUT ALSO add an alternate path in Section 2 for "completely watched" seasons
4. ❌ DUPLICATE the season/rating filters in Section 2 because you can't reference Section 0's logic!

### Why The Duplication?

You duplicated `seasonNumber != 1` and `rating < 7.5` in Section 2 because:

**The section model forces you to treat each section as independent!**

You can't say: "Use the same season/rating filter from Section 0 but with different watching criteria."

Instead, you have to:
- Copy the filter conditions into Section 2
- Manually ensure they stay in sync
- Hope you don't forget to update both places later

---

## The Cognitive Load

To understand this rule, someone has to:

1. **Trace through 3 sections**
2. **Remember which operators are stolen** for section combining
3. **Mentally expand the boolean algebra** to understand the full logic
4. **Notice the duplication** and realize it's intentional (not a copy-paste error!)
5. **Infer your intent** from the structure

**Estimated time to understand:** 10-15 minutes for an experienced developer

---

## What You WANTED To Write (My Best Guess)

Based on the logic, I think your actual intent is:

```
Find seasons where:

(Exclude season 1 OR has low rating)
AND
(
  Either:
    - Recently added but not watched, OR last watched >2 months ago
  OR
    - Everyone who started watching has finished all episodes and no unaired episodes
)
```

Is that close?

If so, here's how you'd write it in the **proposed AST format**:

```json
{
  "operator": "AND",
  "conditions": [
    {
      "operator": "OR",
      "conditions": [
        { "field": "sonarr.seasonNumber", "operator": "!=", "value": 1 },
        { "field": "sonarr.rating", "operator": "<", "value": 7.5 }
      ]
    },
    {
      "operator": "OR",
      "conditions": [
        {
          "operator": "OR",
          "conditions": [
            {
              "operator": "AND",
              "conditions": [
                { "field": "tautulli.addDate", "operator": "before", "value": "30_days" },
                { "field": "tautulli.sw_watchers", "operator": "count_equals", "value": 0 }
              ]
            },
            { "field": "tautulli.lastViewedAt", "operator": "before", "value": "59_days" }
          ]
        },
        {
          "operator": "AND",
          "conditions": [
            { "field": "tautulli.sw_allEpisodesSeenBy", "operator": "contains_all", "value": { "field": "tautulli.sw_watchers" } },
            { "field": "sonarr.unaired_episodes", "operator": "equals", "value": false }
          ]
        }
      ]
    }
  ]
}
```

**Benefits:**
- ✅ **No duplication** - season/rating filter appears once
- ✅ **Clear hierarchy** - you can SEE the nesting
- ✅ **Self-documenting** - operators are where they apply
- ✅ **Maintainable** - change season filter in ONE place

---

## Simplified Visual Tree

Your rule as a tree structure:

```
AND
├─ OR (Season/Rating Filter)
│  ├─ seasonNumber != 1
│  └─ rating < 7.5
└─ OR (Watching Criteria)
   ├─ OR (Not Watched Recently)
   │  ├─ AND
   │  │  ├─ addDate < 30 days
   │  │  └─ watchers = 0
   │  └─ lastViewed < 59 days
   └─ AND (Fully Watched)
      ├─ allEpisodesSeenBy ⊇ watchers
      └─ unaired = false
```

**Compare this to your YAML:**
- YAML: 3 sections, 10 rules, 2 duplicated conditions
- Tree: 1 clear hierarchy, 8 unique conditions, 0 duplication

---

## The Specific Problems In Your Rule

### Problem 1: Stolen Operators Are Invisible

```yaml
- "1":
    - operator: AND  # ← Looks like it ANDs with previous rule
```

**What it ACTUALLY does:** ANDs Section 1 with Section 0

**What it LOOKS LIKE:** ANDs this rule with... nothing (it's the first rule in the section!)

### Problem 2: Duplication Creates Maintenance Burden

If you want to change the season filter to `!= 0` (exclude specials), you have to update:
- Section 0, rule 1
- Section 2, rule 1

**If you forget one**, your rule produces inconsistent results!

### Problem 3: Can't Reuse Logic

You want both paths to use the same season/rating filter, but you can't express:

```yaml
seasonRatingFilter: &filter  # ← YAML anchor
  OR:
    - seasonNumber != 1
    - rating < 7.5

path1:
  - *filter  # ← Reference
  - AND: (watching criteria 1)

path2:
  - *filter  # ← Reference
  - AND: (watching criteria 2)
```

You're stuck duplicating.

### Problem 4: Section Numbers Are Meaningless

What do "0", "1", "2" mean?
- 0 = Initial filter
- 1 = Not watched recently path
- 2 = Fully watched path

But this is **only in your head**! The system just sees sequential numbers.

---

## How The New System Would Handle This

### Option 1: Define Reusable Groups (Future Feature)

```json
{
  "definitions": {
    "seasonRatingFilter": {
      "operator": "OR",
      "conditions": [
        { "field": "sonarr.seasonNumber", "operator": "!=", "value": 1 },
        { "field": "sonarr.rating", "operator": "<", "value": 7.5 }
      ]
    }
  },
  "operator": "AND",
  "conditions": [
    { "$ref": "#/definitions/seasonRatingFilter" },
    { /* watching criteria */ }
  ]
}
```

### Option 2: Inline (Current Proposal)

Just nest the conditions directly - no need for references since there's no forced duplication!

```json
{
  "operator": "AND",
  "conditions": [
    {
      "operator": "OR",
      "conditions": [/* season/rating filter */]
    },
    {
      "operator": "OR",
      "conditions": [/* watching paths */]
    }
  ]
}
```

### Option 3: Named Conditions (UI Helper)

UI could let you name logical groups:

```
Rule: "Find seasons to remove"

Conditions:
  ALL of:
    ✓ Season Filter
      ANY of:
        - Season number is not 1
        - Rating is less than 7.5
    ✓ Watching Status
      ANY of:
        - Not Watched Path
          ALL of:
            ANY of:
              - Added before 30 days and no watchers
              - Last viewed before 59 days
        - Fully Watched Path
          ALL of:
            - All watchers have seen all episodes
            - No unaired episodes
```

---

## The Bottom Line

Your rule is **cumbersome** because:

1. You couldn't nest conditions naturally
2. You had to use sections (which don't map to your mental model)
3. You had to duplicate logic across sections
4. The operators are overloaded and confusing
5. The structure doesn't reflect your intent

With the **proposed AST system**, your rule would:
- ✅ Be ~40% shorter
- ✅ Have zero duplication
- ✅ Be instantly understandable
- ✅ Be maintainable (change in one place)
- ✅ Match how you think about the logic

---

## Challenge Question

**If you wanted to add one more condition:**
> "Also exclude documentaries (genre)"

**Current system:** You'd have to add it to BOTH Section 0 AND Section 2 (3 places total!)

**Proposed system:** Add it once to the season/rating filter group.

This is why duplication is so painful - it compounds with every change.
