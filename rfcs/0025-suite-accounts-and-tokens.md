# RFC-0025: Suite accounts and tokens

Renumbered from 0019. #34 claims that number. 0025 is free.

RFC-0024 (storytold/craft-contrib#27) deliberately left accounts and tokens out, as the most security-sensitive part of the cloud-documents track. This is the proposal it deferred. It also answers the shared-keychain question raised in review there: the same secret layer serves AI connections and storage connections.

## Summary

Secrets live in the operating system's keychain. The account list holds no secret. It lives in one suite-level folder. A person signs in once, in the launcher or in any app, through the same shared components. Every supporting app then uses that session. The launcher stays optional.

One provider-neutral crate holds the secret store. Three tracks consume it: AI connections (the craft-ai-integrations platform contract already states "secret values belong in the secret store, never in the shared JSON"), storage connections (RFC-0024), the launcher's keychain work (storytold/craft-launcher#366). That gives the crate three named consumers from day one.

## What exists today, honestly assessed

`crates/suite-account` was built in the craft-libs#160 branch and held back from that PR's cut-down, correctly, pending this proposal. Its storage design is already right.

1. Tokens go to the platform keychain through the `keyring` crate. Windows Credential Manager, the macOS login keychain, the Secret Service on Linux and the BSDs. No OpenSSL, no C toolchain, `unsafe_code = "forbid"` holds.
2. One suite-wide service name covers every entry. Everything the suite ever writes is therefore findable and removable in one place. Windows entries use machine-local persistence. A token minted for one machine stays on it.
3. The keychain user is a digest of the deployment and the subject. A second sign-in therefore replaces the entry. No stale entry is left behind.
4. Token bytes are zeroized on drop.
5. The account list (display name, service, handle, no secret) is a small JSON file in a suite-level folder (`%APPDATA%\ArtCraft\accounts.json` and platform equivalents), per the analysis in storytold/craft-launcher#366: an app-owned folder would make other apps read a folder they do not own. The launcher's folder would make apps depend on the launcher existing.
6. `TokenStore` is a trait with an in-memory stand-in. Everything above it tests without a keychain.

Its defect, named in review on craft-libs#160: the service half is CollabCraft-specific. `service.rs` talks to a CollabCraft deployment. `store.rs` documents the secret as "the CollabCraft session token". A shared crate must not name a provider.

## What this RFC decides

**1. Where tokens live.** The platform keychain, exactly as the existing code does it. One suite service name. One entry per account handle. Machine-local persistence where the platform offers it. No token ever lands in a JSON file, a shared store, a tool declaration, or an environment variable written by the suite.

**2. Where the account list lives.** The suite-level folder, as item 5 above. The list carries identity and routing only. Grep it for secrets and find none.

On the folder's name. The concrete values today are `ArtCraft` on Windows and macOS, `artcraft` on Linux and the BSDs, with `accounts.json` inside and an `ARTCRAFT_SUITE_DIR` override. The keychain service name is `ai.storyteller.artcraft.suite-account`, one constant covering every entry the suite writes. ArtCraft is the launcher's established product name. The folder deliberately matches the launcher's existing per-platform spelling. That is how the launcher and the apps share one folder while no app has to know the launcher exists. Renaming to a suite-neutral spelling is feasible if the maintainers prefer it. The crate already ships a migration module for exactly this kind of move (read the old location, write the new). The keychain service constant would rename in the same PR. The maintainers' preference decides it on the extraction PR, with the launcher-established name as the default when no preference is stated.

**3. The crate split.** The provider-neutral parts of `crates/suite-account` (the `TokenStore` trait, the keychain implementation, the memory stand-in, the handle digest, the paths, the account list) become the shared secret layer. The CollabCraft device-flow client in `service.rs` moves out, to live beside the provider adapter it belongs to. After the split, nothing in the shared crate names any provider. Whether the result ships as one crate or two (`craft-suite-account` for the list, a `craft-secrets` for the store) is a review-level detail for the extraction PR, with a single-crate default.

**4. One secret layer for AI and storage.** An AI connection's API key and a storage connection's bearer token are the same kind of object. Both go through the shared crate. The craft-ai-integrations contract already points this direction. Nothing AI-specific or storage-specific enters the shared crate; a consumer supplies its own service naming under the suite's namespace.

**5. Sign-in surface, launcher optional.** Configuration follows the craft-ai-integrations platform contract, as RFC-0024 boundary 1 established for connections generally. A person configures an account in the launcher or in any app, through the same shared components. Both write the same account list and the same keychain entries. An app never needs the launcher installed. storytold/craft-launcher#366 becomes the launcher's adoption of this layer. The layer itself has no owner app.

**6. Web builds are out of scope.** A browser has no OS keychain. Web sessions belong to the hosting gateway (RFC-0024 boundary 4) and never pass through this crate. The crate stays non-wasm, as the existing code already declares.

## Cross-app lifecycle guarantees

Several apps run at once against one account list and one keychain. The contract therefore states the concurrent behavior explicitly, with multi-process or simulated-interleaving tests in the extraction PR.

1. **No lost updates.** Account-list and credential updates are atomic at the store level (locked update over the list file, one keychain entry per handle). Two apps updating concurrently never lose each other's writes. A partial failure (list updated, keychain write failed, or the reverse) leaves a defined state the next reader detects and repairs, never a silent mismatch.
2. **Removal wins over refresh.** The account list carries a generation per account. Removing the account bumps the generation and deletes the entry. A refresh that started earlier completes against the old generation and is discarded. A completed refresh can never recreate a removed credential. Only a new sign-in can.
3. **Propagation is stated, never assumed.** No inter-process channel is assumed. An app validates the account's generation against the shared list before each credential use and each refresh. A removed account therefore stops being used and stops being refreshed on the app's next touch. The staleness window is bounded by that rule and stated openly.
4. **Forgetting is local. Revoking is at the provider.** Deleting the keychain entry forgets the credential on this machine. It does not invalidate a token already held in another process's memory. It does not end a server-side session. Provider-side revocation is a distinct operation on the provider adapter. The shared UI names the two actions distinctly so a person knows which one they performed.

Refresh itself is out of this RFC entirely. When a token refreshes, how rotation works, what expiry means: these are properties of the provider's auth flow. They vary even within one protocol. OIDC's device flow, authorization code flow and client-credentials flow each handle refresh differently. The provider adapter owns all of that. This RFC constrains only the boundary where a refresh touches the shared store: a completed refresh writes through the generation check above. Nothing more is dictated. The guarantees above are the shared contract every consumer may rely on.

## Security posture

At rest, the platform encrypts keychain entries and scopes them to the signed-in user. In memory, token bytes are zeroized on drop. For revocation, one suite service name makes every entry visible and deletable in the platform's own credential UI. Removing an account from the list removes its entry. Across apps, no token is in any file. No app ever reads another app's files for one. The threat this design does not address, by the nature of OS keychains: code running as the same user can request the same entries. That is the platform's trust boundary, stated here so nobody mistakes the guarantee.

## Reference implementations, non-normative

Working code backs every claim above, offered as reading aid, never as a dependency. The keychain store, the account list, the handle digest and the memory stand-in are the held-back `crates/suite-account` in the craft-libs#160 branch history. The provider client that must move out of it is its `service.rs`, the device-flow client for one hosted deployment. The problem the design replaces is visible in any current app build: a bearer token in a per-app JSON file beside its settings. None of this is normative. The extraction PR's traits are the only interface a consumer may depend on.

## Landing order

1. This RFC is accepted.
2. One extraction PR to craft-libs, open as a draft: storytold/craft-libs#665. It carries the provider-neutral secret layer out of the held-back `suite-account` code, with the CollabCraft service client removed and the generation field implemented. Named consumers: the launcher (storytold/craft-launcher#366), the AI connection store (craft-ai-integrations), the storage adapters (RFC-0024 track). Named maintainer: wdunn001.
3. storytold/craft-launcher#366 rebases onto the shared crate.
4. The RFC-0024 storage adapters and the AI integration adopt it as they land.

## Open questions left for the PRs

Handle composition for multi-deployment accounts (the digest today covers deployment plus subject; the extraction PR documents it). Enterprise policy hooks (out of scope until someone needs them).

Related to storytold/craft-libs#123. Context: storytold/craft-contrib#27 (RFC-0024), storytold/craft-launcher#366, storytold/craft-libs#160.
