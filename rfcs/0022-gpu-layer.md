# RFC-0022: A shared GPU layer

The gpu slice was split out of storytold/craft-libs#160 by review, into its own draft PR: storytold/craft-libs#663. This RFC gives that slice the suite-level decision it needs before merging.

## Summary

The suite gains one GPU abstraction: `craft-gpu` holds the traits, the job model (`GpuJob`) and a provider registry; `craft-gpu-wgpu` is the baseline implementation over wgpu (WebGPU on wasm, Vulkan, DX12 or Metal natively); `craft-gpu-null` is an in-memory context so CI and headless tests run with no device. Apps and the shared runtime code against `craft-gpu`'s traits. The backend is swappable per the same seam discipline the rest of this track uses.

## What exists

The three crates plus `craft-gpu-providers` are written, held back from craft-libs#160's cut-down pending this decision. The design, from the slice's own docs:

1. `craft-gpu` carries the traits plus `GpuJob` and `ProviderRegistry`, with device capabilities modeled explicitly (`DeviceCaps`).
2. `craft-gpu-wgpu` feature-gates each native backend (`native-vulkan`, `native-dx12`, `native-metal`, `native-gles`) and an `adapter-probe` that fills real capabilities natively, falling back cleanly on machines with no GPU, such as CI runners.
3. On wasm32 it builds with wgpu's `webgpu` feature. WebGPU is the browser baseline.
4. `craft-gpu-null` keeps the whole layer testable with no hardware.

## Why a shared layer

Render-heavy and compute-heavy work recurs across the suite and the runtime track (the worker-host and tier-store slice). Each app growing its own wgpu plumbing reproduces the same device setup, capability probing and fallback logic with drift. One layer makes capability differences explicit data (`DeviceCaps`) and makes "no GPU present" a tested path everywhere. The baseline-plus-seam shape also matches the position already taken publicly in the upstream SolidCraft discussion of hardware ray queries: WebGPU stays the baseline. Vendor-specific acceleration lives behind a seam in a leaf crate.

## Room for optimization, on every backend

Optimization is capability-driven everywhere, through one mechanism. `DeviceCaps` expresses what the adapter actually offers. An app or the runtime therefore takes a faster path when a capability is present (half-precision floats, subgroup operations, timestamp queries, compressed texture formats) and the portable path when it is absent. A new capability, on any backend, becomes a new `DeviceCaps` field plus a gated fast path, with no fork in app code and no new trait. The baseline stays portable. Optimization is always additive on top of it.

The browser has its own particulars within that mechanism. Capability probing there moves from static defaults to real adapter queries once async initialization lands in the worker, per the slice's own docs. Future WebGPU extensions ride the same `DeviceCaps` route as everything else. The browser floor therefore rises without app changes as the standard grows.

## Acceptance requirements for the capability model

Two requirements make the portability and fallback claims verifiable, per review.

1. **Caps describe the created device, never just the adapter.** `DeviceCaps` used to select a fast path must reflect the features and limits actually enabled on the device that was created, since an adapter advertises more than a device may grant. Browser defaults stay conservative until initialization succeeds and real device caps replace them. The no-adapter case is defined behavior, the same degrade path the null context exercises. Device loss surfaces as a fault through the job outcome, with the consumer's recovery stated: fall back to the portable path or re-acquire a device and re-probe, never continue against stale caps.
2. **Real backends in CI, beside the null one.** The null context stays mandatory for every consumer. It cannot validate shader compilation, resource rules or numerical results. Pilot consumers therefore also run representative jobs on a real wgpu backend and in a browser, comparing outputs against expected results with stated tolerances. The suite includes runs with optional features masked off. The portable path is therefore continuously proven.

## What this RFC decides

The layer lives in the shared repository as the four crates above. wgpu is the baseline implementation and WebGPU the browser floor. Capability differences are explicit data, never implicit behavior. Every consumer must run against `craft-gpu-null` in CI. Named consumers land on the draft PR per review precedent: the runtime slice (worker-host executing GPU jobs) is the first. The render-path apps named there are the second set. Vendor-specific backends beyond wgpu, if ever wanted, arrive as adapters behind the same traits in their own PRs.

Related to storytold/craft-libs#123. Context: storytold/craft-contrib#27 (RFC-0018), storytold/craft-libs#160 (the review that split this out).
