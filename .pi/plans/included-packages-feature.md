# Plan: Add `included_packages` to ManifestReference

## Problem

In a non-hierarchical dbt mesh (projects A, B, C all reference each other), every `ManifestReference` must explicitly `excluded_packages` for every transitive dependency to avoid duplicate/conflicting node injection. This exclusion list grows combinatorially — with N projects, each project's config needs O(N) exclusions per manifest reference.

**Current behavior:** Load ALL packages from the referenced manifest, then filter OUT the ones listed in `excluded_packages`.

**Desired behavior:** Optionally load ONLY the packages listed in `included_packages`, ignoring everything else in the manifest.

## Design

### 1. Config model change (`dbt_loom/config.py`)

Add an `included_packages` field to `ManifestReference`:

```python
class ManifestReference(BaseModel):
    name: str
    type: ManifestReferenceType
    config: Union[...]
    excluded_packages: List[str] = Field(default_factory=list)
    included_packages: List[str] = Field(default_factory=list)  # NEW
    optional: bool = False
```

Add a validator to ensure `included_packages` and `excluded_packages` are not both specified (they are logically opposing modes):

```python
@validator("included_packages", always=True)
def check_mutual_exclusion(cls, v, values):
    if v and values.get("excluded_packages"):
        raise ValueError(
            "Cannot specify both 'included_packages' and 'excluded_packages'. "
            "Use one or the other."
        )
    return v
```

### 2. Filtering logic change (`dbt_loom/__init__.py`)

In `dbtLoom.initialize()`, replace the current filtering block:

```python
# Current:
filtered_nodes = {
    key: value
    for key, value in selected_nodes.items()
    if value.package_name not in manifest_reference.excluded_packages
}
```

With logic that checks which mode is active:

```python
if manifest_reference.included_packages:
    # Include-only mode: only keep nodes from explicitly listed packages
    filtered_nodes = {
        key: value
        for key, value in selected_nodes.items()
        if value.package_name in manifest_reference.included_packages
    }
else:
    # Exclude mode (current behavior): keep everything except listed packages
    filtered_nodes = {
        key: value
        for key, value in selected_nodes.items()
        if value.package_name not in manifest_reference.excluded_packages
    }
```

### 3. Documentation update (`docs/advanced-configuration.md`)

Add a new section after "Exclude nested packages":

````markdown
## Include only specific packages

In a mesh with bi-directional references between many projects, maintaining
`excluded_packages` lists can become unwieldy. As an alternative, you can use
`included_packages` to explicitly whitelist which packages from a referenced
manifest should be injected:

```yaml
manifests:
  - name: project_b
    type: file
    config:
      path: ../project_b/target/manifest.json
    included_packages:
      - project_b # Only import nodes owned by this project
```
````

> **Note:** `included_packages` and `excluded_packages` are mutually exclusive.
> You must use one or the other, not both.

````

### 4. Tests

**Unit test** in a new or existing test file:
- Verify that `included_packages` filters correctly (only listed packages pass through)
- Verify that `included_packages` + `excluded_packages` raises a validation error
- Verify that neither specified defaults to include-all behavior

**Integration test** (optional, in `test_projects/`):
- Add a third test project or modify existing projects to demonstrate the mesh use case
- Verify that `included_packages: [project_name]` produces the correct node set

## Files to modify

| File | Change |
|------|--------|
| `dbt_loom/config.py` | Add `included_packages` field + mutual-exclusion validator to `ManifestReference` |
| `dbt_loom/__init__.py` | Update filtering logic in `initialize()` to support include-only mode |
| `docs/advanced-configuration.md` | Add "Include only specific packages" section |
| `tests/test_*.py` | Add unit tests for validation and filtering |

## Risk assessment

- **Low risk** — the change is additive and backward-compatible. Existing configs using `excluded_packages` continue to work unchanged.
- The mutual-exclusion validator prevents ambiguous configs.
- The filtering logic is a simple conditional branch on existing code.

## Example before/after

**Before (5 projects, all referencing each other):**
```yaml
# Project A's config — needs to exclude B, C, D, E from every manifest
manifests:
  - name: project_b
    type: file
    config:
      path: ../project_b/target/manifest.json
    excluded_packages:
      - project_a      # self-exclusion (A includes B, B includes A)
      - project_c      # C is transitively in B's manifest
      - project_d      # D is transitively in B's manifest
      - project_e      # E is transitively in B's manifest
  - name: project_c
    type: file
    config:
      path: ../project_c/target/manifest.json
    excluded_packages:
      - project_a
      - project_b
      - project_d
      - project_e
  # ... repeats for every manifest
````

**After:**

```yaml
# Project A's config — just say which project owns each manifest
manifests:
  - name: project_b
    type: file
    config:
      path: ../project_b/target/manifest.json
    included_packages:
      - project_b
  - name: project_c
    type: file
    config:
      path: ../project_c/target/manifest.json
    included_packages:
      - project_c
  # ... clean and maintainable
```
