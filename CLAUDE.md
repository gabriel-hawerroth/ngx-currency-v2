# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository layout

This is an Angular workspace containing two projects (see [angular.json](angular.json)):

- `ngx-currency` — the published library (`projects/ngx-currency/`). This is the actual product.
- `ngx-currency-demo` — a demo application (`src/`) used for manual testing and the GitHub Pages demo. Not published.

The library's public API is defined in [projects/ngx-currency/src/public-api.ts](projects/ngx-currency/src/public-api.ts) — only `NgxCurrencyDirective`, the config types, and `provideEnvironmentNgxCurrency` are exported.

Node version is pinned to `18.19.1` (`.nvmrc`).

## Commands

```sh
npm start              # serve the demo app (ng serve) at localhost:4200
npm test               # run the LIBRARY unit tests (ng test ngx-currency, Karma + Jasmine)
npm run lint           # ESLint over both projects (angular-eslint)
npm run build:lib      # build the library into dist/ngx-currency (+ copies README/LICENSE)
npm run build          # build the demo app
npm run watch          # rebuild the demo continuously (development config)
npm run release        # commit-and-tag-version (bumps version, updates CHANGELOG)
```

Run a single test by adding `fdescribe`/`fit` in the spec, or:
```sh
ng test ngx-currency --include='**/input.service.spec.ts'
```

Commits must follow Angular Conventional Commits — enforced by commitlint via a Husky `commit-msg` hook (`type: description`, e.g. `feat: ...`, `fix: ...`, `build: ...`).

## Architecture

The library is a single Angular directive backed by three plain (non-Angular) helper classes forming a layered pipeline. Understanding this layering is key to making changes safely:

- **[NgxCurrencyDirective](projects/ngx-currency/src/lib/ngx-currency.directive.ts)** — the only Angular class. Implements `ControlValueAccessor` so the input integrates with Angular forms (`formControlName`/`ngModel`). It merges options from three sources in precedence order: hardcoded defaults → global config injected via `NGX_CURRENCY_CONFIG` → per-input `[currencyMask]` options. It uses a `KeyValueDiffer` in `ngDoCheck` to react to option changes at runtime. All DOM events are captured via `@HostListener` and delegated to the InputHandler.

- **[InputHandler](projects/ngx-currency/src/lib/input.handler.ts)** — translates keyboard/clipboard events (keydown, keypress, input, cut, paste) into semantic operations (add number, remove number, change sign) and invokes the `onModelChange` callback to push values back to the Angular form.

- **[InputService](projects/ngx-currency/src/lib/input.service.ts)** — the core masking logic. `applyMask` converts a numeric/raw string into the formatted display string (prefix/suffix/thousands/decimal, min/max clamping, negative handling); `clearMask` reverses it back to a `number | null`. Also handles cursor positioning logic and Eastern-Arabic (`٠`–`٩`) / Persian (`۰`–`۹`) numeral normalization.

- **[InputManager](projects/ngx-currency/src/lib/input.manager.ts)** — thin wrapper over the raw `HTMLInputElement`: reads/writes `.value`, manages cursor/selection ranges, tracks the previously stored raw value, and enforces `maxLength`.

### Two input modes

Behavior is governed by `NgxCurrencyInputMode` (`Financial` vs `Natural`) defined in [ngx-currency.config.ts](projects/ngx-currency/src/lib/ngx-currency.config.ts). Financial mode fills from the smallest decimal place upward (cash-register style); Natural mode types left-to-right around a fixed decimal point. Much of the branching in `InputService.addNumber` / `removeNumber` exists specifically to differentiate these two modes — check both when editing typing/deletion logic.

### Chrome Android special-casing

`NgxCurrencyDirective` detects Chrome on Android (`isChromeAndroid()`) and routes through the `input` event instead of `keydown`/`keypress`, because those key events are unreliable on that platform. `InputHandler.handleInput` reconstructs the intended keypress by diffing raw value lengths. Preserve this branch when touching event handling.

## Testing notes

Tests live beside the source (`*.spec.ts`) and exercise the helper classes directly (see [input.service.spec.ts](projects/ngx-currency/src/lib/input.service.spec.ts)). [mock.ts](projects/ngx-currency/src/lib/mock.ts) provides fake input-element/service helpers so logic can be tested without a real DOM. `InputHandler.timer` exists purely so `setTimeout` can be stubbed in tests — use it rather than calling `setTimeout` directly.
