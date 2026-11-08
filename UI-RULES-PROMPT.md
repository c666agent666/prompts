# UI implementation rules (apply to every screen, panel and dialog)

You are implementing/adjusting UI. These rules are mandatory. Apply them to all
controls, not just the ones you are currently editing. Do not change application
behaviour while doing this.

## 1. Sizing: every control fits its content

- NEVER stretch buttons, inputs, selects, number fields, colour pickers or sliders
  to fill the panel (`width: 100%` on these is forbidden).
- Buttons size to their label: `width: auto; padding: 6px 12px` (primary/secondary
  alike). A button must never be wider than its text + padding.
- Number inputs: width fits the largest valid value plus steppers (~7ch for up to
  4 digits, `8ch` for 5). Use `ch` units, not percentages.
- Text inputs: width chosen for their purpose (e.g. url/key ~280px, prompt/body
  ~420px max), never full width.
- Colour inputs: small swatch, about 46x26px, `padding: 1px`.
- Selects: `width: auto; max-width: 100%`.
- Sliders: fixed short length (~150px), not full width.
- Rows of fields: `display:flex; flex-wrap:wrap; gap:10-12px`; each field keeps
  its intrinsic width and wraps instead of stretching.
- Panels/drawers: compact padding (10-14px), bounded width (e.g. 480px), no
  horizontal scrolling (`overflow-x: hidden`); long text columns (chat, editor)
  are the only things that may span the panel.
- Group settings into tabs: separate functional settings (audio/device/model/etc.)
  from Appearance (theme, colours, typography, glow). Appearance must be its own
  tab, never mixed with device/behaviour settings.
- Labels: small uppercase monospace; live value shown next to the label
  (e.g. `Max input time  12 s`). Section headings small and compact.

## 2. Sliders: large thumb + deadzone

Every range input must:
- Use a custom 16-18px round thumb, 6px track, `cursor: grab` / `grabbing`,
  `touch-action: none`.
- Have a deadzone: clicking on the track must NOT jump the value. Only grabbing
  the thumb (pointer within ~14px of it) or keyboard changes the value.
- Keep keyboard support: ArrowLeft/Right, PageUp/Down, Home/End.
- Update its value label instantly.

Reference implementation (vanilla, adapt to the framework):

```css
input[type=range] {
  -webkit-appearance: none; appearance: none;
  width: 150px; height: 20px; background: transparent; touch-action: none;
  cursor: pointer;
}
input[type=range]::-webkit-slider-runnable-track {
  height: 6px; border-radius: 3px; background: #111; border: 1px solid #333;
}
input[type=range]::-webkit-slider-thumb {
  -webkit-appearance: none; appearance: none;
  width: 18px; height: 18px; margin-top: -7px; border-radius: 50%;
  background: var(--accent); border: 2px solid var(--bg);
  box-shadow: 0 0 0 1px var(--border); cursor: grab;
}
input[type=range]:active::-webkit-slider-thumb { cursor: grabbing; }
```

```js
function installSliderDeadzone(el) {
  el.addEventListener('pointerdown', (event) => {
    const rect = el.getBoundingClientRect();
    if (!rect.width) return; // hidden panel / no layout (tests)
    const min = Number(el.min || 0), max = Number(el.max || 100);
    const frac = max > min ? (Number(el.value) - min) / (max - min) : 0;
    const thumbX = rect.left + frac * rect.width;
    if (Math.abs(event.clientX - thumbX) > 14) event.preventDefault();
  }, { capture: true });
}
document.querySelectorAll('input[type=range]').forEach(installSliderDeadzone);
```

## 3. Feedback and accessibility

- Show current value next to each control; show a live timer/counter where a
  duration is being recorded.
- Active/selected options get a visible state (border + text colour), not just a
  different background.
- Keep controls keyboard reachable and give icon-only buttons `title`/`aria-label`.

## 4. Verification (do this before saying it is done)

For every screen you touched, check and report pass/fail:
1. Click at 10% and 90% of each slider track: the value must not change.
2. Drag each thumb the full range: value changes smoothly; label updates.
3. Arrow keys move each focused slider.
4. Number boxes: type the maximum allowed value; nothing clips.
5. Buttons: measure width = label + padding, not container width.
6. No input, select or button is full-width in panels.
7. Resize the window/panel: rows wrap, no horizontal scrollbar appears.
8. Settings are split so Appearance is separate from functional settings.
9. Nothing in the app's behaviour changed unless explicitly requested.

## 5. Forbidden

- Full-width buttons/inputs/selects in panels or settings.
- Oversized boxes ("just stretch it to the grid").
- Tiny 10px slider thumbs.
- Slider values jumping on track click.
- Mixing colour/theme controls with device/behaviour controls in one long panel.
- Declaring done without running the checks above.
