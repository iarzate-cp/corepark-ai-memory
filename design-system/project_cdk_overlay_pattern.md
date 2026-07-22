---
name: CDK Overlay Animated Panel Pattern — backoffice
description: How to wire up a CDK overlay panel with enter/exit animations and animated backdrop in frontend-backoffice
type: project
originSessionId: 68fbc8bf-6233-4778-ae34-196a53bbfd68
---
## Pattern: animated CDK overlay with exit animation

Used in `full-date-filter` (date range picker overlay). The challenge: Angular's `animations: []` `:leave` trigger doesn't fire when `OverlayRef.dispose()` is called (CDK bypasses the animation engine). Solution: manual CSS class + setTimeout.

### Service (`date-filter-overlay.service.ts`)

```ts
// Store ComponentRef to access instance for startLeave()
#componentRef: ComponentRef<FullDateFilterComponent> | null = null

this.#ref = this.#overlay.create({
  positionStrategy,
  hasBackdrop:    true,
  backdropClass:  ['cdk-overlay-dark-backdrop', 'fdf-backdrop'],
  scrollStrategy: this.#overlay.scrollStrategies.reposition(),
})

this.#componentRef = this.#ref.attach(
  new ComponentPortal(FullDateFilterComponent, null, injector)
) as ComponentRef<FullDateFilterComponent>

#dispose(): void {
  // ...null guards...
  ref.backdropElement?.classList.add('is-leaving')   // triggers backdrop exit anim
  compRef?.instance.startLeave()                      // triggers panel exit anim
  setTimeout(() => ref.dispose(), 160)                // wait for animation to finish
}
```

### Component host class

```ts
@Component({ host: { '[class.is-leaving]': 'isLeaving()' }, styles: [`
  :host {
    display: block;
    animation: fdf-panel-in 160ms cubic-bezier(0.16, 1, 0.3, 1) both;
    &.is-leaving { animation: fdf-panel-out 160ms cubic-bezier(0.4, 0, 1, 1) forwards; }
  }
  @keyframes fdf-panel-in  { from { opacity: 0; transform: translateY(-6px) scale(0.97); } to { opacity: 1; transform: translateY(0) scale(1); } }
  @keyframes fdf-panel-out { from { opacity: 1; transform: translateY(0) scale(1); } to { opacity: 0; transform: translateY(-6px) scale(0.97); } }
`] })
export class FullDateFilterComponent {
  readonly isLeaving = signal(false)
  startLeave(): void { this.isLeaving.set(true) }  // called by service
}
```

### Global backdrop animation (`src/assets/scss/_cdk-overlay.scss`)

```scss
.fdf-backdrop             { animation: fdf-backdrop-in  160ms ease-out forwards; }
.fdf-backdrop.is-leaving  { animation: fdf-backdrop-out 160ms ease-in  forwards; }
@keyframes fdf-backdrop-in  { from { opacity: 0; } to { opacity: 1; } }
@keyframes fdf-backdrop-out { from { opacity: 1; } to { opacity: 0; } }
```

Added to `src/styles.scss` via `@use 'cdk-overlay'`.

### Notes
- `backdropClick()` and Escape key both go through `#dispose()` → animated close
- Apply button calls `this.#overlayRef.dispose()` directly from the component → immediate close (no exit animation) — intentional UX
- `cdk-overlay-dark-backdrop` provides the semi-transparent background; `fdf-backdrop` provides the animation hook
