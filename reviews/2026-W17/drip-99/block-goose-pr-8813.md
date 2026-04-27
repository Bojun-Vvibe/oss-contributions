# block/goose#8813 — fix: gracefully handle corrupted permission config instead of panicking

- **Head**: `1565ccfe7f1ccacb57737d93e946b1ce2b962a66`
- **Size**: +26/-16 across 1 file
- **Verdict**: `merge-as-is`

## Context

`PermissionManager::new()` is called during process startup. The previous
implementation `panic!`ed on two distinct failure modes — (a) `fs::read_to_string`
failing (e.g. permission denied, transient I/O error) and (b)
`serde_yaml::from_str` failing (corrupted YAML) — which crashed the
entire process before any subsystem could come up. A user with a hand-edited
or partially-written `permission.yaml` was effectively locked out of the
tool until they could find and delete the file from a separate shell.
This PR replaces both panic paths with `tracing::error!` + fallback to
`HashMap::new()`, letting the process boot with empty permissions and an
actionable error message in the logs.

## Design analysis

### Before: two `panic!` calls in `permission.rs:48-58`

```rust
let file_contents =
    fs::read_to_string(&permission_path).expect("Failed to read permission.yaml");
serde_yaml::from_str(&file_contents).unwrap_or_else(|e| {
    tracing::error!(...);
    panic!("Corrupted permission config at {}. Fix or remove the file to continue.", ...);
})
```

The first failure mode (`expect("Failed to read permission.yaml")`) crashes
with no actionable message. The second (`panic!` after `tracing::error!`)
at least logs first, but still aborts the process.

### After: nested match producing `HashMap::new()` on either failure path

```rust
match fs::read_to_string(&permission_path) {
    Ok(contents) => match serde_yaml::from_str(&contents) {
        Ok(map) => map,
        Err(e) => {
            tracing::error!(
                path = %permission_path.display(),
                error = %e,
                "Failed to parse permission config — starting with empty permissions. \
                 Fix or remove the file to suppress this warning.",
            );
            HashMap::new()
        }
    },
    Err(e) => {
        tracing::error!(
            path = %permission_path.display(),
            error = %e,
            "Failed to read permission config — starting with empty permissions.",
        );
        HashMap::new()
    }
}
```

Three things done right here:

1. **Both error paths now have structured logging** with `path` and `error`
   as separate `tracing` fields, not just interpolated into a message string.
   That's the right shape for log aggregation (`grep path=...` or filter
   by error value) — the previous code put `permission_path.display()`
   inline as text, which is less queryable.

2. **The fallback (`HashMap::new()`) is the safe default** for a
   permissions map. Empty permissions = "no special grants", which means
   the user lands in the most-restrictive state where every action will
   prompt for approval. This is the right fail-open-but-still-safe
   posture: the tool keeps working, but the user is never silently
   granted permissions they didn't intend.

3. **The error message is actionable**: "Fix or remove the file to suppress
   this warning." Tells the user exactly what to do without requiring
   them to spelunk through source.

### Test refactor: `permission.rs:367-372`

```rust
-#[should_panic(expected = "Corrupted permission config")]
-fn test_corrupted_permission_file_panics() {
+fn test_corrupted_permission_file_falls_back_to_empty() {
    let temp_dir = TempDir::new().unwrap();
    let permission_path = temp_dir.path().join(PERMISSION_FILE);
    fs::write(&permission_path, "{{invalid yaml: [broken").unwrap();
-   PermissionManager::new(temp_dir.path().to_path_buf());
+   // A corrupted file should not panic — it should start with an empty permission map.
+   let manager = PermissionManager::new(temp_dir.path().to_path_buf());
+   assert!(manager.get_permission_names().is_empty());
}
```

This is exactly the right test transformation. The old test asserted
behavior that the PR is intentionally changing (panic), so it has to
flip. The new assertion (`get_permission_names().is_empty()`) directly
pins the safe-default behavior. Renaming the test function to match
the new behavior is a nice bonus that keeps `cargo test`'s output
self-documenting.

## Concerns / nits

1. **The unrelated `expect("Failed to create config directory")`** at
   the path where `permission_path` does not exist (the `else` branch
   of `if permission_path.exists()`) still uses `expect`. If this PR's
   philosophy is "don't crash startup on filesystem hiccups," that
   second `expect` is inconsistent. The fix probably wants to be in
   scope here too — log + fallback to in-memory empty map. Not a
   blocker (this path only fires when the directory doesn't exist *and*
   `mkdir -p` fails, which is much rarer than a corrupted YAML file),
   but worth a follow-up.

2. **No test for the I/O-error path** (the new `Err(e) => { ... }` arm
   on `fs::read_to_string`). The new test pins the parse-error path;
   the read-error path is still untested. A test that creates the file
   with `0o000` perms (or makes the parent dir un-listable) on a
   non-Windows runner would close the gap. The path is small, but
   it's the one most likely to hit a real user (locked-down corp
   filesystems, NFS hiccups), and "we tested only one of the two new
   error arms" is exactly the kind of asymmetry that comes back to
   bite later.

3. **`tracing::error!` for an expected-failure-mode** is debatable.
   "Permission file got corrupted, falling back to empty" is loud
   enough to be `error!`, but if the policy elsewhere in the project
   is `warn!` for "we recovered, here's what we did", it might be more
   consistent. Cosmetic.

4. **Both error messages start with "Failed to ..." and end with
   "starting with empty permissions"** but use slightly different
   phrasing for the recovery hint (`"Fix or remove the file to
   suppress this warning."` only on the parse-error arm). A user who
   hits the read-error arm will get a less-actionable message.
   Trivially fixable by adding the same hint to both.

## Verdict reasoning

Surgical fix that eliminates an uncrashed-startup panic on a real
user-reachable failure mode (corrupted or unreadable `permission.yaml`).
The recovery posture (empty permissions = most-restrictive prompts)
is safe. Test correctly flips from `should_panic` to a behavioral
assertion. Nits are all "while you're here" follow-ups, not
blockers. This is the kind of fix that should land immediately —
it's strictly safer than HEAD.

## What I learned

`expect("Failed to read X")` and `unwrap_or_else(|e| panic!(...))` are
two of the most common "we'll handle it later" placeholders in Rust
codebases, and they tend to live in initialization code where they're
the most damaging — by the time you discover one panicking on a user's
machine, the whole process is dead and you have no chance to log a
diagnostic from a higher layer. The fix here (nested `match` with
structured `tracing::error!` on both arms, fallback to a safe default)
is the canonical replacement pattern. The fact that the *test* had to
be rewritten (not just augmented) is a tell that the original code
was specifying the wrong contract: "panic on corruption" was load-bearing
behavior pinned by a test, when it should never have been the contract
in the first place.
