# Golden Signature Comparison Cases

## Golden Examples

**Case A: QR-code order page (3 images = 3 states of the same component)**

```
# A1 pending（待使用）
T: title -> meta
M: card(media + card(title -> meta -> meta) -> media)
B: hint -> action + hint
O: -
F: -
notes: { divider=dashed }

# A2 used（已核销）
T: title -> meta
M: card(media + card(title -> meta -> meta) -> media -> status)
B: meta
O: -
F: -
notes: { divider=dashed }

# A3 expired（已过期）
T: title -> meta
M: card(media + card(title -> meta -> meta) -> media -> status)
B: meta
O: -
F: -
notes: { divider=dashed }
```

Decision: T/O/F are identical; M skeleton A1=`card(_+card(_->_->_)->_)`, A2/A3=`card(_+card(_->_->_)->_->_)`, the difference is one leaf node `status` added at the end (≤1) → state variant; B slot has a leaf-node swap → state variant. **Conclusion: same component, 3 states.**

---

**Case B: List page vs detail page (different components)**

```
# B1 List page
T: nav
M: list(card(title -> meta + status))
B: nav
O: -
F: -
notes: {}

# B2 Detail page
T: nav
M: title -> meta -> media -> status -> form
B: action + action
O: -
F: -
notes: {}
```

Decision: At the top of M slot, B1=`list(card(...))`, B2=`_ -> ...`, skeletons differ → **different components**.

---

**Case C: Detail page + confirm dialog (Overlay)**

```
# C1 Normal state
T: nav
M: title -> meta -> media -> form
B: action + action
O: -
F: -
notes: {}

# C2 Confirm Modal popped
T: nav
M: title -> meta -> media -> form
B: action + action
O: card(title -> meta -> action + action)
F: -
notes: { overlay_type=modal }
```

Decision: After stripping the O line, T/M/B/F are identical → same base component; the O line stands alone → produces an independent overlay component `ConfirmModal`.

---

**Case D: Login form with 2 states (idle vs error)**

```
# D1 idle
T: title -> meta
M: form(form -> form -> action)
B: hint
O: -
F: -
notes: {}

# D2 error
T: title -> meta
M: form(form -> form -> hint -> action)
B: hint
O: -
F: -
notes: {}
```

Decision: T/B/O/F identical; M skeleton D1=`form(_ -> _ -> _)`, D2=`form(_ -> _ -> _ -> _)`，差异为 M 内 form 容器中追加一个 `hint` 叶子节点（≤1）→ state variant。**Conclusion: same component, 2 states (idle / error)**

---

**Case E: Empty state vs filled list (different leaf, same skeleton)**

```
# E1 empty
T: nav
M: empty
B: action
O: -
F: -
notes: {}

# E2 filled
T: nav
M: list(card(title -> meta))
B: action
O: -
F: -
notes: {}
```

Decision: M 容器拓扑差异：E1 为单叶子 `empty`，E2 为 `list(card(...))`，容器角色和拓扑都变化 → **different components**（empty state 与列表视图建议拆为独立组件，避免在同一组件内承担两种结构）。

---
