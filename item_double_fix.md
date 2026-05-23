# Item Double Bug Fix — applyCompactLayout

## The Bug

When a player buys an item that fills a **gap in the equip API array**, `buildVisualLayout` repacks the visual slots, causing an existing item to shift positions. This triggers a **slide animation** for the existing item starting from its old visual slot. At the same time, the new item appears instantly in that same old slot with no animation — causing both items to visually occupy the same slot simultaneously for ~350ms.

**Example:**
- Equip state before: `[A, 9999, B]` → visual: `[A, B, 9999]`
- Equip state after:  `[A, C, B]`   → visual: `[A, C, B]`
- B slides from visual slot 1 → 2
- C appears at visual slot 1 instantly
- **Result:** B and C both appear at slot 1 during the slide = double item

---

## The Fix

Two changes inside `applyCompactLayout`:

### 1. Track which slots are being vacated by a slide

Add this block **before** the animation loop:

```js
const slidingFrom = new Set();
newLayout.forEach((newId, j) => {
  if (newId === '9999') return;
  const oldJ = oldIds.indexOf(newId);
  if (oldJ !== -1 && oldJ !== j) slidingFrom.add(oldJ);
});
```

This records every old visual slot that an item is sliding away from.

---

### 2. Fix the pop condition and add delay for overlap cases

**Find this block** in the animation loop (inside the `newLayout.forEach`):

```js
} else if (oldIds[j] === '9999') {
  slotEl.classList.remove('item-pop', 'item-combine');
  void slotEl.offsetWidth;
  slotEl.classList.add('item-pop');
}
```

**Replace with:**

```js
} else if (oldJ === -1) {
  // delay pop if a sliding item is vacating this slot to avoid visual overlap
  const delay = slidingFrom.has(j) ? 200 : 0;
  setTimeout(() => {
    slotEl.classList.remove('item-pop', 'item-combine');
    void slotEl.offsetWidth;
    slotEl.classList.add('item-pop');
  }, delay);
}
```

**Why the condition changed:**
- Old: `oldIds[j] === '9999'` — only popped if the slot itself was empty before
- New: `oldJ === -1` — pops any item that wasn't in the old layout at all, regardless of what was previously in that slot

**Why the delay:**
- If a sliding item is leaving slot `j`, delaying the new item's pop by 200ms gives the slide animation time to visually clear the slot before the new item appears

---

## Full applyCompactLayout reference (after fix)

```js
function applyCompactLayout(p, newLayout) {
  const oldIds = p.itemSlots.map(s => s.querySelector('img').dataset.itemId || '9999');
  if (oldIds.join(',') === newLayout.join(',')) return;

  const oldRects = p.itemSlots.map(s => s.getBoundingClientRect());

  newLayout.forEach((newId, j) => {
    const imgEl = p.itemSlots[j].querySelector('img');
    if (newId === (imgEl.dataset.itemId || '9999')) return;
    imgEl.dataset.itemId = newId;
    imgEl.src = `items/${newId === '9999' ? '99999' : newId}.png`;
    imgEl.onerror = () => { imgEl.src = 'items/99999.png'; };
  });

  // track which old visual slots are being vacated by a slide
  const slidingFrom = new Set();
  newLayout.forEach((newId, j) => {
    if (newId === '9999') return;
    const oldJ = oldIds.indexOf(newId);
    if (oldJ !== -1 && oldJ !== j) slidingFrom.add(oldJ);
  });

  newLayout.forEach((newId, j) => {
    if (newId === '9999') {
      const imgEl = p.itemSlots[j].querySelector('img');
      imgEl.style.transition = '';
      imgEl.style.transform  = '';
      return;
    }
    const oldJ   = oldIds.indexOf(newId);
    const imgEl  = p.itemSlots[j].querySelector('img');
    const slotEl = p.itemSlots[j];
    if (oldJ !== -1 && oldJ !== j) {
      const dx = oldRects[oldJ].left - oldRects[j].left;
      imgEl.style.transition = 'none';
      imgEl.style.transform  = `translateX(${dx}px)`;
      requestAnimationFrame(() => requestAnimationFrame(() => {
        imgEl.style.transition = 'transform 0.35s cubic-bezier(0.22,1,0.36,1)';
        imgEl.style.transform  = 'translateX(0)';
        setTimeout(() => { imgEl.style.transition = ''; imgEl.style.transform = ''; }, 400);
      }));
    } else if (oldJ === -1) {
      const delay = slidingFrom.has(j) ? 200 : 0;
      setTimeout(() => {
        slotEl.classList.remove('item-pop', 'item-combine');
        void slotEl.offsetWidth;
        slotEl.classList.add('item-pop');
      }, delay);
    }
  });
}
```
