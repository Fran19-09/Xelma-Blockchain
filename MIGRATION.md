# Migration Notes

## Package Rename: `@tevalabs/xelma-bindings` → `@xelma/bindings`

**Introduced in:** `fix/bindings-package-metadata`

### What changed

The npm package name was updated from the placeholder org-scoped name
`@tevalabs/xelma-bindings` to the canonical Xelma namespace `@xelma/bindings`.

The following metadata fields were also added or corrected:

| Field | Before | After |
|-------|--------|-------|
| `name` | `@tevalabs/xelma-bindings` | `@xelma/bindings` |
| `repository` | _(absent)_ | `https://github.com/TevaLabs/Xelma-Blockchain` |
| `author` | _(absent)_ | `TevaLabs` |
| `license` | _(absent)_ | `MIT` |

### Migration steps for consumers

1. **Uninstall the old package** (if previously published under the old name):

   ```sh
   npm uninstall @tevalabs/xelma-bindings
   ```

2. **Install the new package:**

   ```sh
   npm install @xelma/bindings
   ```

3. **Update all import statements:**

   ```diff
   - import { Client } from '@tevalabs/xelma-bindings';
   + import { Client } from '@xelma/bindings';
   ```

### Import path impact

Only the package name changed. All exported symbols (`Client`, `ContractError`,
`BetSide`, `RoundMode`, `UserPosition`, etc.) remain identical — no code changes
are required beyond updating the import path.
