---
name: image-to-component
description: Use when the user points to a directory of UI screenshots or design mockup images and wants component skeleton code generated. Triggers when images show app screens, pages, or UI states. Generates React, Vue 3, or Vue 2 skeletons with TypeScript or JavaScript using BEM or CSS Modules.
---

# image-to-component

## Overview

Convert a directory of UI screenshots into typed component skeleton(s). The critical step is **structural comparison first** — multiple images often represent one component in different states, not multiple components.

**Hard context boundary:** The main agent must never call Read on image files. All image reading happens inside signature subagents. If subagent dispatch is unavailable in the current environment, pause and ask the user how to proceed; do not silently fall back to direct image reading.

## Workflow

### Step 1 — Gather context (ask once, upfront)

Before reading images, confirm if not already provided.

**主要方式**：使用 Questions 工具（或平台提供的交互式选择工具），为每个问题单独呈现可点击的选项，用户无需打字。四个问题依次提问：

问题 1 — 框架：
- React（推荐，完整支持）
- Vue 3
- Vue 2
- 其他框架（仅给出结构建议，不生成可运行骨架）

问题 2 — 输出方式：
- 只在对话里输出目录结构和骨架代码
- 直接写入项目文件

问题 3 — 语言：
- TypeScript（推荐）
- JavaScript

问题 4 — 样式技术栈：
- CSS Modules（推荐，新项目）
- plain CSS + BEM（兼容性最好，全局样式）

**降级方案**：如果当前环境不支持交互式选择工具，则改为文字选择题格式：

在开始读图前，先确认 4 个设置：

**① 框架**
A. React（推荐，完整支持）
B. Vue 3
C. Vue 2
D. 其他框架（degraded 模式，仅给结构建议）

**② 输出方式**
A. 只在对话里输出目录结构和骨架代码
B. 直接写入项目文件

**③ 语言**
A. TypeScript（推荐）
B. JavaScript

**④ 样式技术栈**
A. CSS Modules（推荐，新项目）
B. plain CSS + BEM（兼容性最好，全局样式）

直接回 "AAAA" 即可采用所有推荐默认值。

Do NOT assume. Do NOT start reading images yet.

### Step 2 — Capture user intent

Before doing any structural comparison, check whether the user's initial message **already declares the relationship between images**.

**Signals that count as a declaration:**
- Explicit state labels: "图1是 pending，图2是 used，图3是 expired"
- Group statement: "这 3 张图是同一组件的不同状态"
- English equivalents: "these are state variants of the same page", "each image is a different status"

**If a declaration is found:**
1. Record the stated intent (e.g., "user declared: same component, states = pending / used / expired").
2. Use this declaration as an input fact. Do not ask the user to re-decide the same relationship later.

**If no declaration is found** (e.g., "把这个目录的图转成组件", "convert these images"):
- Continue as normal; the structural decision will be made from the generated signatures.

### Step 3 — List files and create read batches

First, run `ls <directory>` (or equivalent) to get the exact file list.

**Never assume the file list from memory or user description** — always `ls` first to prevent hallucinating files that don't exist.

**Image count guard** (run as soon as the `ls` result is available):

- If the directory contains 0 images → stop and ask the user to point at a directory containing screenshots.
- If the directory contains 1–20 images → proceed normally.
- If the directory contains 21–50 images → pause and confirm with the user:

  > The directory contains <N> images, which will dispatch ~<ceil(N/5)> signature subagents in parallel and produce a cross-batch comparison set of size <N>. Please confirm:
  > A. Proceed with all <N> images.
  > B. Provide a filtered subset (list specific filenames, comma-separated; e.g., pending.png, used.png, expired.png).
  > C. Cancel.

- If the directory contains > 50 images → refuse to proceed automatically. Show the same prompt as above but with option A removed; require the user to either filter (B) or cancel (C).

This guard runs once per skill invocation. If the user picked B in either band, restart Step 3 with only the filtered filenames; the count guard is not re-evaluated against the original full list.

Do not read images in this step. Only create a read-batch plan:
- If there are **≤ 5 images**, create one read batch containing all images.
- If there are **> 5 images**, first do mechanical filename pre-grouping, then split into read batches of at most 5 images.

For directories with > 5 images, apply filename pre-grouping in priority order:

| Rule | Grouping method |
|---|---|
| Filename contains a status keyword (`pending` / `used` / `expired` / `active` / `disabled`) | Place in the same candidate state group |
| Filename contains a sequence keyword (`page1` / `page2` / `step1` / `step2`) | Place in the same candidate sequence group |
| All other files | Fill batches in alphabetical order |

**Difference between candidate groups and read batches:**
- A candidate group is a semantic classification, derived from filename pre-grouping
- A read batch is the unit of the image-reading operation, with at most 5 images per batch
- If a candidate group exceeds 5 images, split it into multiple read batches but keep the same candidate-group label

### Step 4 — Dispatch signature subagents

Before dispatching any subagents, output a progress message to the user listing all images about to be processed. Example format:

```
正在为以下 N 张图片生成结构签名，请稍候：
- image1.png
- image2.png
- image3.png
```

Dispatch one signature subagent for each read batch from Step 3. This applies even when there is only one batch.

For each batch:
1. Fill `{paths}` in `subagent-prompt.md` with the image paths for that batch (one path per line — they will be wrapped automatically by the template's `===paths-data-begin===` / `===paths-data-end===` fence).
   If the runtime requires absolute paths, also substitute the absolute path of `signature-spec.md` into the prompt before dispatching.
   If you need to inject a correction instruction (e.g., the Container integrity check re-dispatch wording), place it strictly between `===dispatcher-instructions-begin===` and `===dispatcher-instructions-end===`. Never inject correction text without the fence.
2. Dispatch a signature subagent with the filled prompt.
3. The subagent must read `signature-spec.md`, then read the image files, then return only signature text.
4. The subagent must not decide same/different component, define props, generate skeleton code, or describe image contents.

All batch subagents may run in parallel. The main agent only receives text signatures.

If subagent dispatch is unavailable in the current environment (e.g., no Agent tool, or the runtime refused dispatch), pause and present this recovery menu — do NOT silently call Read on image files from the main agent:

> Subagent dispatch is unavailable in this environment, so the main agent cannot read images while preserving the structure-only boundary. Please choose:
>
> A. Provide signatures manually — I will paste signatures (one per image) following `signature-spec.md`; the skill resumes at Step 5 validation with the provided signatures.
> B. Allow the main agent to read images this run — accepts the trade-off that the structure-only boundary will be relaxed for this invocation only; the main agent will read each image and apply the form-filling flow itself.
> C. Cancel the skill — exit cleanly with no output.

User picks A → wait for signatures, validate them against the Step 5 rules, then proceed to Step 6.
User picks B → record "boundary relaxed for this run" and proceed by reading images directly in the main agent; still follow signature-spec.md and the card-boundary rule.
User picks C → output a single line "image-to-component cancelled" and stop.

### Step 5 — Validate returned signatures

Validate every returned signature before any structural comparison. A signature is valid only if all checks pass:

- 5 slot lines `T` / `M` / `B` / `O` / `F` all exist.
- Slot order is exactly `T` → `M` → `B` → `O` → `F`, followed by a `notes` line.
- Every role word in slot expressions is from the 12-word vocabulary in `signature-spec.md`.
- Inside slot expressions, only the allowed operators appear: `:`, `->`, `+`, `()`, `-`, `?`.
- `status`, `hint`, `brand`, `empty`, `media`, `title`, and `meta` are not followed by `(`.
- Container parentheses only follow `list`, `card`, `form`, or `nav`.
- `->` and `+` have spaces on both sides.
- The right side of `+` does not contain a bare `->` sequence.
- `overlay` does not appear as a role word.
- **Bracket balance**: every `(` in a slot expression has a matching `)`; counting from left to right, the running open-count is never negative and ends at zero. Treat this as a hard syntax check — fail the signature on imbalance even if every other rule passes.
- **F slot cardinality**: when the `F` slot is not `-`, it must contain a single top-level role expression (one of `action`, `status`, `media`, or `nav`); the F slot must not contain `->` or top-level `+`. Multiple floating anchors require separate signatures, not a sequence inside F.
- **Notes key allowlist**: every key in `notes` must come from the enumerated set in `signature-spec.md` (`overlay_type`, `float_anchor`, `occluded`, `divider`, `tab_active`, `list_count`). Any other key — including visual keys like `bg`, `color`, `shadow`, `theme` — invalidates the signature.
- If the `O` slot is not `-`, `notes` includes `overlay_type = modal | drawer | toast | sheet`.
- If the `F` slot is not `-`, `notes` includes `float_anchor = br | bl | tr | tl`.
- **Container integrity check** (soft warning, not an automatic re-dispatch): If a slot's top-level sequence is `card(...) -> X -> …` AND the trailing leaves satisfy ALL of: (a) the leaves' roles are typical card-internal content (`title` / `meta` / `media` / `status`); (b) the trailing-leaf count is ≥ 2; (c) the slot has no other top-level container — then surface the suspicion to the user before re-dispatching, in this form:

    > Signature `<slot>: <expression>` shows trailing leaves after `card(...)`. They may belong inside the card. Please confirm: (1) trailing leaves are inside the card → re-dispatch with correction; (2) trailing leaves are independent page regions → keep signature as-is.

  Do NOT re-dispatch automatically. A single trailing `hint` / `action` after a card is a common legitimate pattern (footer text, sticky action) and must not fire this check.

If validation fails for a batch:
1. First failure: re-dispatch the same batch and include the concrete validation errors in the prompt.
2. Second failure: pause and show the bad signature plus validation errors, then ask:

```
Signature validation failed twice for this batch.

Bad signature:
<signature>

Validation errors:
<errors>

Please choose:
A. Provide corrected signatures for this batch manually
B. Skip this batch
C. Stop the workflow
```

User picks A → validate the user-provided signatures, then continue if valid
User picks B → exclude that batch from later comparison
User picks C → stop the workflow

Before proceeding to Step 6, output a natural-language summary of each image's structure — do NOT show the raw signature text to the user. In addition, output a JSX component tree derived mechanically from the signature beneath each image's summary, so the user can visually verify that the AI has correctly parsed the UI structure. Format:

```
已完成结构分析，共 N 张图：

**image1.png**：Header 区（title + meta）+ M 槽券信息卡（含 media 与 status overlay）+ B 槽时间戳

<ExpiredOrderPage>                  # Root component, expired state
  ├── <Header>                      # T slot
  │   ├── <HeaderTitle>             # T-level title leaf
  │   └── <HeaderMeta>              # T-level meta leaf
  │
  └── <Card>                        # M slot top-level container
      ├── <CouponInfo>              # M-internal card subcontainer
      │   ├── <IconThumb>           # media leaf inside CouponInfo
      │   └── <InfoText>            # title+meta vertical sequence inside CouponInfo
      │       ├── <CouponTitle>     # title leaf
      │       ├── <CouponMeta>      # meta leaf
      │       └── <CouponAmount>    # meta leaf (amount)
      ├── <DashedDivider>           # Decorative divider (from notes: divider=dashed)
      ├── <QRCodeArea>              # media leaf in M (state-dependent rendering)
      │   └── <ExpiredStamp>        # status leaf overlaid on media (rendered when status !== 'pending')
      └── <ExpiredTime>             # meta leaf (timestamp, rendered when status !== 'pending')

**image2.png**：<一句话描述>

<ComponentName>
  ├── ...
  └── ...
```

**JSX 组件树推导规则**（从签名机械推导）：

1. **组件命名**：仅根据 (a) 签名 role + 槽位、(b) 来自 Step 2 用户声明或 Step 6 比较结果的状态语境，使用 PascalCase 推导组件名。不得使用签名未携带的视觉信息（位置 / 颜色 / 图标形状 / 文案内容）。
   - `T` 槽容器 → 通用名 `<Header>`；若 Step 2 提供了页面用途（如"风险评估页"），可命名 `<RiskAssessmentHeader>`。
   - `M` 槽中的 `card(...)` → 通用名 `<Card>`；若用户声明提供了业务对象（如"券"），可命名 `<CouponCard>`。
   - `media` → 通用名 `<MediaA>`、`<MediaB>` 按 M 内出现顺序编号。**不允许**根据位置或视觉内容命名（如 `<IconThumb>` / `<QRCodeArea>`），除非 Step 2 用户声明明确指出了该 media 的语义。
   - `status` → 通用名 `<StatusStamp>`；若已知具体状态名（来自 Step 2 或 Step 6），可命名 `<UsedStamp>` / `<ExpiredStamp>`。
   - `hint` → 通用名 `<Hint>` 或 `<HintA>` / `<HintB>` 按出现顺序编号。**不允许**根据文案内容命名。
   - `action` → 通用名 `<ActionA>` / `<ActionB>` 按出现顺序编号；若 Step 2 用户声明给出了动作语义（如"刷新"），可命名 `<RefreshButton>`。
   - `form` → 通用名 `<FormFieldA>` / `<FormFieldB>` 按出现顺序编号。
   - `B` 槽 → 通用名 `<Footer>` 或直接展开为叶子节点。

2. **树结构符号**：
   - `├──` 表示非最后子节点
   - `└──` 表示最后子节点
   - `│` 保持缩进连线
   - **每一个节点（无论层级深浅）都必须带 `#` 注释**。注释内容仅限于：(a) 该节点对应的签名 role（title / meta / media / status / hint / action / form / nav / brand / empty）；(b) 所在槽位（T/M/B/O/F）和容器嵌套位置；(c) 状态语境（来自 Step 2 用户声明或 Step 6 比较结果，例如 "rendered when status === 'expired'"）；(d) 来自 `notes` 字段的非视觉属性（如 `divider=dashed`、`overlay_type=modal`）。**禁止描述颜色、字号、阴影、图标形状、具体文案内容**——这些信息签名不携带，主代理无法得知。

   **禁止的注释示例**（这些信息不在签名里，会触发幻觉）：
   - ❌ `# 橙色渐变背景` / `# 白色卡片，圆角阴影` — 颜色和阴影
   - ❌ `# 警告图标` / `# ¥ 图标` — 图标形状/字符
   - ❌ `# "订单已过期"` / `# "您的订单已过期！"` — 具体文案
   - ❌ `# 灰色小字，居中` — 字号、颜色、对齐

   **允许的注释示例**：
   - ✅ `# T slot, title leaf`
   - ✅ `# media leaf in M (rendered when status !== 'pending')`
   - ✅ `# status leaf overlaid on media`
   - ✅ `# decorative divider (from notes: divider=dashed)`

3. **`+` 水平并列**：同层兄弟节点，各自一行
4. **`->` 垂直序列**：父子关系，缩进一级
5. **Overlay**：在树的根节点之外单独列一个弹窗树，标注 `# overlay: modal/drawer/toast`
6. **多状态处理**：如果多张图是同一组件的不同状态，每张图单独输出一棵树，根节点名称体现状态（如 `<PendingOrderPage>`、`<UsedOrderPage>`、`<ExpiredOrderPage>`）

在每张图的自然语言摘要下方输出对应的 JSX 组件树。如果用户发现结构与设计稿不符，可在 Step 6 前指出，主代理重新派发签名子代理。

然后说明接下来要做结构比较（例如："接下来将对比这 N 张图的结构，判断是否属于同一组件的不同状态。"）。

原始签名文本只在主代理内部使用，不展示给用户。

### Step 6 — Collect signatures and make structural decisions

**Batching only limits image reading, not signature comparison.**

After all included batches have valid signatures:
1. Collect every valid signature text in one place.
2. Compare the complete signature set as a whole.
3. Decide same component vs different components, then proceed to Step 7.

The model does not need to remember visual information across batches; it only needs to process short text signatures together.

If Step 2 recorded an explicit relationship declaration, treat it as the **default** component-relationship decision and use the signatures to locate concrete differences for Step 7's prop modeling.

**Declaration vs signature conflict check** (mandatory before applying the declaration):

Run the mechanical rules in 6.1 against the signatures and compare the mechanical result with the user's declaration.

| Declaration | Mechanical result | Action |
|---|---|---|
| "same component, N states" | 6.1 also yields "same component" (state variant) | Apply the declaration; proceed to Step 7. |
| "same component, N states" | 6.1 yields "different component" (container role / topology differs, or role count differs by > 50%) | **Pause.** Show the user the signature diff and the conflict, then ask: A. Force-merge into a single component despite structural differences (model both branches via `status`). B. Override my declaration — treat as different components. C. Restart Step 4 with corrected images (one of the images was wrong). Do NOT silently merge. |
| "different components" | 6.1 yields "same component" | Apply the declaration (different); user's semantic intent trumps coincidental structural match — equivalent to 6.2 option B/C. |
| Sequenced flow declaration ("step1, step2, step3 of one wizard") | 6.1 yields "same component" or "manual review" | Apply the declaration; treat as 6.2 option C (single component with a `step`/`phase` discriminator). |

The conflict-check prompt format when "same component" declaration conflicts with mechanical evidence:

> Your declaration: <declaration text>
> Mechanical signature analysis says these are different components because: <specific reason — container role / topology / role-count diff>.
>
> Please choose:
> A. Force-merge — keep one component, model differences via `status` (use only if the differences are truly semantic state, not structural).
> B. Override — accept the mechanical result and treat as different components.
> C. One of the images was wrong — restart Step 4 with a corrected image set.

#### 6.1 Mechanical decision rules

**Structural skeleton (容器拓扑)** = drop all leaf-node roles, keep only container structure and topology. Distinct from the **code skeleton** generated in Step 8 (multi-file component scaffold).
Example: structural skeleton of `title -> list(card(title -> meta))` = `_ -> list(card(_ -> _))`

**Same component vs different component** (first strip the O line, only compare T/M/B/F):

| Condition | Decision |
|---|---|
| The structural skeletons of all four T/M/B/F lines are identical, or differ only by allowed leaf-node additions/removals below | Candidate "same component", proceed to state-variant decision |
| Any container role or container topology differs | **Different component** |
| Total role count differs by > 50% | **Different component** (guards against degenerate structural-skeleton false matches) |

**State variant vs structural variant** (given no container-role or container-topology differences):

| Difference type | Classification |
|---|---|
| Leaf-node role swap (`hint` ↔ `status`) | State variant |
| Leaf-node `?` appears | State variant (uncertain) |
| Leaf node added/removed, total count change ≤ 1 | State variant |
| An entire slot's leaf-only content is fully replaced (container topology unchanged; all leaves in that single slot differ) | State variant (whole-slot replacement, model via `status` per Step 7) |
| Leaf nodes added/removed in ≥ 2 distinct slots | Structural variant (manual review) |
| Repetition count inside a container changes | State variant (data-driven) |

**Overlay rules**:
- During comparison, strip the O line first; the base layer (T/M/B/F) is judged independently
- O lines are aggregated separately into an overlay candidate set, then compared internally by the same rules
- Different `overlay_type` → directly judged as different overlay components

**F slot rule**: F slot appearing/disappearing counts as a state variant and does not affect component-identity decisions.

---

#### 6.2 Manual review exit

When Step 6's decision is "structural variant (manual review)", run the following flow:

**First: output the signature diff**

Show the slot signatures that differ and annotate the number of differences; do not just say "I'm not sure":

```
Structural skeletons are identical, but there are 2+ leaf-node differences; cannot decide mechanically.

Signature comparison:
Image 1  M: <image 1 M slot signature>
         B: <image 1 B slot signature>

Image 2  M: <image 2 M slot signature>
         B: <image 2 B slot signature>

Difference locations: <list specific locations and counts>

Please choose:
A. Different states of the same component (express differences via props)
B. Different components, structure is coincidentally similar (generate independent code skeletons)
C. Sequential steps of one flow that happen to share structure (generate a single component with a step/phase prop)
```

**Then: wait for user to pick A, B, or C**

Identical structural skeletons do not imply same component. Two semantically independent pages can share the same skeleton (e.g. several settings pages with `T: nav, M: list(form), B: action`). And sequenced flow steps (wizard step 1, step 2, …) may also share structure while differing in content. Only the user can resolve this ambiguity from the business context.

**Finally: continue based on the choice**

| User choice | Follow-up action |
|---|---|
| A (same component) | Record "user confirmed: same component", proceed to Step 7, model differing leaf nodes per the mapping rules |
| B (different components, coincidentally similar structure) | Record "user confirmed: different components, coincidentally similar structure", run Steps 7 and 8 separately for each image, generate independent code skeletons |
| C (sequential flow steps) | Record "user confirmed: sequential flow steps", proceed to Step 7 with a `step: 'step1' \| 'step2' \| …` (or `phase`) discriminator prop; treat per-step content differences as state variants under that discriminator |

#### 6.3 Large-directory candidate-group conflicts

Only evaluate candidate-group conflicts after all signatures have been collected.

**Mechanical definition of a conflict:**
During signature comparison, two or more candidate groups have different structural skeletons, and each candidate group contains more than 1 image.

**A single isolated image does not count as a conflict:**
By default, a single image is matched into the candidate group whose structural skeleton is closest to it; if no group matches, treat it as an independent component candidate.

**On conflict, pause and ask the user:**

```
Merging found multiple candidate groups with different structural skeletons; cannot auto-merge.

Candidate group 1 (N images): <filename list>  Structural skeleton: <skeleton signature>
Candidate group 2 (N images): <filename list>  Structural skeleton: <skeleton signature>

Please choose:
A. Split by component, generate independent code skeletons for each
B. Treat as a state set of the same component, force merge
C. Process only a specified subset of files. List filenames directly,
   e.g.: pending.png, used.png, expired.png
   The model will restart from Step 3 and only process these files.
```

User picks A → run Steps 7 and 8 separately for each candidate group
User picks B → merge all signatures, proceed to Step 7, model all differences as props
User picks C → restart from Step 3, only processing the files the user listed

---

#### 6.4 Golden Examples

Read `examples/golden-cases.md` when triggering Manual Review Exit (6.2), or when comparing 4+ signatures with mixed leaf-node replacement/addition/removal patterns.

### Step 7 — Define props

**Signature diff → prop mapping rules**

After obtaining the Step 6 diff result, map mechanically per the table below; do not improvise:

| Diff type | Modeling approach |
|---|---|
| `status` leaf node appears/disappears | `status: StatusUnion` drives conditional rendering; do not split into a separate prop |
| `meta` leaf-node content varies with status | A concrete data prop, e.g. `timestamp?: string` |
| `action` leaf node disappears with status | The corresponding callback becomes optional, e.g. `onRefresh?: () => void` |
| An entire slot is fully replaced | `status` drives conditional rendering; do not split into a separate prop |
| `hint` leaf node disappears with status | Static copy, hardcoded inside the component, not a prop |
| `media` leaf-node content varies with status | A concrete asset prop, e.g. `qrCodeUrl: string` |

For a single component with status variants, always use:

```ts
// ✅ Correct: flat status union
type OrderStatus = 'pending' | 'used' | 'expired'

interface ComponentProps {
  status: OrderStatus
  // ... shared data fields
  timestamp?: string   // shown only in some states
  onRefresh?: () => void  // shown only in some states
}
```

```ts
// ❌ Avoid: over-split props that force callers to pre-compute state
interface BadProps {
  bannerData: BannerToBeUsed | BannerUsed | BannerExpired  // caller shouldn't need this
  voucher: VoucherInfo
}
```

Status naming convention: `pending` (not `to-be-used`), `used`, `expired`, `active`, `inactive`.

### Step 8 — Generate code skeleton

Generate code skeleton files that match the Step 9 directory tree. Do not output one monolithic component when the directory tree contains subcomponents.

#### Split rules

- The root component owns `status`, shared data props, and composition.
- Split by semantic region from the signature: `T` → header/status hero, `M` → main content/card/media regions, `B` → footer/action area, `O` → overlay component.
- Split a region into its own component when ANY of the following holds:
  - It is **status-varying** (renders differently across `status` union members).
  - It is **repeated** (rendered inside a `list(...)` container — the list item is always its own component).
  - It is **resource-heavy** (loads media, runs animations, owns its own data fetching).
  - It is **structurally non-trivial**: the signature shows ≥ 2 levels of nesting under that region (e.g., `card(media + card(title -> meta))` qualifies because the inner card adds a level), OR it contains ≥ 4 leaf nodes.
  - It is a **distinct semantic region** named by a slot or container with its own role identity (a card holding a coupon, an overlay modal, a tab bar).
- Static business-object regions such as coupon/order/item info should still be separate components when they are visually distinct.
- Pass only the props a child needs. Do not pass the entire parent props object.

#### Class composition

- For React output, use the project's existing `cn`, `clsx`, or `classnames` helper if present.
- For React output, if no helper exists, add a shared `cn` helper at `utils/cn.ts` (TypeScript) or `utils/cn.js` (JavaScript), and import it wherever needed.
- For Vue output, use Vue's native array/object class bindings. Do not add a `cn` helper unless the target project already uses one.
- Do not hand-build long conditional class strings with template-string concatenation.

#### Template reference

Before generating component files, read exactly one template based on the Step 1 **framework × language × style-stack** answers:

| Framework | Language | Style stack | Template |
|---|---|---|---|
| React | TypeScript | CSS Modules | `templates/react-tsx-css-modules.md` |
| React | TypeScript | plain CSS + BEM | `templates/react-tsx-bem.md` |
| React | JavaScript | CSS Modules | `templates/react-jsx-css-modules.md` |
| React | JavaScript | plain CSS + BEM | `templates/react-jsx-bem.md` |
| Vue 3 | TypeScript or JavaScript | CSS Modules | `templates/vue3-sfc-css-modules.md` |
| Vue 3 | TypeScript or JavaScript | plain CSS + BEM | `templates/vue3-sfc-bem.md` |
| Vue 2 | TypeScript or JavaScript | CSS Modules | `templates/vue2-sfc-css-modules.md` |
| Vue 2 | TypeScript or JavaScript | plain CSS + BEM | `templates/vue2-sfc-bem.md` |

The selected template contains the multi-file skeleton (React TSX/JSX or Vue SFC), styling skeleton, class composition pattern, shared types (TS) or JSDoc typedefs (JS), accessibility examples, resource rules, and export conventions.

Never mix TS and JS in the same generation: do not emit `interface` / `type` / generic syntax in JS output, and do not strip TS annotations from a TS-selected template.

If the user selected "其他框架"（degraded mode），仅输出 Step 9 的目录树和组件树（Step 5→6 风格的组件树），明确告知用户当前 skill 不会生成可运行代码，建议手工迁移结构。不要尝试用 React 或 Vue 模板伪造其他框架代码。

### Step 9 — Output

**Regardless of whether the user asked for chat output or file writing, always output a directory tree first**, then output or write the actual files.

Directory tree format (adapt filenames to the selected Step 8 template):

```
src/components/
└── QRCodeOrderPage/
    ├── index.ts               # 命名导出 root component 和公共类型
    ├── QRCodeOrderPage.tsx    # 主组件，接收 status prop 并组合子组件
    ├── QRCodeOrderPage.css    # BEM 样式骨架，与 TSX className 对齐
    ├── types.ts               # 公共类型，避免子组件反向依赖父组件 props
    ├── utils/
    │   └── cn.ts              # className 组合 helper（无项目级 helper 时）
    └── components/
        ├── Header.tsx         # 图标+标题+副标题（按状态变化）
        ├── CouponInfo.tsx     # 券名称、有效期、金额（静态）
        ├── QRCodeArea.tsx     # 二维码展示（active/grayed + stamp）
        └── Footer.tsx         # 底部提示/时间（按状态变化）
```

For the CSS Modules variant (default), the same tree becomes:

```
src/components/
└── QRCodeOrderPage/
    ├── index.ts
    ├── QRCodeOrderPage.tsx
    ├── QRCodeOrderPage.module.css
    ├── types.ts
    ├── utils/
    │   └── cn.ts
    └── components/
        ├── Header.tsx
        ├── Header.module.css
        ├── CouponInfo.tsx
        ├── CouponInfo.module.css
        ├── QRCodeArea.tsx
        ├── QRCodeArea.module.css
        ├── Footer.tsx
        └── Footer.module.css
```

Rules for the directory tree:
- Add a `#` comment after each file with one sentence describing its responsibility.
- Specifically note which files change by `status` and which are static.
- The files shown in Step 8 must match this tree. Do not print a single-file code skeleton when this tree contains subcomponents.
- For React CSS Modules, list `QRCodeOrderPage.module.css` and each child `components/<Child>.module.css` in the tree. Do not collapse all child styles into the root module.
- For Vue CSS Modules, use `<style module>` inside each `.vue` SFC by default. Do not list separate `.module.css` files unless the target project already uses external Vue CSS Modules.
- Apply the selected style stack to filenames:

| Style stack | Directory tree adjustment |
|---|---|
| CSS Modules | React: use `QRCodeOrderPage.module.css` plus `components/<Child>.module.css`; Vue: use `<style module>` per SFC and `$style` bindings |
| plain CSS + BEM | React: use `QRCodeOrderPage.css`; Vue: import `QRCodeOrderPage.css` from the root SFC; both use BEM class names |

- Apply the selected framework and language to filenames and the type-sharing file:

| Framework / Language | Directory tree adjustment |
|---|---|
| React + TypeScript | Use `.tsx` for components, `.ts` for utils/index, and include `types.ts` for shared `OrderStatus` / `CouponInfoData`. The barrel re-exports the component **and** public types. |
| React + JavaScript | Use `.jsx` for components, `.js` for utils/index. **Replace `types.ts` with `types.js`** — declare shared JSDoc `@typedef` blocks (status unions, data shapes) inside `types.js`. Reference them via `import('./types').XYZ` from the root component (same directory as `types.js`) and via `import('../types').XYZ` from subcomponents under `components/` (one level deeper). The barrel re-exports the component only. Do not declare shared typedefs inside the root `.jsx` — that creates a reverse dependency from children to the parent file. |
| Vue 3 + TypeScript | Use `.vue` SFCs with `<script setup lang="ts">`, `index.ts`, and `types.ts`. |
| Vue 3 + JavaScript | Use `.vue` SFCs with `<script setup>`, `index.js`, and root-level JSDoc typedefs. Do not create `types.ts`. |
| Vue 2 + TypeScript | Use `.vue` SFCs with Options API, `<script lang="ts">`, `Vue.extend`, `PropType`, `index.ts`, and `types.ts`. |
| Vue 2 + JavaScript | Use `.vue` SFCs with Options API, plain `<script>`, `index.js`, and root-level JSDoc typedefs. Do not create `types.ts`. |

The example tree below uses the **React + TypeScript + plain CSS + BEM** variant. If the user selected CSS Modules (the default recommendation), every `.css` file becomes `.module.css` per the style-stack table above. For Vue or JavaScript output, adapt filenames according to the framework / language tables above and the selected Step 8 template.

After outputting the directory tree:

**If user asked for chat output:** Print the code skeleton(s) directly in the response. Do NOT create files.

Before writing any file, check if a file already exists at the target path. If yes, do NOT overwrite silently. List the conflicts to the user and ask:

A. Overwrite all
B. Skip existing files (only create missing ones)
C. Cancel write, output to chat instead

Wait for explicit user choice before proceeding.

**If user asked to write files:** Create files, then confirm paths. Suggest directory: `src/components/ComponentName/`.

**If user did not specify:** Default to **chat output only**. Do NOT create files without being asked.

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Skip comparison, jump to "N separate components" | Always compare structure first — shared layout = shared component |
| Create files without being asked | Default to chat output; ask before writing to disk |
| Use `'to-be-used'` or similar hyphenated status | Use `'pending'` — idiomatic TS union member naming |
| Split props into status-specific sub-objects | Keep flat: one `status` string + optional shared fields |
| Assume framework from project files | Ask upfront if not told |
| Assume styling strategy | Ask the Step 1 style-stack question and use the matching Step 8 template |
| Main agent reads image files directly | Dispatch signature subagents for every batch; the main agent only handles filenames and returned signature text |
| Trust user's description of how many images exist | Always `ls` the directory first — never hallucinate the file list |
| Replace icon placeholder with status text | Keep the icon placeholder and use SVG/icon library/assets according to the screenshot |
| Subcomponent imports parent component's props type | Define shared types in `types.ts`; never reverse-depend on parent props |
| Re-define `cn` helper in every component file | Extract `cn` to `utils/cn.ts` and import it everywhere |
| Hardcode `<h1>` in reusable component | Accept `titleAs?: 'h1' | 'h2' | 'h3'` or equivalent heading prop |
| Mix TS syntax in JS template (or vice versa) | Match the Step 1 language answer; never include `interface` / `type` / generic syntax in JS output, and never strip TS annotations from a TS-selected template |
