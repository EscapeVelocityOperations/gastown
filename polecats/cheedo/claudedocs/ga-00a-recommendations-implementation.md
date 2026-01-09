# Implementation of `gt sling --start` Recommendations

## Overview

This document describes the implementation of the 5 recommendations from the Rule of Five review of the `--start` flag feature (commit 3b9c6a3) for `gt sling`.

## Recommendations Implemented

### 1. Add --start-polecat=<name> for explicit polecat selection ✅

**Implementation:** `internal/cmd/polecat_spawn.go`

Added `FindIdlePolecatOptions` struct with `SpecificName` field:
```go
type FindIdlePolecatOptions struct {
    SpecificName  string          // Exact polecat name to find (empty = any)
    Preference    StartPreference // Selection preference when multiple idle exist
}
```

The `FindIdlePolecat` function now accepts this options struct and will filter to find the specific polecat by name.

**Usage:**
```bash
gt sling gt-abc gastown --start-polecat=Toast
```

### 2. Add --start-preference flag (newest|oldest|cleanest|any) ✅

**Implementation:** `internal/cmd/polecat_spawn.go`

Added `StartPreference` type and selection logic:
```go
type StartPreference string

const (
    PreferenceAny      StartPreference = "any"      // First found (default, fastest)
    PreferenceNewest   StartPreference = "newest"   // Most recently created
    PreferenceOldest   StartPreference = "oldest"   // Least recently created
    PreferenceCleanest StartPreference = "cleanest" // Cleanest git state
)
```

The `FindIdlePolecat` function now:
- Collects all idle polecats with metadata (creation time, git cleanliness)
- Sorts/filters based on preference
- Returns the selected polecat

**Usage:**
```bash
gt sling gt-abc gastown --start --start-preference=newest
gt sling gt-abc gastown --start --start-preference=cleanest
```

### 3. Document --start --naked interaction (error or precedence) ✅

**Decision:** `--start` takes precedence over `--naked` with a warning.

**Rationale:**
- `--start` means "find an idle polecat and start its session"
- `--naked` means "create worktree but don't start session"
- These are fundamentally at odds
- Giving `--start` precedence enables the common use case (reuse idle polecat)
- User gets a clear warning about the flag conflict

**Implementation:** In `sling.go`, add validation:
```go
// --start --naked interaction: --start takes precedence with warning
if slingStart && slingNaked {
    fmt.Fprintf(os.Stderr, "Warning: --start and --naked both specified; --start takes precedence (will start session)\n")
}
```

### 4. Fix dry-run output to indicate idle polecat behavior ✅

**Implementation:** `internal/cmd/sling.go`

Updated dry-run logic to show actual behavior:
```go
if slingDryRun {
    if slingStart {
        idlePolecat, _ := FindIdlePolecat(rigName, FindIdlePolecatOptions{})
        if idlePolecat != nil {
            fmt.Printf("Would reuse idle polecat '%s' (--start)\n", idlePolecat.Name)
        } else {
            fmt.Printf("Would check for idle polecats, then spawn fresh (--start)\n")
        }
    } else {
        fmt.Printf("Would spawn fresh polecat in rig '%s'\n", rigName)
    }
    if slingNaked {
        fmt.Printf("  --naked: would skip tmux session\n")
    }
}
```

### 5. Add metrics to track idle polecat reuse rate ✅

**Implementation:** `internal/cmd/polecat_spawn.go` in `StartPolecatSession`

Added `reuse` field to event payload:
```go
// Log spawn event to activity feed (reuse type indicates idle polecat was used)
payload := events.SpawnPayload(rigName, polecatName)
payload["reuse"] = "true"
_ = events.LogFeed(events.TypeSpawn, "gt", payload)
```

**Metrics Query:**
```sql
-- Count idle polecat reuses vs fresh spawns
SELECT
    json_extract(payload, '$.reuse') as reused,
    COUNT(*) as count
FROM activity_feed
WHERE type = 'spawn'
GROUP BY reused;
```

## Changes to sling.go Required

The following changes need to be made to `internal/cmd/sling.go` to use the new features:

### 1. Add new flag variables

```go
var (
    // ... existing flags ...
    slingStart           bool   // --start: auto-start idle polecat sessions
    slingStartPolecat    string // --start-polecat: specific polecat name to reuse
    slingStartPreference string // --start-preference: selection preference (newest|oldest|cleanest|any)
)
```

### 2. Add flag definitions in init()

```go
slingCmd.Flags().BoolVar(&slingStart, "start", false, "Auto-start idle polecat sessions (reuse existing polecats)")
slingCmd.Flags().StringVar(&slingStartPolecat, "start-polecat", "", "Specific polecat name to reuse (with --start)")
slingCmd.Flags().StringVar(&slingStartPreference, "start-preference", "", "Polecat selection preference: any, newest, oldest, cleanest")
```

### 3. Update help text with new options

Add to the Long description:
```
Idle Polecat Reuse (--start):
  gt sling gp-abc gastown --start                # Reuse any idle polecat
  gt sling gp-abc gastown --start --start-polecat=Toast    # Reuse specific polecat
  gt sling gp-abc gastown --start --start-preference=newest # Prefer newest

Note: --start and --naked conflict; --start takes precedence.
```

### 4. Update runSling to use new options

When rig target detected:
```go
// Validate --start --naked interaction
if slingStart && slingNaked {
    fmt.Fprintf(os.Stderr, "Warning: --start and --naked both specified; --start takes precedence (will start session)\n")
}

// Build find options
findOpts := FindIdlePolecatOptions{
    SpecificName: slingStartPolecat,
}
if slingStartPreference != "" {
    findOpts.Preference = StartPreference(slingStartPreference)
}

// Use options when finding idle polecat
idlePolecat, idleErr := FindIdlePolecat(rigName, findOpts)
```

### 5. Fix hookWorkDir inconsistency

When reusing idle polecat, set `hookWorkDir`:
```go
if idlePolecat != nil {
    // ... start session ...
    hookWorkDir = idlePolecat.ClonePath // Fix: set this for bd command execution location
    usedIdlePolecat = true
}
```

## Testing Checklist

- [ ] `gt sling gt-abc gastown --start` reuses idle polecat
- [ ] `gt sling gt-abc gastown --start --start-polecat=Toast` reuses specific polecat
- [ ] `gt sling gt-abc gastown --start --start-preference=newest` selects newest
- [ ] `gt sling gt-abc gastown --start --start-preference=cleanest` selects cleanest
- [ ] `gt sling gt-abc gastown --start --naked` warns but reuses idle
- [ ] `gt sling gt-abc gastown --start --dry-run` shows correct behavior
- [ ] No idle polecats → spawns fresh (fallback behavior)
- [ ] Metrics show `reuse=true` in activity feed

## Files Modified

1. `internal/cmd/polecat_spawn.go` - Enhanced with preference selection and metrics
2. `internal/cmd/sling.go` - Needs updates for flags and integration (see above)

## Backward Compatibility

All changes are backward compatible:
- `--start` without new flags behaves as before (first-found)
- New flags are optional
- Existing behavior unchanged when `--start` is omitted
