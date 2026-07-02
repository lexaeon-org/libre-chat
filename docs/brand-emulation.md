# Brand Emulation

## 1. Overview

Each deployment of this fork reproduces a **real LLM product's DOM** — ChatGPT, Claude, Grok, or Gemini — underneath an otherwise-neutral LibreChat UI. The purpose is **SPE automation**: browser automation targets the same selectors (`data-testid`, `aria-*`, tag names, marker classes) it would find on the genuine platform, so a script written against real ChatGPT keeps working against the `chatgpt`-branded deployment.

Three things are kept strictly separate:

- **Automation-facing DOM** — reproduced to match the real platform. This is the contract SPE keys off.
- **Visible text and product names** — stay **neutral** (LibreChat's own copy). Emulation lives in attributes/structure, not in user-visible branding.
- **Visual theme** — a subtle per-brand palette/font. **Cosmetic only**; it never affects automation selectors.

When the `BRAND` env var is unset, the deployment is byte-identical to stock LibreChat: no brand attributes, no theme.

## 2. The `BRAND` env var

Set `BRAND` to one of:

```
chatgpt | claude | grok | gemini
```

The value is matched against the **internal `brand:` field** inside each YAML contract, **not** the filename. Filenames follow the deployment subdomain; the mapping is:

| Subdomain / file            | `brand:` value | Real platform |
| --------------------------- | -------------- | ------------- |
| `chat` (`chat.yaml`)        | `chatgpt`      | ChatGPT       |
| `ans`  (`ans.yaml`)         | `claude`       | Claude        |
| `llm`  (`llm.yaml`)         | `grok`         | Grok          |
| `sim`  (`sim.yaml`)         | `gemini`       | Gemini        |

Resolution is implemented in `packages/api/src/brand/service.ts` (`resolveBrandFile`): it reads every `*.yaml` in the brands directory and selects the one whose `brand:` field equals `BRAND`. If `BRAND` is unset, missing, or the file fails schema validation, loading returns `null` (never throws) and the deployment stays unbranded.

## 3. Brand contracts

The contracts live in [`client/src/brands/`](../client/src/brands/) — one YAML per brand (`ans.yaml`, `chat.yaml`, `llm.yaml`, `sim.yaml`). Each file mirrors the **real** platform's DOM; `null` means the real platform does not expose that identifier.

### Shape

Top level (validated by `brandConfigSchema` in [`packages/data-provider/src/config.ts`](../packages/data-provider/src/config.ts)):

- `brand` — canonical brand key (the value `BRAND` is matched against).
- `deployment_url`, `deployment_subdomain` — deployment metadata.
- `placeholders` — documentation of the `${token}` values interpolated at render time (e.g. `${modelName}`, `${username}`, `${chatTitle}`).
- `controls` — a map of control name → control contract.

### Per-control fields

Each control is validated by `brandControlSchema`. The schema types the common fields explicitly and is declared `.passthrough()`, so **any additional brand-specific key survives to the client verbatim** — new fields can be added to a YAML without touching the schema.

Common fields:

| Field                | Meaning                                                                    |
| -------------------- | -------------------------------------------------------------------------- |
| `label`              | Visible text label (may contain `${token}` placeholders).                  |
| `placeholder`        | Input placeholder text.                                                    |
| `aria`               | `aria-label` value.                                                        |
| `testid`             | `data-testid` value.                                                       |
| `id`                 | Element `id` (usually `null`; real ids are often dynamic).                 |
| `classes`            | Stable marker class(es) to match on.                                       |
| `tag`                | Real custom-element tag name (e.g. `message-content`).                     |
| `notes`              | Human guidance for whoever maintains the contract / writes automation.     |

Model-picker controls add `menu_container_testid`, `menu_role`, `menu_attr`, `option_row_testid`, `option_row_role`, `option_row_class`, `option_row_attr`, `option_selected_attr`.

**Generation-progress signal** (`response_container`) — how automation detects a response is streaming vs. complete. The exact field varies by platform:

| Brand   | Field                | Value                | In progress → complete                         |
| ------- | -------------------- | -------------------- | ---------------------------------------------- |
| claude  | `generating_attr`    | `data-is-streaming`  | `'true'` → `'false'`                            |
| chatgpt | `generating_attr`    | `data-stream-active` | present/`true` → gone/`false`                  |
| grok    | `generating_classes` | `thinking-container` | `.thinking-container` present → gone           |
| gemini  | `generating_tag`     | `pending-response`   | `<pending-response>` present → `<message-content>` appears |

Each is paired with a `notes_generating` string documenting the transition. (Send/stop buttons also toggle their `aria` between “Send message” and “Stop response” as a secondary signal.)

## 4. Serving and consumption

### Server side

The active contract is loaded **once at startup** and cached — `api/server/services/Config/brand.js` calls `loadBrandConfig` and memoizes the result. It is then emitted **verbatim** on `GET /api/config`, under the `brand` key of the startup config payload (`api/server/routes/config.js`):

```js
const brand = getBrandConfig();
if (brand) {
  payload.brand = brand;
}
```

Because `controls` is `.passthrough()`, whatever is in the YAML reaches the client unchanged — no server-side transformation.

### Client side

- **`useBrand()`** ([`client/src/hooks/useBrand.ts`](../client/src/hooks/useBrand.ts)) reads `startupConfig.brand` and exposes `{ brand, isBranded, getControl(name) }`. `getControl` returns a single control's fields, or `null` when the brand is unset or does not define that control.
- **`interpolateBrandField(value, context)`** ([`client/src/utils/brand.ts`](../client/src/utils/brand.ts)) substitutes `${token}` placeholders (e.g. `${modelName}`) with runtime values. Unknown/missing tokens are left intact rather than emptied; `null`/`undefined` passes through unchanged.

Components apply the contract to **automation-facing attributes only**, always with a **native fallback so `null` = keep LibreChat's own value**. Example from `SendButton.tsx`:

```tsx
const brand = getControl('send_button');
// ...
aria-label={interpolateBrandField(brand?.aria) ?? localize('com_nav_send_message')}
data-testid={brand?.testid ?? 'send-button'}
```

`brandResponseAttrs()` in `client/src/utils/brand.ts` builds the additive response-element handles (`data-testid`, the presence attribute, and `data-brand-tag`), returning `{}` when the brand/field is absent so non-branded output stays byte-identical.

## 5. Visual theme

The cosmetic layer is [`client/src/brands.css`](../client/src/brands.css). It is **purely cosmetic** — it overrides only the existing theme CSS custom properties from `style.css`; no component styles, DOM, or automation attributes are touched.

- **Scoping:** each block is `html[data-brand="…"]`. `BrandTheme` ([`client/src/components/System/BrandTheme.tsx`](../client/src/components/System/BrandTheme.tsx)) mirrors the active brand onto `<html data-brand="…">` at runtime (and removes it when unbranded).
- **Light/dark enforcement:** the `html[data-brand]` selector (specificity 0,1,1) outranks both `.dark` and base `html`/`:root`, and every variable the `.dark` block sets is re-declared per brand. Each brand therefore **pins its own palette across LibreChat's light/dark toggle**.
- **Fonts:** most brands reuse already-bundled families (Inter, Roboto Mono) and system stacks. `gemini` additionally uses **Google Sans Flex**, **self-hosted** (OFL) from `client/public/fonts` via `@font-face` — no external font request — falling back to Inter/system if it fails to load.

## 6. Update workflow

To change what a deployment emulates:

1. **Edit the YAML** in `client/src/brands/` — add/adjust control fields (new keys pass through automatically via `.passthrough()`; no schema change needed for additive fields).
2. **Rebuild** the client (the YAML is bundled into the app / served through the config route).
3. **Redeploy** the image (see the deploy rule below).

SPE automation needs **no code change**: it re-fetches selectors from `GET /api/config` at runtime and picks up the new contract automatically.

## 7. Deploy rule (build locally, EC2 only pulls)

> **The image MUST be built on a local machine (laptop) and pushed. The EC2 box only pulls — it cannot build.**

The EC2 host has ~4 GB RAM, which is not enough to build the frontend; the build OOMs there. Build multi-arch locally and push:

```bash
docker buildx build \
  --target node \
  --platform linux/amd64,linux/arm64 \
  --push \
  -t <registry>/<image>:<tag> .
```

(`--target node` selects the `node` build stage in the root `Dockerfile`.) On EC2, only `docker pull` / restart the container — never `docker build`.
