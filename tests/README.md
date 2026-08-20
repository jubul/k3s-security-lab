# Policy test cases

Negative test cases for the Kyverno ClusterPolicies in [`../policies/`](../policies/). Each file is a minimal Pod exercising **one** bypass route, with its expected result declared in the header comment.

## Running them

```bash
kubectl apply -f tests/ -n demo --dry-run=server
```

`--dry-run=server` sends the object to the API server, which runs it through the admission webhooks and returns the verdict **without persisting anything**. This is the right way to test policies: it exercises the real admission path, not a simulation of it.

## Expected results

| File | Route tested | Rule that must catch it | Expected |
|---|---|---|---|
| `01-latest-container.yaml` | `:latest` in `containers` | `validate-image-tag` | 🚫 blocked |
| `02-notag-container.yaml` | no tag in `containers` | `require-image-tag` | 🚫 blocked |
| `03-latest-initcontainer.yaml` | `:latest` in `initContainers` | `validate-image-tag` | 🚫 blocked |
| `04-notag-initcontainer.yaml` | no tag in `initContainers` | `require-image-tag` | 🚫 blocked |
| `05-valido.yaml` | everything pinned | none | ✅ allowed |

The first four must return an admission error; the fifth must report `created (server dry run)`.

## Design criteria

**In cases 03 and 04 the main container is deliberately valid.** If the case fails, the only possible cause is the initContainer. A test case with two things wrong at once can't tell you which rule caught it, and stops being useful as a diagnostic.

**The positive case isn't optional.** A policy that blocks everything is as useless as one that blocks nothing. Case `05` verifies that the *conditional anchors* (`=(field)`) work: without them, a Pod that doesn't declare `initContainers` would fail validation for lacking a field it has no business having.

**One file, one route.** Case `04` is the one that caught the residual bug from [`journal.md` #8](../docs/journal.md): a capitalization error (`initcontainers` instead of `initContainers`) present in only one of the two rules. It's the only case that needs that rule **and** that container list simultaneously — with coarser cases, the hole went unnoticed.

## Next step

Migrate to **`kyverno test`**, the Kyverno CLI, which declares each case's expected result in a file and runs **without a cluster**. That allows running the suite in CI and making a pull request that opens a policy hole fail the pipeline.
