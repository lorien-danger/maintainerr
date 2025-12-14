# Null Handling Bugs: The Silent Failures

## Overview

The rules engine has **systemic null/undefined handling bugs** that cause rules to fail silently or produce unexpected results. These bugs stem from:

1. **Broken null checking logic**
2. **Getters returning null/undefined for missing data**
3. **Comparison functions that don't handle edge cases**
4. **No user feedback when rules fail**

---

## 🔴 Bug #1: The Broken Null Check (Lines 212-215)

### Location
`server/src/modules/rules/helpers/rule.comparator.service.ts:212-215`

### The Code
```typescript
if (
  (firstVal !== undefined || null) &&
  (secondVal !== undefined || null)
) {
  // do action
  const comparisonResult = this.doRuleAction(...);
}
```

### The Problem

**This condition is BROKEN!** Let me break down why:

```typescript
(firstVal !== undefined || null)
```

**What the developer THOUGHT this meant:**
> "Check if firstVal is not undefined AND not null"

**What it ACTUALLY means:**
```typescript
// Step 1: firstVal !== undefined → returns boolean (true or false)
// Step 2: boolean || null

// If firstVal is defined:
//   true || null → true ✓

// If firstVal is undefined:
//   false || null → null
//   if (null) → falsy, so condition fails ✓

// If firstVal is null:
//   true || null → true ✗  ← BUG! Null values pass through!
```

**The bug:** When `firstVal` is `null`, the check **PASSES** because:
- `null !== undefined` → `true`
- `true || null` → `true`
- Comparison runs with `null` values!

### The Correct Check Should Be
```typescript
if (
  firstVal !== undefined && firstVal !== null &&
  secondVal !== undefined && secondVal !== null
) {
  // do action
}
```

### Impact
- Rules run with `null` values
- Comparisons produce unexpected results
- `null < someDate` → `false` (not an error, just returns false!)
- **User's rule silently excludes media that was never watched**

---

## 🔴 Bug #2: The "Never Watched" Problem

### User's Scenario

**Rule:** "Find seasons not watched in the last 60 days"

```yaml
- firstValue: Tautulli.lastViewedAt
  action: BEFORE
  customValue: "60_days_ago"
```

**User's expectation:**
> "This should match media that hasn't been watched in 60+ days, INCLUDING media that's NEVER been watched"

**What actually happens:**

### Step 1: Getter Returns Null

`tautulli-getter.service.ts:156-158`
```typescript
case 'lastViewedAt': {
  const sortedHistory = history.filter(...).sort().reverse();

  return sortedHistory.length > 0
    ? new Date(sortedHistory[0] * 1000)
    : null;  // ← Never watched = null
}
```

### Step 2: Comparison Gets Null

```typescript
// firstVal = null (never watched)
// secondVal = Date 60 days ago

// The broken check:
if ((null !== undefined || null) && ...) {
  // null !== undefined → true
  // true || null → true
  // Condition PASSES!
}
```

### Step 3: Comparison Runs... and Fails

```typescript
doRuleAction(val1: null, val2: Date, action: BEFORE): boolean {
  // ...no special null handling...

  if (action === RulePossibility.BEFORE) {
    return val1 < val2;  // null < Date → false!
  }
}
```

**JavaScript behavior:**
```javascript
null < new Date()  // false
null > new Date()  // false
null === new Date() // false
```

**Result:** Media that's **never been watched is excluded** from the rule!

### The Fix Needed

```typescript
case 'lastViewedAt': {
  return sortedHistory.length > 0
    ? new Date(sortedHistory[0] * 1000)
    : new Date(0);  // ← Unix epoch = "infinitely long ago"
}
```

Or in the comparator:
```typescript
if (action === RulePossibility.BEFORE) {
  // Treat null as "never happened" = infinitely far in past
  if (val1 === null || val1 === undefined) {
    return true;  // null is before any date
  }
  return val1 < val2;
}
```

---

## 🔴 Bug #3: Empty Array Confusion

### User's Scenario

**Rule:** "Find seasons where all watchers have finished all episodes"

```yaml
- firstValue: Tautulli.sw_allEpisodesSeenBy
  action: CONTAINS_ALL
  lastValue: Tautulli.sw_watchers
```

**User's expectation:**
> "If watchers = [] (nobody watching), then vacuously true (everyone who started, finished)"

**What actually happens:**

### The Data

```typescript
// Season that nobody has watched:
sw_watchers = []
sw_allEpisodesSeenBy = []
```

### The Comparison

`rule.comparator.service.ts:522-547` (CONTAINS_ALL logic)
```typescript
if (action === RulePossibility.CONTAINS_ALL) {
  try {
    if (!Array.isArray(val1) || !Array.isArray(val2)) {
      return (val1 as unknown[])?.includes(val2);
    } else {
      if (val2.length > 0) {  // ← Check if val2 has elements
        return val2.every((el) => {
          return (val1 as unknown[])?.includes(el);
        });
      } else {
        return false;  // ← Empty array returns FALSE!
      }
    }
  } catch (_err) {
    return null;
  }
}
```

**Result:** `[] CONTAINS_ALL []` → `false`

### The Logic Problem

**Mathematically:** "All elements of empty set are in some other set" is **vacuously true** (there are no elements to check).

**In the code:** Empty arrays return `false`.

### The Ambiguity

What SHOULD happen for empty arrays? It depends on user intent:

1. **Vacuous truth interpretation:** `[] CONTAINS_ALL []` → `true`
   - "Everyone who started watching has finished" when nobody started → TRUE

2. **Strict interpretation:** `[] CONTAINS_ALL []` → `false`
   - "There must be at least one watcher who finished" → FALSE

**The current code chooses interpretation #2 without documenting it!**

### User Impact

The user probably wanted interpretation #1 (vacuous truth), so they had to work around it:

```yaml
# Instead of just checking "all watchers finished":
- firstValue: Tautulli.sw_allEpisodesSeenBy
  action: CONTAINS_ALL
  lastValue: Tautulli.sw_watchers

# They had to add BOTH:
- firstValue: Tautulli.sw_watchers
  action: CONTAINS
  lastValue: Tautulli.sw_allEpisodesSeenBy
- operator: AND
  firstValue: Tautulli.sw_allEpisodesSeenBy
  action: CONTAINS_ALL
  lastValue: Tautulli.sw_watchers
```

**This is checking both directions** to handle the empty array case!

---

## 🔴 Bug #4: Try-Catch Returns Null

### Location
Throughout `doRuleAction` method

### The Pattern
```typescript
if (action === RulePossibility.CONTAINS) {
  try {
    // ... comparison logic
  } catch (_err) {
    return null;  // ← Swallowed error!
  }
}
```

### The Problem

**When errors occur:**
1. Exception is caught
2. `null` is returned
3. The calling code doesn't check for `null` return!
4. `null` is treated as `false` (falsy)
5. **Rule silently fails with no error message**

### Example Failure Case

```typescript
// val1 is not an array, but we try to use array methods:
(val1 as unknown[])?.includes(val2)

// If val1 is a string or number, this might:
// - Work (strings have includes)
// - Fail silently (return null)
// - Return unexpected results
```

**User never knows their rule failed!**

---

## 🔴 Bug #5: Inconsistent Null/Undefined Returns

### The Problem

Different getters return different things for "no data":

```typescript
// Some return null:
return null;

// Some return undefined:
return undefined;

// Some return empty array:
return [];

// Some return 0:
return 0;
```

### Real Examples

**Tautulli getter:**
```typescript
case 'lastViewedAt':
  return sortedHistory.length > 0 ? new Date(...) : null;

case 'sw_lastWatched':
  return null;  // Always null for some reason?

// But at the end of the switch:
default:
  return undefined;
```

**Sonarr getter:**
```typescript
case 'sizeOnDisk':
  return null;  // Missing file

case 'quality':
  return undefined;  // Missing quality

case 'episodes':
  return undefined;  // Missing episodes
```

### Why This Is Bad

**The comparator doesn't handle null consistently:**

```typescript
// null < 100 → false (excludes media)
// undefined < 100 → false (excludes media)
// 0 < 100 → true (includes media)
```

**Same "missing data" concept, different behavior based on which getter returned what!**

---

## 🔴 Bug #6: The COUNT Operators Don't Handle Null

### User's Rule
```yaml
- firstValue: Tautulli.sw_watchers
  action: COUNT_EQUALS
  customValue: 0
```

**User expects:** "No watchers"

**What if sw_watchers returns null?** (No data from Tautulli)

```typescript
// COUNT_EQUALS is just redirected to EQUALS:
case RulePossibility.COUNT_EQUALS:
  return val1 === val2;

// If val1 = null:
null === 0  // false ← Excludes media even though there ARE no watchers!
```

**The fix:**
```typescript
case RulePossibility.COUNT_EQUALS:
  // Treat null as 0 count
  const count1 = val1 ?? 0;
  return count1 === val2;
```

---

## Real-World Impact on User's Rule

Let's trace through the user's actual rule with null handling bugs:

### Scenario: Season that's NEVER been watched

```typescript
// Data from APIs:
{
  seasonNumber: 2,
  rating: 6.0,
  addDate: 90 days ago,
  sw_watchers: [],  // Nobody watching
  lastViewedAt: null,  // ← NEVER WATCHED
  sw_allEpisodesSeenBy: []
}
```

### Section 1 Evaluation

```yaml
- firstValue: Tautulli.addDate
  action: BEFORE
  customValue: "30"  # 30 days
- operator: AND
  firstValue: Tautulli.sw_watchers
  action: COUNT_EQUALS
  customValue: 0
- operator: OR
  firstValue: Tautulli.lastViewedAt  # ← null!
  action: BEFORE
  customValue: "59"  # 59 days
```

**Rule 1:** `addDate (90 days ago) BEFORE 30 days ago` → ✓ TRUE
**Rule 2:** `sw_watchers.length (0) EQUALS 0` → ✓ TRUE
- Combined: `TRUE AND TRUE` → ✓ TRUE

**Rule 3:** `lastViewedAt (null) BEFORE 59 days ago` → ✗ FALSE
- `null < Date` → `false`
- Combined: `TRUE OR FALSE` → ✓ TRUE

**Section 1 result:** MATCHES ✓

### But here's the problem...

**If the user ONLY had Rule 3:**

```yaml
- firstValue: Tautulli.lastViewedAt
  action: BEFORE
  customValue: "59"
```

**Result:** `null < 59 days ago` → ✗ FALSE

**The season would be EXCLUDED even though it's NEVER been watched!**

This is counter-intuitive because:
- User thinks: "Not watched in 59 days includes never watched"
- System thinks: "null is not comparable, return false"

---

## The Systemic Problem

### Why These Bugs Exist

1. **No type safety** - Values are `any` or `RuleValueType` (union of everything)
2. **No null handling strategy** - Each getter does its own thing
3. **No documentation** - Users don't know what null means
4. **No error messages** - Silent failures everywhere
5. **Try-catch suppression** - Errors swallowed, return null
6. **Complex logic** - 615 lines make it hard to spot edge cases

### Why Users Don't Report Them

1. **Silent failures** - Rules just "don't match" expected media
2. **Workarounds** - Users add extra conditions to cover edge cases (like the user did!)
3. **Assume it's their fault** - "I must have written the rule wrong"
4. **Hard to debug** - No feedback on why rule failed

---

## The Better Approach

### Option 1: Explicit Null Handling

```typescript
interface RuleCondition {
  field: string;
  operator: string;
  value: any;
  nullBehavior?: 'exclude' | 'include' | 'treat_as_value';
}

// Example:
{
  field: "tautulli.lastViewedAt",
  operator: "before",
  value: "60_days",
  nullBehavior: "include"  // ← User explicitly says "include never-watched"
}
```

### Option 2: Default Null Semantics

```typescript
// For date comparisons:
// - null BEFORE any date → TRUE (treat as infinitely far in past)
// - null AFTER any date → FALSE

// For count comparisons:
// - null EQUALS 0 → TRUE (no data = zero count)
// - null > 0 → FALSE

// For contains:
// - null CONTAINS anything → FALSE
// - anything CONTAINS null → FALSE
```

**Document these clearly in the UI!**

### Option 3: Validation & Warnings

```typescript
// When rule is created:
if (condition.field === 'lastViewedAt' && condition.operator === 'before') {
  showWarning(
    'Note: Media that has never been watched will be excluded. ' +
    'If you want to include never-watched media, use a different approach.'
  );
}
```

### Option 4: Type-Safe Getters

```typescript
interface FieldDefinition {
  name: string;
  type: 'date' | 'number' | 'string' | 'array';
  nullable: boolean;
  nullMeaning?: string;  // "Never watched", "No data", etc.
  defaultValue?: any;
}

// Example:
{
  name: 'tautulli.lastViewedAt',
  type: 'date',
  nullable: true,
  nullMeaning: 'Never watched',
  // When null, comparison strategy is defined per operator
}
```

---

## How This Compounds With Other Issues

### Issue 1: Duplication + Null Bugs

User had to duplicate rules, and EACH duplicate might handle nulls differently!

```yaml
# Section 0: rating check
- firstValue: Sonarr.rating  # What if null? (no rating data)
  action: SMALLER
  customValue: 7.5

# Section 2: same rating check duplicated
- firstValue: Sonarr.rating  # Same null issue!
  action: SMALLER
  customValue: 7.5
```

If `rating` is `null`, both checks fail. User can't work around it in one place!

### Issue 2: Complexity Hides Bugs

With 615 lines and 3 mutable arrays, it's **impossible** to trace what happens with null values through the entire execution flow.

### Issue 3: No Testing

There are **zero tests** for null/undefined handling in the rule comparator!

```typescript
// These tests DON'T EXIST:
it('should include media with null lastViewedAt when using BEFORE', ...)
it('should handle empty array in CONTAINS_ALL', ...)
it('should treat null as 0 for COUNT_EQUALS', ...)
```

---

## Summary: The Null Handling Disaster

| Bug | Impact | User Sees |
|-----|--------|-----------|
| Broken null check (lines 212-215) | Nulls pass through to comparison | Unexpected exclusions |
| lastViewedAt returns null | Never-watched media excluded | "Why isn't this matching?" |
| Empty array in CONTAINS_ALL | Returns false (not vacuously true) | Had to duplicate rules |
| Try-catch returns null | Silent failures | Rule just doesn't work |
| Inconsistent null/undefined | Different getters behave differently | Unpredictable results |
| COUNT with null | null !== 0 | "No watchers" doesn't match |

**Bottom line:** The null handling is fundamentally broken, undocumented, and untested. Users work around it with complex duplicated rules, making the system even more confusing.

---

## How The New System Would Fix This

### 1. Type Safety
```typescript
type FieldValue =
  | { type: 'date', value: Date | null }
  | { type: 'number', value: number | null }
  | { type: 'array', value: any[] }  // Never null, always array (possibly empty)
```

### 2. Explicit Null Semantics
```typescript
const dateComparisonSemantics = {
  before: {
    nullBehavior: 'treat_as_epoch',  // null = infinitely far in past
    description: 'Never-watched media will be included'
  },
  after: {
    nullBehavior: 'exclude',
    description: 'Never-watched media will be excluded'
  }
};
```

### 3. Validation
```typescript
// Before running rule:
const warnings = validateRule(rule);
// "Warning: Field 'lastViewedAt' can be null. Null values will be treated as..."
```

### 4. Better Error Messages
```typescript
// Instead of silently returning null:
throw new RuleEvaluationError(
  `Cannot compare ${field} (null) with ${operator}. ` +
  `This field is null when media has never been watched.`
);
```

### 5. Comprehensive Tests
```typescript
describe('Null handling', () => {
  it('treats null date as epoch for BEFORE comparisons');
  it('excludes null for AFTER comparisons');
  it('treats empty array as vacuously true for CONTAINS_ALL');
  it('treats null count as 0');
});
```

**Result:** No silent failures, predictable behavior, documented edge cases, and users can see warnings BEFORE their rules fail!
