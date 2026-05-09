# Plan: Port Moonray to Gaffer (via Hydra delegate)

## Context

OpenMoonRay (https://github.com/OpenMoonRay/openmoonray) is DreamWorks Animation's
production path tracer (Apache 2.0, C++17 + ISPC). Gaffer's existing renderer
backends (Arnold, Cycles, 3Delight, RenderMan) all subclass
`IECoreScenePreview::Renderer` and call native renderer SDKs directly.
GafferHQ/gaffer issue #6573 asks for Moonray support; it currently has zero
responses, so this is fresh territory.

**Goal:** ship `GafferMoonray` as a first-class renderer backend, reaching
feature parity with `GafferCycles` for v1 (interactive + batch render, meshes /
curves / volumes, motion blur, light linking, AOVs, render adaptors, shader
network wrapping moonshine), and adding Arras distributed rendering + OptiX 7.6
XPU support in v2.

**Decisions already made (with the user):**
- **Integration path:** Hydra delegate (`hdMoonray`). This makes Moonray the
  first Hydra-routed renderer in Gaffer; every other backend talks to its
  renderer SDK directly. The user chose this knowing the trade-off.
- **Platform:** Linux only. Gaffer supports Linux + Windows; Moonray supports
  Linux + macOS; intersection is Linux.
- **Shader strategy:** v1 wraps moonshine SceneClasses to generate Gaffer plugs
  (analog to `ArnoldShader`'s metadata-driven plug generation). OSL bridge
  deferred to a later phase — Moonray has no OSL runtime, so the easy
  `OSLShader::registerCompatibleShader("ai:surface")` trick that Arnold uses
  does not transfer.
- **Scope:** v1 = GafferCycles parity. v2 = Arras + XPU.

## Why this is hard (the hurdles)

Listed worst-first. Each must have an answer before / during the phase that
hits it.

| # | Hurdle | Severity | Notes & mitigation |
|---|---|---|---|
| H1 | **USD version pinning collision.** Gaffer 1.6 ships **USD 26.05** (verified in `GafferHQ/dependencies`); Moonray uses Rez + dynamic `find_package` with no public pin file, but `CMakeLinuxPresets.json` references Python 3.9 — strongly suggesting Moonray's expected USD is 24.x or 25.x, not 26.05. USD has well-known ABI breaks across minor versions (`HdSceneIndex` API changed substantially between 24.x and 25.x). Both halves must link the *same* USD shared libs at runtime, or `HdRenderIndex` constructed by Gaffer can't be consumed by hdMoonray. | **HIGH — confirmed blocker** | **Resolve before Phase 1.** Concrete actions: (1) build Moonray against USD 26.05 — likely needs source patches to track 26.x API drift; (2) backport `hdMoonray` to USD 26.05 if upstream Moonray isn't there yet (this is the real ongoing maintenance cost); (3) alternative: pin Gaffer's USD down to whatever Moonray uses — but breaks the rest of Gaffer's USD-aware code. Path 1 is the only sustainable option. |
| H1b | **Python ABI mismatch (NEW finding).** Gaffer 1.6 ships **Python 3.11.14**; Moonray's `CMakeLinuxPresets.json` references `python39` (Boost.Python component name `python39`, OIIO Python at `lib64/python3.9/site-packages`). Boost.Python compiles against a specific Python ABI. If Moonray's `hdMoonray.so` and its dependencies are built against Python 3.9 and Gaffer is 3.11, the `IECoreMoonray.so` cannot bridge them in the same process. | **HIGH — confirmed blocker** | Build Moonray (and its full dep stack — Boost, OIIO, USD with Python bindings) against Python 3.11. Requires patching Moonray's CMake presets. No way around this. Adds ~1–2 weeks to Phase 0. |
| H2 | **ABI clashes on Boost / TBB / OIIO / OpenEXR.** Gaffer 1.6 ships: **Boost 1.85.0, oneTBB 2021.13.0, OIIO 3.0.6.1, OpenEXR 3.3.6, Imath 3.1.12, Embree 4.4.0**. Moonray's pinned versions are not publicly stated, but its OptiX 7.6 lock and Python 3.9 reference suggest a 2023-era dep snapshot — likely Boost 1.78–1.82, OIIO 2.4–2.5, OpenEXR 3.1, TBB older release. Different Boost or TBB in the same process is undefined behavior. | **HIGH** | Build Moonray against Gaffer's bundled deps. Required source patches to Moonray are likely small (CMake `find_package` version checks; some Boost API renames between 1.82 → 1.85). OIIO 2.x → 3.x has more API churn — may need real porting work in moonshine's image-loading code. Budget 2–3 weeks of Moonray-side patching in Phase 0. |
| H3 | **First Hydra bridge in Gaffer.** No prior art in the tree — `grep` confirms zero existing references to `HdRenderIndex`/`HdSceneDelegate`/`UsdImagingDelegate` in `src/`. Risk: subtle dirtying bugs (transform edits not propagating, material rebind not re-shading) that don't appear in Cycles/Arnold because they bypass Hydra entirely. | **HIGH** | Architectural guidance from existing open implementations (Houdini Solaris, Maya MtoH, Katana hydraImage). Realistic effort: 2–4 engineer-months for IPR fidelity equivalent to `GafferCycles`. Spike-test interactive transform edits in Phase 2 before scaling up. |
| H4 | **No OSL on Moonray (and unlikely to change upstream).** GafferOSL is pervasive in Gaffer: shader expressions, light filters, surface networks. Cycles and Arnold both have an OSL runtime; Moonray does not, **by deliberate architectural choice.** Per the 2017 DreamWorks paper *Vectorized Production Path Tracing* (Lee & Green, HPG 2017), Moonray's whole performance story is breadth-first wavefront path tracing with ISPC (5× shading speedup). OSL's closure model + implicit integrator pattern conflicts with Moonray's explicit queue management. Search of the OpenMoonRay repo found **zero issues or discussions about OSL** — nobody is working on it. Adding it would require rewriting OSL's evaluation model (closures → explicit BSDF eval), call it 6+ engineer-months of R&D, and DreamWorks' contribution process round-trips PRs through internal review. Realistic acceptance probability for an upstream OSL contribution: **low**. Mixed networks (OSL nodes feeding into a `MoonrayShader`) will fail. | MEDIUM | v1: emit a clear error from `MoonrayShader::loadShader` when OSL nodes are upstream of a Moonray binding; document as a known limitation. v2: optional Gaffer-side OSL→moonshine network translator for the subset of OSL nodes with clean moonshine equivalents (lossy — can't preserve `oslc` programmability). Forking Moonray with OSL is technically possible but a high-maintenance dead end. |
| H5 | **Hydra IPR fidelity unproven for Moonray.** hdMoonray supports IPR per DreamWorks docs, but it's less battle-tested than hdCycles. Some Cycles IPR features (light-link toggles without restart, mid-render deformation samples) might force a full Hydra prim re-sync on Moonray. | MEDIUM | Spike-test in Phase 2: change a transform mid-render via `HdRetainedSceneIndex::DirtyPrims` and verify hdMoonray converges without restarting. If it can't, escalate scope; surface limits to user as known issues for v1. |
| H6 | **OptiX 7.6 hard-pin (XPU).** Moonray's GPU mode requires OptiX 7.6, not 8.x. NVIDIA driver back-compat is fine, but the SDK header version must be exactly 7.6 at build time. | MEDIUM | v2 only. Make XPU a build-time switch; ship CI builds with OptiX 7.6 redistributable. |
| H7 | **Arras requires out-of-process infrastructure.** Distributed mode needs a Coordinator / session manager / transport. Beyond the scope of "embed a renderer in Gaffer." | MEDIUM | v2 only. Wrap `Arras::SDK` behind a `MoonrayRenderer::command("moonray:arras:configure", …)` so users can flip to Arras transport at session creation. |
| H8 | **Linux-only release.** Gaffer ships Windows; this module won't. Build must gate cleanly. | LOW | Mirror the platform guard idiom used elsewhere — `requiredOptions: ["MOONRAY_ROOT"]` in SConstruct, `with IECore.IgnoredExceptions(ImportError): import GafferMoonray` in `startup/gui/viewer.py`. |
| H9 | **Build complexity.** Moonray spans 19 separate git repos with strict version pinning; building in CI is non-trivial. | LOW–MEDIUM | Use Moonray's official `/building` scripts on Rocky Linux 9. Don't try to integrate Moonray's build into Gaffer's SCons — link to a pre-built Moonray install via `MOONRAY_ROOT`. |
| H10 | **Object IDs through Hydra.** `Renderer.h` line 228 specifies `assignID(uint32_t)` for the picking AOV; Hydra has its own primId scheme. | LOW | Map Gaffer ID to a `primvars:hd:primId` data source on the Hydra prim and add a render setting telling hdMoonray to write that into the picking AOV. Verify behavior. |
| H11 | **No existing Cortex→Hydra utility in Gaffer.** `GafferUSD/USDLayerWriter.cpp` shows USD's Sdf is already linkable, but converting `IECoreScene::Primitive` into Hydra data sources is new code. Cortex's external `IECoreUSD::USDScene` translates to USD prims, not Hydra prims, so we can borrow the primvar-shape work but not the authoring path. | LOW | `MeshAlgo.cpp`, `CurvesAlgo.cpp`, `CameraAlgo.cpp` will be hand-written. Modest in scope. |

**Why Gaffer's other renderers don't use Hydra (user asked):** the
`IECoreScenePreview::Renderer` abstraction predates widespread Hydra delegates
and was designed for low-latency direct calls into renderer SDKs. USD support
in Gaffer (`GafferUSD/USDLayerWriter`, `GafferUSD/USDShader`) is purely for
scene I/O — read/write USD files. Gaffer can render without USD existing in
the pipeline. Going Hydra for Moonray is correct given DreamWorks recommends
hdMoonray, but acknowledges we're paving a new road inside Gaffer.

### Reference point: Houdini-MoonRay integration status

Houdini is DreamWorks' higher-priority DCC integration target than Gaffer.
Its current state is the closest real-world reference for the work we're
about to do. As of Feb 2026 (sourced from `dreamworksanimation/openmoonray`
issues + discussions):

- **Active community use exists.** Users post real shader / parameter
  questions about hdMoonray inside Houdini Solaris (e.g., kubo-von's
  shader-asset thread, answered Feb 24, 2026). The integration is real
  enough that practitioners hit and discuss substantive issues.
- **Issue #136 ("Building MoonRay for Houdini 20 / GCC 11.2") is
  unresolved.** Reporter hit `std::unique_ptr` template instantiation
  failures in `arras4_log/Logger.cc`. Workaround was "downgrade to GCC 9,"
  which the reporter found unacceptable. Exactly the toolchain-pinning
  conflict our Phase 0 will face.
- **"Houdini 21 compatibility?" discussion (Feb 25, 2026) is unanswered.**
  Strong signal that hdMoonray hasn't been forward-ported to Houdini 21's
  USD version yet — i.e., upstream Moonray itself isn't tracking the
  newest USD releases that DCCs use. We'll be doing that USD-version
  forward-port ourselves for Gaffer 1.6's USD 26.05.
- **Houdini-specific blockers** (the `hboost` namespace problem) are
  unresolved upstream. We don't inherit `hboost`, but we do inherit the
  analogous Python 3.11 / Boost 1.85 / USD 26.05 deltas.

Implications for the Gaffer port:
1. **The integration path is real.** Someone has hdMoonray rendering
   through a real DCC; we're not pioneering hdMoonray, only
   hdMoonray-via-Gaffer.
2. **The dep-pinning problem is real and recurring**, even for higher-priority
   DCC targets. We will hit similar build/version friction and will own the
   resolution without DreamWorks' help.
3. **The ~1 engineer-week-per-quarter ongoing fork-maintenance estimate is
   if anything optimistic.** Houdini hasn't kept pace with Houdini 21's USD
   release; if we want to track Gaffer's dep updates promptly, our cost is
   probably higher than that.

### Strategy: soft-fork Moonray with a patch overlay

Phase 0 work effectively means we are **forking Moonray to match Gaffer's
dependency stack.** Naming it honestly:

- **Maintain a soft fork** of `dreamworksanimation/openmoonray` on a
  Gaffer-side GitHub org.
- **Keep upstream Moonray clean.** Patches live in
  `dependencies/moonray-patches/*.patch` (or analogous in
  `GafferHQ/dependencies`), applied at build time during dep installation.
- **Rebase periodically** against new upstream Moonray releases (every
  2–3 months). Each rebase requires re-validating patches.
- **Submit subsets upstream** opportunistically — generic USD-26
  compatibility patches that don't change defaults may be acceptable to
  DreamWorks; Python-3.11 / Boost-1.85 bumps probably won't.

Why this and not the alternatives:
- **Hard fork** (no rebase strategy): drifts indefinitely; loses Moonray
  improvements. Don't.
- **Drag Gaffer's deps backward** to match Moonray: breaks `GafferUSD` and
  the rest of Gaffer's recent USD-aware code. Other Gaffer users would
  revolt. Not viable.
- **Wait for upstream to catch up:** indefinite blocking; not a strategy.

### Fallback: out-of-process via Arras (Plan B)

If Phase 0 blows past 8 weeks, or if soft-fork maintenance becomes
unsustainable, the escape hatch is **running Moonray entirely out-of-process
via the Arras framework.** Arras was designed for distributed rendering, but
its "local mode" puts Moonray in a separate process with its own dep stack,
talking to Gaffer over IPC. Each side keeps its own USD/Python/Boost — no
ABI sharing required.

- **Pros:** Phase 0 collapses to "install upstream Moonray as-is." Zero
  fork-maintenance burden. Multi-version coexistence becomes possible.
- **Cons:** Loses sub-frame IPR latency — every transform edit becomes an
  IPC round-trip + serialization, may not match the GafferCycles interactive
  feel. More complex deployment (need to ship the Arras Coordinator binary
  alongside Gaffer). Pixel data round-trip via `mcrt_dataio` streaming.

Plan A (in-process Hydra + soft-fork) and Plan B (out-of-process Arras)
are not mutually exclusive long-term: v1 could be in-process, v2 could add
Arras anyway, and v3 could fall back to Arras-only if the fork burden
exceeds the IPR-latency benefit.

## Recommended approach

Implement an in-process Hydra scene-index bridge, **split into two layers** so
the Hydra-bridge work is a reusable standalone contribution to GafferHQ
independent of Moonray:

- **Layer 1 — `IECoreSceneHydra`**: a generic `IECoreScenePreview::Renderer`
  base class that takes any Hydra render delegate plugin name as a constructor
  argument. Owns the `HdRenderIndex`, `HdRetainedSceneIndex`, `HdEngine`,
  `HdTaskController`, prototype/material caches, light linker. Translates
  Cortex `MeshPrimitive` / `CurvesPrimitive` / `Camera` / `VDBObject` into
  Hydra data sources. Knows nothing about Moonray.
- **Layer 2 — `IECoreMoonray`**: the Moonray-specific subclass. Configures
  Layer 1 with `HdMoonrayRendererPlugin`, registers `TypeDescription("Moonray")`
  and `TypeDescription("MoonrayXPU")`, owns moonshine SceneClass enumeration,
  Moonray render-setting names (`mny:*`), Moonray AOV name translation table.

Why split: the Hydra-bridge work is the architecturally novel part of this
project (no prior art in Gaffer — confirmed: zero references to `HdRenderIndex`
or `HdSceneDelegate` in the tree, and OpenUSD itself doesn't ship a generic
"scene-abstraction → Hydra" library). Splitting it gives GafferHQ a smaller,
standalone PR to review and de-risks the Moonray-specific work behind a stable
internal API. It also opens the door for future hdStorm / hdEmbree / hdPrman /
hdCycles support without rewriting the bridge.

Caveat: the generic bridge factors out roughly 30–40% of the work (Cortex →
Hydra prim translation, motion-blur sampling, prototype/material caching,
light linker). It does **not** eliminate per-renderer work for shader catalogs
(each Hydra delegate uses different `info:id` values), per-renderer light
types, AOV name translations, or interactive-edit escape hatches — those still
live in `IECoreMoonray`.

### Why Layer 1 has value to Gaffer beyond Moonray

`IECoreSceneHydra` is future-leverage, not just Moonray plumbing. Concrete
benefits if it ever ships:

- **Renderer pluralism almost for free.** Once the bridge exists, supporting
  any Hydra-equipped renderer is mostly configuration: HdStorm (Pixar's
  GL/Vulkan rasterizer — real-time viewport preview), HdEmbree (Intel's free
  reference path tracer), hdPrman, hdCycles, hdArnold, and the commercial GPU
  delegates (Redshift, V-Ray, Octane all ship Hydra delegates).
- **Industry alignment.** USD/Hydra is the de facto VFX interchange. Pixar,
  Apple, NVIDIA, Autodesk, SideFX (Houdini Solaris), Foundry (Katana), and
  DreamWorks have all bet on it. New renderers increasingly ship Hydra
  delegates as their primary DCC integration.
- **HdStorm viewport preview.** USD ships HdStorm; Gaffer's Viewer could use
  a Hydra-driven preview sharing the same scene graph as offline renders.
- **Maintenance offload.** Direct-SDK integrations track per-renderer ABI
  drift; Hydra delegates are maintained by the renderer vendor.

Trade-offs (why Gaffer hasn't done this already):

- **Latency.** Direct API integrations (Cycles' `ccl::Object` mutation,
  Arnold's `AiNode*` pokes) have been tuned for sub-frame interactive edits;
  Hydra adds a translation layer that may degrade IPR fidelity.
- **Feature fidelity.** Renderer-specific features (Arnold operators, Cycles
  light groups, RenderMan display filters) work best via direct API.
- **Existing investment.** Cycles/Arnold/RenderMan already work. Replacing
  them with Hydra-routed versions is regression risk for no user benefit.
- **Generic-bridge upkeep.** OpenUSD does not ship a generic
  "scene-abstraction → Hydra" library; Gaffer maintains it.

**The pragmatic line:** build Layer 1 Moonray-first with clean
renderer-agnostic API boundaries so a second consumer can land cheaply, but
do not actively design for hypothetical consumers (HdStorm, HdEmbree,
hdPrman) until one materializes. If GafferHQ's maintainers reject the split
during PR review, fall back to a Moonray-private bridge in
`src/IECoreMoonray/`.

End-to-end at runtime: every Gaffer scene call (`object()`, `light()`,
`attributes()`, `transform()`) authors / dirties Hydra prims via the retained
scene index. Interactive edits propagate as `HdSceneIndexObserver`
notifications, hdMoonray re-syncs into RDL2, Moonray IPR re-converges. No USD
file round-trip.

### `IECoreSceneHydra` public API sketch

The Layer-1 base class. Names are illustrative; final shape is a Phase-2
deliverable. Renderer-agnostic methods are concrete; renderer-specific hooks
are pure virtual for the subclass (`MoonrayRenderer`) to override.

```cpp
namespace IECoreSceneHydra
{

class HydraRenderer : public IECoreScenePreview::Renderer
{
    public :

        // RAII construction; subclass passes its delegate plugin name.
        HydraRenderer(
            const pxr::TfToken &delegatePluginName,        // e.g. "HdMoonrayRendererPlugin"
            RenderType renderType,
            const std::string &fileName,
            const IECore::MessageHandlerPtr &messageHandler
        );
        ~HydraRenderer() override;

        // Concrete (renderer-agnostic) — subclasses normally do NOT override.
        IECore::InternedString name() const override = 0;  // subclass returns its label
        void option( const IECore::InternedString &name, const IECore::Object *value ) override;
        void output( const IECore::InternedString &name, const IECoreScene::Output *output ) override;
        AttributesInterfacePtr attributes( const IECore::CompoundObject *attributes ) override;
        ObjectInterfacePtr camera(...) override;
        ObjectInterfacePtr object(...) override;
        ObjectInterfacePtr light(...) override;
        ObjectInterfacePtr lightFilter(...) override;
        void render() override;
        void pause() override;
        IECore::DataPtr command( const IECore::InternedString name, const IECore::CompoundDataMap & ) override;

    protected :

        // ---- Renderer-specific virtual hooks (subclass MUST override) ----

        // Attribute namespace: "moonray:" / "hdSt:" / "ri:" / etc.
        // Used to filter the IECore::CompoundObject in attributes() down to
        // this renderer's attribute set before authoring.
        virtual std::string attributePrefix() const = 0;

        // Map a Gaffer option name to the renderer's HdRenderSettingsMap key.
        // E.g. {"sampleMotion", BoolData} -> {"mny:sampleMotion", VtValue(true)}.
        virtual std::pair<pxr::TfToken, pxr::VtValue> translateOption(
            const IECore::InternedString &gafferName,
            const IECore::Object *value
        ) const = 0;

        // Map a light shader's `info:id` to its Hydra prim type token.
        // E.g. moonshine "RectLight" SceneClass -> HdPrimTypeTokens->rectLight.
        virtual pxr::TfToken lightPrimType( const pxr::TfToken &shaderId ) const = 0;

        // Map a Gaffer AOV name to the renderer's render-var token.
        // E.g. "rgba" -> "color" for hdMoonray; "albedo" -> "Albedo".
        virtual pxr::TfToken aovToken( const IECore::InternedString &gafferAovName ) const = 0;

        // Optional: massage the HdMaterialNetwork before authoring.
        // Default = no-op. Override for renderer-specific shader graph fixes.
        virtual void transformMaterialNetwork( pxr::HdMaterialNetwork2 &network ) const {}

        // Optional: Custom render-delegate setup (e.g. set a license path,
        // load extra DSOs). Called once during construction after the
        // delegate is instantiated. Default = no-op.
        virtual void configureDelegate( pxr::HdRenderDelegate *delegate ) {}

        // ---- Accessors for subclasses that need to author custom prims ----
        pxr::HdRetainedSceneIndex *retainedSceneIndex();
        pxr::HdRenderDelegate *renderDelegate();
};

} // namespace IECoreSceneHydra
```

```cpp
namespace IECoreMoonray
{

class MoonrayRenderer : public IECoreSceneHydra::HydraRenderer
{
    public :
        MoonrayRenderer(
            RenderType renderType,
            const std::string &fileName,
            const IECore::MessageHandlerPtr &messageHandler
        );

        IECore::InternedString name() const override { return "Moonray"; }

    protected :
        std::string attributePrefix() const override { return "moonray:"; }
        std::pair<pxr::TfToken, pxr::VtValue> translateOption( ... ) const override;
        pxr::TfToken lightPrimType( const pxr::TfToken &shaderId ) const override;
        pxr::TfToken aovToken( const IECore::InternedString & ) const override;
        void transformMaterialNetwork( pxr::HdMaterialNetwork2 & ) const override;

    private :
        static Renderer::TypeDescription<MoonrayRenderer> g_typeDescription;
        static Renderer::TypeDescription<MoonrayRenderer> g_typeDescriptionXPU;  // "MoonrayXPU"
};

} // namespace IECoreMoonray
```

This shape keeps Layer 1 ignorant of Moonray and gives the subclass exactly
five mandatory hooks plus two optional ones. A future `HdStormRenderer` would
implement the same five hooks; nothing else.

### Effort sizing (single senior engineer, dedicated)

| Phase | Best case | Likely | Worst case | Notes |
|---|---|---|---|---|
| Phase 0 — Moonray-against-Gaffer-deps port | 3 wk | 4 wk | 8 wk | **Revised after dep verification.** Confirmed mismatches: USD 26.05 vs likely 24.x/25.x, Python 3.11 vs 3.9, Boost 1.85 vs likely 1.78–1.82. Worst case = OIIO 2→3 API churn forces real source patches in moonshine. |
| Phase 1 — skeletons (both layers) | 1 wk | 1.5 wk | 2 wk | Mostly mechanical SConstruct work. |
| Phase 2 — minimal Hydra session | 3 wk | 4 wk | 6 wk | Highest variance. First Hydra bridge in tree; H3 + H5 bite here. |
| Phase 3 — full geometry + motion blur | 2 wk | 3 wk | 4 wk | Largely mirroring `IECoreRenderMan/*Algo.cpp` patterns. |
| Phase 4 — shader integration | 3 wk | 4 wk | 6 wk | Discovering moonshine SceneClass introspection API may have surprises (OQ-5). |
| Phase 5 — attributes / AOVs / options / adaptors | 2 wk | 2.5 wk | 3 wk | Well-templated by Cycles. |
| Phase 6 — viewer + tests + docs | 1 wk | 1.5 wk | 2 wk | |
| **v1 subtotal** | **15 wk** | **20.5 wk** | **31 wk** | **~3.5 / 5 / 7 months** |
| Phase 7 — XPU | 2 wk | 2.5 wk | 4 wk | OptiX 7.6 build-tooling fragility. |
| Phase 8 — Arras | 3 wk | 3.5 wk | 5 wk | Out-of-process infra; integration testing burden. |
| **v2 subtotal** | **5 wk** | **6 wk** | **9 wk** | **1.25 / 1.5 / 2.25 months** |
| **v1 + v2 grand total** | **20 wk** | **26.5 wk** | **40 wk** | **~5 / 6 / 9 months** |

Hidden costs not in the table:
- Build-pipeline work for CI: +2–4 wk for first integration, much less thereafter.
- Documentation, examples, test scenes: +1–2 wk per phase if user-facing.
- Code review and revision cycles with GafferHQ maintainers: 2–8 weeks of
  calendar time per major PR (Layer 1 PR, then Layer 2 PR), depending on
  reviewer availability and feedback depth. This is calendar time, not
  engineer-time.

Skill profile required:
- Strong C++17/20 (templates, RAII, modern idioms).
- Hydra / USD experience: `HdSceneIndex`, `HdRenderDelegate`, `HdEngine`
  internals. **Critical** — without this, Phase 2 doubles or triples.
- Familiarity with Gaffer's `IECoreScenePreview::Renderer` pattern (or
  willingness to read all four existing implementations end-to-end).
- Moonray / RDL2 familiarity (Phase 4 onward) — moderate; the SceneClass
  introspection API is small.

If staffing is two engineers, Phase 4 (shader work) can run in parallel with
Phase 3 (geometry); shave ~3 weeks off the likely-case total.

```
GafferScene → IECoreScenePreview::Renderer
                    ↓ (TypeDescription "Moonray" / "MoonrayXPU")
              MoonrayRenderer       (src/IECoreMoonray/Renderer.cpp)
                : public HydraRenderer
                                                        ┐
                    ↓                                   │
              HydraRenderer         (src/IECoreSceneHydra/Renderer.cpp)
                : public IECoreScenePreview::Renderer   │ ── Layer 1
                ├── HdRenderIndex                       │   (generic; reusable
                ├── HdRetainedSceneIndex   ← author    │    for hdStorm /
                ├── HdRenderDelegate (any plugin)      │    hdEmbree / hdPrman
                ├── HdEngine + HdTaskController        │    / hdCycles)
                ├── PrototypeCache                     │
                ├── MaterialCache                      │
                └── LightLinker                        ┘
                    ↓
              hdMoonray → RDL2 SceneContext → Moonray RenderContext → image
```

Per-prim: `MoonrayObject` / `MoonrayLight` / `MoonrayCamera` are
`ObjectInterface` subclasses that hold an `SdfPath` and route mutations into
the retained scene index. Mapping table:

| Gaffer call | Hydra prim type | Authoring API |
|---|---|---|
| `object(MeshPrimitive)` | `mesh` | `HdMeshSchema` |
| `object(CurvesPrimitive)` | `basisCurves` | `HdBasisCurvesSchema` |
| `object(VDBObject)` | `volume` | `HdVolumeSchema` |
| `light()` | `rectLight` / `sphereLight` / etc. | `HdLightSchema` |
| `camera()` | `camera` | `HdCameraSchema` |
| `attributes()` | (binding only) | material binding + visibility data sources |
| `output()` | `renderBuffer` + render settings | `HdAovSettingsSchema` |
| `option()` | `HdRenderSettingsMap` (`mny:*`) | `HdRenderDelegate::SetRenderSetting` |

Bypass `UsdImagingDelegate` entirely — author into `HdRetainedSceneIndex`
directly (modern scene-index API, USD ≥ 23.05). If hdMoonray only supports the
legacy `HdSceneDelegate` API (open question), fall back to a custom
`HdSceneDelegate` subclass — well-documented, ~2x the code.

## Phased implementation

### v1 — GafferCycles parity (estimated 4–6 engineer-months)

**Phase 0: Resolve H1 / H1b / H2 (3–4 weeks, blocking)**

Findings already confirmed (`GafferHQ/dependencies` inspection):

| Library | Gaffer 1.6 ships | Moonray expects (best guess) | Mismatch? |
|---|---|---|---|
| OpenUSD | **26.05** | 24.x or 25.x | YES |
| Python | **3.11.14** | **3.9** (CMakeLinuxPresets.json) | YES — confirmed |
| Boost | 1.85.0 | 1.78–1.82 (likely) | LIKELY |
| oneTBB | 2021.13.0 | older | LIKELY |
| OpenImageIO | 3.0.6.1 | 2.4–2.5 (likely) | LIKELY |
| OpenEXR | 3.3.6 | 3.1.x (likely) | maybe |
| Imath | 3.1.12 | 3.1.x | likely match |
| Embree | 4.4.0 | 3.x or 4.x | uncertain |
| OptiX | n/a (Gaffer doesn't ship) | 7.6 (hard pin, not 8.x) | n/a |

Concrete Phase 0 actions:
1. Build Moonray against Python 3.11. Patch `CMakeLinuxPresets.json` (the `python39` references) and verify Boost.Python and OIIO Python builds succeed.
2. Build Moonray against USD 26.05. Expect non-trivial patches — `HdSceneIndex` API changed between 24.x → 25.x and 25.x → 26.x. Track Moonray's master branch for any USD-26 prep work.
3. Build Moonray against Boost 1.85 / OIIO 3.0 / OpenEXR 3.3. Most likely lands cleanly with minor CMake patches; OIIO 2.x → 3.x has the most API churn and may require source changes in moonshine.
4. Run a smoke render with the patched Moonray standalone (not yet through Gaffer) to confirm it still works.
5. Don't proceed past Phase 0 until all four succeed and produce identical pixels to upstream Moonray on a reference scene.

**If Phase 0 takes more than 4 weeks**, reconsider whether to:
- Submit the patches upstream to OpenMoonRay (long calendar, low acceptance probability for breaking changes).
- Maintain a Moonray fork (real ongoing maintenance burden — every Moonray release requires re-patching).
- Pin Gaffer's deps backwards to match Moonray (breaks the rest of Gaffer's recent USD work; not viable).

This is the single largest source of schedule risk in the entire project.
The original "1 week" estimate was wrong — concrete data shows it's
3–4 weeks of Moonray-side dependency porting work before Gaffer code can
even start.

**Phase 1: Layer-1 build skeleton — `IECoreSceneHydra` (1–2 weeks)**
- Add SConstruct entry for new `IECoreSceneHydra` library (no `MOONRAY_ROOT` requirement; depends only on USD).
- Stub `src/IECoreSceneHydra/Renderer.{h,cpp}`: define `HydraRenderer` as an abstract subclass of `IECoreScenePreview::Renderer`. Constructor takes a Hydra render-delegate plugin name. Virtual hooks for subclass overrides (e.g., `defaultMaterialBinding()`, `translateRenderSettingName()`).
- Add `MOONRAY_ROOT` SConstruct option (mirror `RENDERMAN_ROOT` block at `SConstruct:177-180`).
- Add three Moonray-specific library entries: `IECoreMoonray`, `GafferMoonray`, `GafferMoonrayUI` (mirror `GafferCycles` at `SConstruct:1387-1419`).
- Stub `src/IECoreMoonray/Renderer.cpp`: subclass `HydraRenderer` with `HdMoonrayRendererPlugin`; register `IECoreScenePreview::Renderer::TypeDescription<MoonrayRenderer>("Moonray")`. All virtual methods log "not yet implemented" and return.
- Acceptance: Gaffer launches with `MOONRAY_ROOT` set, "Moonray" appears in the renderer dropdown.

**Phase 2: Layer-1 Hydra session — generic, in `IECoreSceneHydra` (3–4 weeks)**
- Create `src/IECoreSceneHydra/HydraSession.{h,cpp}`: construct `HdRenderIndex` / `HdRetainedSceneIndex` / `HdEngine` / `HdTaskController`, load the configured render-delegate plugin.
- Implement `HydraRenderer::camera()` → one `HdCamera` prim.
- Implement `HydraRenderer::object()` for `MeshPrimitive` only → `HdMesh`.
- Implement `HydraRenderer::light()` (delegates light-type → Hydra-prim-type mapping to subclass via virtual hook).
- Implement `HydraRenderer::output()` → `HdRenderBuffer` + AOV binding. Wire to `IECoreImage::DisplayDriver` (mirror `src/GafferCycles/IECoreCyclesPreview/IEDisplayOutputDriver.cpp`).
- Implement `HydraRenderer::render()` for both `Batch` (drive `HdEngine::Execute` to completion) and `Interactive` (engine thread with periodic `Execute`).
- Spike-test H5: change a transform mid-render via `HdRetainedSceneIndex::DirtyPrims`; confirm hdMoonray converges without restart.
- Acceptance: `gaffer execute --renderer Moonray` renders a quad with one light. **(`IECoreSceneHydra` is now ready to be a standalone PR to GafferHQ, independent of any Moonray work.)**

**Phase 3: Layer-1 geometry + motion blur — generic (2–3 weeks)**
- `src/IECoreSceneHydra/{MeshAlgo,CurvesAlgo,PointsAlgo,SphereAlgo,VDBAlgo,CameraAlgo}.cpp` — translate Cortex objects into Hydra data sources. Renderer-agnostic.
- Multi-sample motion blur via `HdSampledDataSource::GetContributingSampleTimesForInterval`.
- Geometry prototype caching (mirror `src/IECoreRenderMan/GeometryPrototypeCache.{cpp,h}`): hash-based dedup before authoring into Hydra.

**Phase 4: Shader integration — Moonray-specific in `IECoreMoonray` / `GafferMoonray` (3–4 weeks)**
- `src/GafferMoonray/SceneClassMetadataLoader.{h,cpp}`: query RDL2 SceneClass attributes via `scene_rdl2/scene/rdl2/SceneClass.h`'s introspection API. Cache via `IECorePreview::LRUCache<std::string, ConstCompoundDataPtr>` keyed by SceneClass name. Mirror `metadataGetter` at `src/GafferArnold/ArnoldShader.cpp:252-289`.
- `src/GafferMoonray/SocketHandler.{h,cpp}`: map RDL2 attribute types to Gaffer plugs (mirror `src/GafferCycles/SocketHandler.cpp`).
  - `INT/LONG` → `IntPlug`; `FLOAT` → `FloatPlug`; `RGB` → `Color3fPlug`; `RGBA` → `Color4fPlug`; `VEC2F`/`VEC3F` → `V2fPlug`/`V3fPlug`; `STRING` → `StringPlug`; `BOOL` → `BoolPlug`; `SCENE_OBJECT` (shader connection) → bare `Plug`.
- `src/GafferMoonray/MoonrayShader.cpp` (mirror `src/GafferCycles/CyclesShader.cpp`): `loadShader` walks SceneClass attributes via `SocketHandler`.
- `src/GafferMoonray/MoonrayLight.cpp`, `MoonrayLightFilter.cpp` (mirror `CyclesLight.cpp`): wrap moonshine light SceneClasses.
- `src/IECoreMoonray/ShaderNetworkAlgo.cpp`: translate `IECoreScene::ShaderNetwork` → `HdMaterialNetworkSchema` with `info:id` set to the moonshine SceneClass name.
- Bootstrap moonshine DSOs at module init via an RDL2 `SceneContext` in registration mode (analog to `IECoreArnold::UniverseBlock`).

**Phase 5: Attributes, light linking, AOVs, options, adaptors (2 weeks)**
- `src/IECoreMoonray/Attributes.{h,cpp}` (mirror `src/IECoreRenderMan/Attributes`): `moonray:` prefix; `moonray:visibility:*` similar to `ai:visibility:*`.
- `src/IECoreMoonray/LightLinker.{h,cpp}` (mirror RenderMan).
- `src/GafferMoonray/MoonrayOptions.cpp`, `MoonrayAttributes.cpp` (mirror `CyclesOptions.cpp` / `CyclesAttributes.cpp`).
- `startup/GafferMoonray/ocio.py` (mirror `startup/GafferArnold/ocio.py`): OCIO color manager render adaptor.
- `python/GafferMoonrayUI/`: metadata-driven UI generation (mirror `python/GafferCyclesUI/CyclesShaderUI.py`).

**Phase 6: Viewer integration + tests (1–2 weeks)**
- `startup/gui/viewer.py`: register Moonray (mirror Cycles block at `:252-261`).
- `startup/gui/shaderView.py`: register `MoonrayShaderBall`.
- `python/GafferMoonrayTest/`, `python/GafferMoonrayUITest/`.

### v2 — Production grade

**Phase 7: XPU GPU mode (2–3 weeks)**
- Register a second renderer type: `TypeDescription<MoonrayRenderer>("MoonrayXPU")` (mirror `RenderManXPU`).
- Set `mny:device=xpu` render setting in the XPU constructor.
- SConstruct: detect OptiX 7.6, gate XPU on a compile-time `MOONRAY_XPU_AVAILABLE` define.

**Phase 8: Arras distributed (3–4 weeks)**
- `MoonrayRenderer::command("moonray:arras:configure", {host, port, sessionId, …})`: set up `Arras::SDK` client transport.
- Render setting `mny:transport=arras` switches hdMoonray to render through Arras.
- Surface Arras worker state via `RenderMessageHandler` (see `src/GafferScene/InteractiveRender.cpp:103`).

**Phase 9: OSL bridge investigation (deferred)**
- Spike: OSL → ISPC transpilation, or wrap `oslc` output in a moonshine DSO. Out of scope for v1/v2.

## Files to create

**Source (new — Layer 1 `IECoreSceneHydra`, generic):**
- `include/IECoreSceneHydra/Export.h`
- `src/IECoreSceneHydra/{Renderer,HydraSession,Globals,Session,Attributes,Light,LightLinker,MaterialCache,GeometryPrototypeCache,Camera,Object}.{h,cpp}`
- `src/IECoreSceneHydra/{MeshAlgo,CurvesAlgo,PointsAlgo,SphereAlgo,VDBAlgo,CameraAlgo,ShaderNetworkAlgo}.cpp`
- `src/IECoreSceneHydraModule/IECoreSceneHydraModule.cpp`

**Source (new — Layer 2 `IECoreMoonray` / `GafferMoonray`, Moonray-specific):**
- `include/IECoreMoonray/Export.h`
- `src/IECoreMoonray/{Renderer,LightFilter,SceneClassRegistry}.{h,cpp}` (subclasses + Moonray-only types)
- `src/IECoreMoonrayModule/IECoreMoonrayModule.cpp`
- `src/GafferMoonray/{MoonrayShader,MoonrayLight,MoonrayLightFilter,MoonrayMeshLight,MoonrayAttributes,MoonrayOptions,MoonrayBackground,SceneClassMetadataLoader,SocketHandler}.cpp`
- `src/GafferMoonrayModule/GafferMoonrayModule.cpp`
- Headers under `include/IECoreMoonray/` and `include/GafferMoonray/`.

**Python (new):**
- `python/GafferMoonray/__init__.py`, `MoonrayShaderBall.py`
- `python/GafferMoonrayUI/{__init__,MoonrayShaderUI,MoonrayLightUI,MoonrayAttributesUI,MoonrayOptionsUI,MoonrayShaderBallUI,ShaderMenu}.py`
- `python/GafferMoonrayTest/`, `python/GafferMoonrayUITest/`

**Startup (new):**
- `startup/GafferMoonray/{ocio,userDefaults,renderCompatibility}.py`

**Modify:**
- `SConstruct`: add `IECoreSceneHydra` library entry (USD-only deps), add `MOONRAY_ROOT` option, add three Moonray-specific library entries.
- `startup/gui/viewer.py`: add Moonray block (mirror lines 252–261 for Cycles).
- `startup/gui/shaderView.py`: register `MoonrayShaderBall`.

**Submission strategy:** The Layer-1 `IECoreSceneHydra` library is a coherent
standalone change to GafferHQ — submit it as a separate PR after Phase 3,
before any Moonray-specific code. Reviewers can evaluate the Hydra-bridge
design without Moonray context. Layer-2 lands in subsequent PRs.

## Files to reuse as templates

The five most load-bearing templates — internalize these before writing new code:

- `/home/user/gaffer/include/GafferScene/Private/IECoreScenePreview/Renderer.h` — the contract; read all comments.
- `/home/user/gaffer/src/IECoreRenderMan/Renderer.cpp` — thinnest renderer in the tree; closest in spirit to a Hydra-bridge wrapper. Plus its `Globals.{h,cpp}`, `Session.{h,cpp}`, `MaterialCache.{cpp,h}`, `GeometryPrototypeCache.{cpp,h}`, `LightLinker.{cpp,h}`.
- `/home/user/gaffer/src/GafferCycles/IECoreCyclesPreview/Renderer.cpp` — most fully-featured; reference for IPR semantics, options buffering, output handling. Plus `IECoreCycles.cpp` (module init / SceneClass enumeration), `IEDisplayOutputDriver.cpp` (DisplayDriver hookup).
- `/home/user/gaffer/src/GafferCycles/{SocketHandler,CyclesShader,CyclesLight,CyclesOptions,CyclesAttributes}.cpp` — Gaffer-side node patterns.
- `/home/user/gaffer/src/GafferArnold/ArnoldShader.cpp` lines 252–289 — canonical metadata-cache pattern for shader plug generation.
- `/home/user/gaffer/SConstruct` lines 1387–1419 (`GafferCycles` block) — build template.
- `/home/user/gaffer/startup/GafferArnold/ocio.py` — render adaptor template.

## Open questions (resolve during the noted phase)

1. **Phase 0 blocker:** What USD version does Moonray require? Does it match Gaffer's bundled USD?
2. **Phase 2:** Does hdMoonray support the modern Hydra Scene Index API (USD ≥ 23.05) or only the legacy `HdSceneDelegate`? Determines whether we author into `HdRetainedSceneIndex` or implement a custom delegate (~2× the code).
3. **Phase 4:** What is hdMoonray's light-filter representation — material relationship, custom prim type, or unsupported?
4. **Phase 3:** Does hdMoonray accept >2 motion-blur samples (Cycles allows it), or pin to shutter-open / shutter-close?
5. **Phase 4:** Is there a moonshine API to enumerate registered light SceneClasses, or do we hardcode the catalog?
6. **Phase 2:** How does `pause()` (Renderer.h:325) map to hdMoonray — flag-poll, or stop-the-engine-thread?
7. **Phase 5:** AOV naming — what render-var tokens does hdMoonray accept? Need a translation table to standard Gaffer AOV names.
8. **Phase 5:** Background/environment — moonshine `EnvLight` SceneClass, or a SceneVariable? Determines whether `MoonrayBackground.cpp` is a Light or Options node.
9. **Phase 2:** SceneDescription render type — supported? RenderMan rejects it, Cycles supports it. Default to v1 supports `Batch` + `Interactive` only (mirror RenderMan).
10. **Phase 5:** Object IDs — confirm the Hydra primId → Moonray picking AOV path before relying on `assignID()` for Viewer selection feedback.

## Verification

Acceptance gates per phase (run end-to-end, not unit tests):

- **Phase 1:** `gaffer` launches with `MOONRAY_ROOT` set; "Moonray" appears in the renderer dropdown of any `Render` or `InteractiveRender` node.
- **Phase 2:** `gaffer execute -script tests/moonray/quad.gfr` renders a one-quad-one-light scene to EXR via `Render` node with `renderer="Moonray"`. Interactive transform edit in the Viewer converges without crash.
- **Phase 3:** Render a moving sphere with motion blur; AOVs (color, depth) resolve via `IECoreImage::DisplayDriver` to the Catalogue.
- **Phase 4:** Build a Lambert-equivalent moonshine network in the Node Graph; render it on a sphere via the shader-ball viewer.
- **Phase 5:** Light-link two lights to two objects; render and verify only the linked lights illuminate each object. OCIO render adaptor injects color manager automatically.
- **Phase 6:** `python -m unittest GafferMoonrayTest` passes; Viewer shows interactive Moonray feedback comparable to GafferCycles.
- **v2 Phase 7:** Same scene renders ≥1.5× faster with `renderer="MoonrayXPU"` on an OptiX 7.6 GPU.
- **v2 Phase 8:** Same scene renders against an Arras coordinator; worker state visible in Gaffer's render messages panel.
