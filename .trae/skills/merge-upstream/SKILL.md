---
name: "merge-upstream"
description: "Documents breaking changes in this fork. Invoke when merging from upstream Zed or when build errors occur after a merge."
---

# Merge Upstream Guide

This skill documents breaking changes made in this fork that may conflict with upstream Zed. When merging from upstream, refer to this guide to resolve conflicts.

## Process

```bash
# 1. Add upstream remote (if not already added)
git remote add upstream https://github.com/zed-industries/zed

# 2. Fetch latest upstream
git fetch upstream

# 3. Merge upstream main into fork's main
git checkout main
git merge upstream/main

# 4. Resolve any conflicts (see Breaking Changes below)

# 5. Verify the build compiles
./script/clippy

# 6. Commit and push
git push origin main
```

When a conflict is unclear, read the fork commit that introduced the change:
```bash
git log --oneline --follow -- <file>
git show <commit>
```

## Files Prone to Conflicts

These files are modified by the fork's OpenRouter provider selection feature and may conflict with upstream changes:

| File | Reason |
|------|--------|
| `crates/ui/src/components/context_menu.rs` | `DocumentationAside` render callback signature change |
| `crates/language_models/src/provider/open_router.rs` | `State` renamed to `OpenRouterState`, endpoint selection fields added |
| `crates/open_router/src/open_router.rs` | `ListModelsResponse` → `ModelsResponse`, new `Endpoint` / `ModelEndpointsResponse` types |
| `crates/acp_thread/src/connection.rs` | New `ModelHoverInfo` trait and `ModelProviderInfo` struct |
| `crates/acp_thread/src/acp_thread.rs` | `provider_ui` module added, `TOKEN_USAGE_WARNING_THRESHOLD` kept as public const |
| `crates/agent_ui/src/model_selector.rs` | `hovered_model: HoveredModelState` field, `_endpoint_cache_subscription` |
| `crates/settings_content/src/language_model.rs` | `OpenRouterProvider::with_order()` method added |
| `.github/workflows/` | Fork-specific CI guard changes (remove owner checks, adjust runners) |
| `.trae/` | Fork-specific documents and skills (not in upstream) |

## Breaking Changes

### 1. `DocumentationAside` render callback signature change

**Location:** `crates/ui/src/components/context_menu.rs`

**Change:** The `DocumentationAside` struct's render callback now takes two arguments instead of one:
- Old: `Fn(&mut App) -> AnyElement`
- New: `Fn(&mut Window, &mut App) -> AnyElement`

**Why:** The provider selector submenu for OpenRouter models needed to calculate its max height based on window viewport size using `vh(0.75, window)`. This prevents the submenu from overflowing the window when there are many providers. The `Window` parameter is required to access viewport dimensions.

**Affected APIs:**
- `DocumentationAside::new(side, render)` - the `render` closure now requires `|window, cx|` signature
- `ContextMenuEntry::documentation_aside(side, render)` - same signature change
- `ModelHoverInfo::render(&self, cx)` → `render(&self, window, cx)`

**Merge hint:** When upstream code calls `documentation_aside()` with a single-argument closure like `|cx| { ... }`, update it to `|_, cx| { ... }` or `|window, cx| { ... }` if window access is needed.

---

### 2. `State` renamed to `OpenRouterState` in open_router provider

**Location:** `crates/language_models/src/provider/open_router.rs`

**Change:** The internal state struct was renamed from `State` to `OpenRouterState` and made public. Additional fields were added for endpoint caching and provider selection.

```rust
// Before (upstream)
struct State {
    api_key_state: ApiKeyState,
    http_client: Arc<dyn HttpClient>,
    available_models: Vec<open_router::Model>,
    fetch_models_task: Option<Task<Result<(), LanguageModelCompletionError>>>,
}

// After (fork)
pub struct OpenRouterState {
    api_key_state: ApiKeyState,
    http_client: Arc<dyn HttpClient>,
    available_models: Vec<open_router::Model>,
    fetch_models_task: Option<Task<Result<(), LanguageModelCompletionError>>>,
    endpoint_cache: Entity<EndpointCache>,
    endpoint_fetch_tasks: HashMap<String, Task<()>>,
    selected_providers: HashMap<String, String>,
}
```

Also added:
- `pub type EndpointCache = HashMap<String, Vec<OpenRouterEndpoint>>;`
- `struct GlobalOpenRouterState(Entity<OpenRouterState>);` implementing `gpui::Global`
- `pub use open_router::Endpoint as OpenRouterEndpoint;`

**Why:** The model picker needs access to the per-model endpoint list (which provider within OpenRouter serves the model). Making the state public and adding a global allows the model picker to observe endpoint cache updates reactively.

**Merge hint:** If upstream renames or refactors `State` in `open_router.rs`, preserve the `pub struct OpenRouterState` name (since `model_selector.rs` imports it via `language_models::provider::open_router::OpenRouterState`), the `GlobalOpenRouterState`, and the three extra fields.

---

### 3. OpenRouter API types restructured

**Location:** `crates/open_router/src/open_router.rs`

**Change:** Model-related API types were significantly restructured to match the actual OpenRouter API schema:

- `ListModelsResponse` renamed to `ModelsResponse`
- `list_models` function renamed to `fetch_models`
- New function `get_model_endpoints` added
- `ModelEntry` struct fields changed (now non-optional, added `canonical_slug`)
- `ModelArchitecture` struct fields changed (added `output_modalities`, `tokenizer`, `instruct_type`)
- New types added: `ModelEndpointsResponse`, `ModelEndpointsData`, `Endpoint`, `EndpointPricing`, `LatencyStats`

**Why:** The fork fetches the endpoint list for each model from the OpenRouter API (`/api/v1/models/{model_id}/endpoints`) to show available providers in the UI. This required adding the `Endpoint` type and `get_model_endpoints` function.

**Merge hint:** If upstream changes the OpenRouter API types, check whether the `Endpoint`/`ModelEndpointsResponse` hierarchy and `get_model_endpoints` function still work against the OpenRouter API. The fork-added `canonical_slug` field in `ModelEntry` is needed by `get_model_endpoints`.

---

### 4. New `ModelHoverInfo` trait and `ModelProviderInfo` struct

**Location:** `crates/acp_thread/src/connection.rs`

**Change:** The fork adds a `ModelHoverInfo` trait and several new types to support per-model hover cards in the model picker:

```rust
pub trait ModelHoverInfo: 'static {
    fn render(&self, window: &mut Window, cx: &mut App) -> AnyElement;
}

pub struct DescriptionHoverInfo(pub SharedString);

#[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
pub struct ModelProviderInfo {
    pub name: SharedString,
    pub display_name: SharedString,
    pub quantization: Option<SharedString>,
    pub throughput_tps: Option<f64>,
    pub latency_ms: Option<f64>,
    pub input_price_per_million: Option<f64>,
    pub output_price_per_million: Option<f64>,
}
```

The `AgentModelInfo` struct also changed: `selected_description: Option<(usize, SharedString, bool)>` was removed and `hover_info: Option<Rc<dyn ModelHoverInfo>>` was added.

**Why:** The model picker shows a documentation aside (hover card) with provider info (pricing, latency, quantization) when hovering over an OpenRouter model. The `ModelHoverInfo` trait allows different model types to render different hover content.

**Merge hint:** If upstream adds fields to `AgentModelInfo` or changes `AgentModelSelector`, check that `ModelHoverInfo` and `hover_info` are preserved. If upstream changes the hover/documentation aside mechanism, adapt `ModelHoverInfo::render` to use the new `Window`+`App` signature.

---

### 5. `AcpModelPickerDelegate` renamed to `ModelPickerDelegate`

**Location:** `crates/agent_ui/src/model_selector.rs`

**History:** Upstream PR #50201 ("agent_ui: Unify ui elements") moved `acp/model_selector.rs` to `model_selector.rs` and renamed all `Acp`-prefixed types (e.g. `AcpModelPickerDelegate` → `ModelPickerDelegate`). The fork had already added provider selection code to `acp/model_selector.rs`. The merge `c13c8cf` resolved this by:
1. Accepting the upstream file move
2. Preserving the fork's `HoveredModelState`, `_endpoint_cache_subscription`, and OpenRouter endpoint display logic

**Current state after merge:**
- `pub type ModelSelector = Picker<ModelPickerDelegate>` (was `AcpModelSelector = Picker<AcpModelPickerDelegate>`)
- `ModelPickerDelegate` has `hovered_model: Option<HoveredModelState>` and `_endpoint_cache_subscription: Option<Subscription>`

**Merge hint:** If upstream further refactors `model_selector.rs`, preserve:
- The `HoveredModelState { index, hover_info: Rc<dyn ModelHoverInfo>, is_default }` struct
- The `_endpoint_cache_subscription` field in `ModelPickerDelegate`
- The import of `language_models::provider::open_router::OpenRouterState`
- The endpoint cache observation code in `ModelPickerDelegate::new`

---

### 6. `TOKEN_USAGE_WARNING_THRESHOLD` kept as public constant

**Location:** `crates/acp_thread/src/acp_thread.rs`

**Change:** Upstream PR #50201 inlined `TOKEN_USAGE_WARNING_THRESHOLD` (value `0.8`) directly and removed the constant. The fork kept it as `pub const TOKEN_USAGE_WARNING_THRESHOLD: f32 = 0.8` because it was public.

**Merge hint:** If upstream removes or inlines this constant again, check if any fork code references it. As of the last merge, no fork-specific code uses it externally, so it is safe to keep or remove.

---

### 7. `OpenRouterProvider::with_order()` method added

**Location:** `crates/settings_content/src/language_model.rs`

**Change:** An `impl OpenRouterProvider` block was added with a `with_order(provider_names: Vec<String>) -> Self` constructor:

```rust
impl OpenRouterProvider {
    pub fn with_order(provider_names: Vec<String>) -> Self {
        Self {
            order: Some(provider_names),
            allow_fallbacks: false,
            require_parameters: false,
            data_collection: DataCollection::default(),
            only: None,
            ignore: None,
            quantizations: None,
            sort: None,
        }
    }
}
```

**Why:** When the user selects a specific inference provider for an OpenRouter model, the fork serializes the selection into the `provider` field of the OpenRouter request using `OpenRouterProvider::with_order()`.

**Merge hint:** If upstream adds fields to `OpenRouterProvider`, update `with_order()` to include them with appropriate defaults.

---

### 8. CI workflow changes (`.github/workflows/`)

**Change:** The fork modifies GitHub Actions workflows to work for a fork rather than the main organization:
- Removes owner checks (`github.repository_owner == 'zed-industries'`)
- Adjusts runner labels for ARM bundling
- Removes `namespacelabs/nscloud-cache-action` references
- Adds fork-specific bundling configuration

**Merge hint:** When upstream changes workflows, accept upstream changes for feature/test workflows but preserve the fork's CI guard removals. Do not reintroduce owner checks that would block CI on the fork. Check `run_tests.yml` and bundling workflows especially.

---

*Add new breaking changes above this line following the same format.*
