# Bryan's Blocking Issue Taxonomy

## Decision Tree: Would Bryan Block This?

```
1. Does it break existing consumers (profiles, CLI args, custom integrations)?
   YES -> BLOCK. Always. No exceptions. Add backward-compat unit tests.

2. Does it introduce a new type/enum/abstraction?
   -> Does an equivalent already exist in .NET or VC framework?
      YES -> BLOCK. Use the existing one.

3. Does it change profile structure?
   -> Does it destroy --scenarios filtering or collapse distinct scenarios?
      YES -> BLOCK. Scenarios are functional selectors.
   -> Does it expose a parameter that contradicts the profile's purpose?
      YES -> BLOCK. Remove it from globals.
   -> Does it over-parameterize (expose implementation knobs)?
      YES -> BLOCK. Move constants to job files or code.

4. Is the naming inconsistent?
   -> Does a parallel concept already have a naming pattern?
      YES -> BLOCK. Follow the pattern.
   -> Does the name leak implementation details?
      YES -> BLOCK. Use domain concepts.

5. Is the fix in the wrong layer?
   -> Are you restructuring profiles to fix a code bug?
      YES -> BLOCK. Fix the code.
   -> Is one-time setup in Actions instead of Dependencies?
      YES -> BLOCK. Move it.

6. Is there a missing validation/guard?
   -> Can a user pass an unsupported value with no error?
      YES -> BLOCK (medium). Add throw with ErrorReason.

7. Are code conventions violated?
   -> Logic in property getters? Duplicated derivation logic?
      YES -> BLOCK (medium). Move to InitializeAsync, centralize as statics.
```

## Meta-principle
Bryan optimizes for long-term maintainability of the profile contract. Profiles are the primary user-facing API. Every decision filters through: "Will this make profiles harder to understand, harder to compose, or break existing consumers?"

## Category Details

### 1. BACKWARD COMPATIBILITY (Always blocking)
- Any change breaking existing --scenarios filtering, time parameters, or profile parameter names
- Test: "Does any existing consumer's command line or custom profile stop working?"
- Evidence: PR 601 (integer→timespan backward compat tests), PR 562 (scenario name preservation)

### 2. ABSTRACTION LEVEL — Over-engineering (High)
- Don't introduce new types when .NET/VC framework types already model the concept
- Test: "Does .NET or VC already have a type for this?"
- Evidence: PR 412 (rejected MetricVerbosity enum in favor of LogLevel)

### 3. PROFILE COMPOSITION — Wrong parameterization (High)
- Exposed parameters must be things users SHOULD vary for that recipe
- Test: "If someone overrides this parameter, does the profile still make sense?"
- Evidence: PR 303 (remove Benchmark from TPCC profile globals), PR 300 (OLTP weights → job file)

### 4. NAMING/IDENTITY (Medium-High)
- Consistent with existing patterns, domain terminology, no implementation leaks
- Test: "Would a profile reader understand this without seeing the code?"
- Evidence: PR 544 (LoopExecution→SequentialExecution), PR 641 (file/class name mismatch)

### 5. DESIGN DIRECTION — Wrong layer (High)
- Fix bugs where they originate, don't restructure higher layers to compensate
- Test: "Am I changing profile/architecture to work around a code bug?"
- Evidence: PR 562 ("This looks like a code bug, not a profile issue"), PR 271 (DB creation→Dependencies)

### 6. MISSING VALIDATION (Medium)
- Every user-input code path needs terminal guard for unsupported values
- Test: "What happens if user passes invalid value?"
- Evidence: PR 303 (throw NotSupported for unknown benchmark), PR 491 (parameter cohesion)

### 7. CODE CONVENTIONS (Medium)
- No logic in property getters. Shared logic centralized. Lifecycle methods own init.
- Test: "Is this logic in the right method? Is it duplicated?"
- Evidence: PR 271 (3x "avoid logic in getters"), PR 271 (static methods on config class)

## Verified Against 14 CHANGES_REQUESTED PRs
PRs analyzed: 647, 641, 613, 601, 584, 562, 544, 491, 471, 414, 412, 303, 300, 271
