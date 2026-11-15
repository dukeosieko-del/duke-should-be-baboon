# 13. Walkthrough engine

**Ground truth:** `src/lib/walkthrough/**`, `src/components/walkthrough/**`,
`src/components/OrderForm.tsx` (`data-walkthrough` attrs).

## Components
- `WalkthroughRenderer` — renders the active step overlay.
- `Spotlight` — highlights the target element.
- `StepPopover` — the step content card.
- `engine/index.ts` — the core engine.
- `engine/target-resolver.ts` — resolves `data-walkthrough` selectors to DOM nodes.
- `engine/useWalkthrough.tsx` — React hook.
- `hooks/useWalkthroughTarget.ts` — per-target hook.
- `state/events.ts` — event bus.
- `state/storage.ts` — persisted completion state.
- `types.ts` — types.

## Journeys (`src/lib/walkthrough/journeys/`)
| Journey | Purpose |
|---|---|
| `orientation` | first-visit tour |
| `customer` | registered-user onboarding |
| `guest-order` | guest checkout walkthrough |
| `blog` | blog discovery |
| `notifications` | notification centre tour |
| `all` | aggregate |

## Integration
Passive — `data-walkthrough` attributes are added to key components
(`MpesaModal.tsx:186` `walkthrough-mpesa-payment`, `:293` `walkthrough-mpesa-pay`).
The engine reads them; it never modifies the component logic.

## Rationale
- **Walkthrough is non-modifying.** It is a read-only overlay. It was added as
  passive `data-walkthrough` attributes (commit `b443025`) precisely so it could
  not break the components it annotates.
- **Completion is persisted** so returning users skip the tour.