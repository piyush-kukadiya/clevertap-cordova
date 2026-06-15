---
name: ionic-native-typings
description: How to expose a CleverTap Cordova plugin method to Ionic/Angular apps by adding it to the awesome-cordova-plugins (formerly ionic-native) typings. Use after adding/updating a public method in the Cordova plugin, when a method is missing from @awesome-cordova-plugins/clevertap or @ionic-native/clevertap, or when preparing the Ionic typings PR for a released clevertap-cordova version.
---

# Ionic Native Typings (awesome-cordova-plugins)

Ionic/Angular apps don't call the Cordova plugin directly — they consume CleverTap through the
**`awesome-cordova-plugins`** package (older name: `@ionic-native`). Until a new Cordova plugin
method is added to that package's typings, Ionic clients can't call it type-safely (this is the
typings-lag gotcha in [`example-app-patterns`](../example-app-patterns/SKILL.md), where samples must
fall back to `(CleverTap as any)`). This skill is how you close that gap.

> **When to use:** after a public method is added/updated in `www/CleverTap.js` (see
> [`api-wrapper-patterns`](../api-wrapper-patterns/SKILL.md)) **and a `clevertap-cordova` version
> carrying it has been released** — the typings PR tracks a released plugin version.

## Repos and the two-PR flow

| Role | Repo | Default branch |
|------|------|----------------|
| Upstream (the published package) | `danielsogl/awesome-cordova-plugins` | `main` |
| CleverTap fork (where we work) | `CleverTap/ionic-native` | `master` |

1. **PR #1 — into the fork (automatable).** Branch off `CleverTap/ionic-native` `master`, edit the
   one file, open a PR **against the fork's `master`** for internal CleverTap review/merge.
2. **PR #2 — fork → upstream (human-initiated).** After PR #1 merges, a maintainer opens a PR from
   `CleverTap/ionic-native:master` → `danielsogl/awesome-cordova-plugins:main` via GitHub's
   "Contribute → Open pull request". This step is a deliberate human action — our GitHub App can't
   open PRs on a third-party repo it isn't installed on.

PR title convention (see the real example
[`danielsogl/awesome-cordova-plugins#4883`](https://github.com/danielsogl/awesome-cordova-plugins/pull/4883)):
`feat(clevertap): support clevertap-cordova X.Y.Z`.

> **Branch drift:** the upstream renamed `master`→`main` historically. Verify each repo's actual
> default branch at PR time rather than assuming.

## The one file

Everything lives in a single file:

```
src/@awesome-cordova-plugins/plugins/clevertap/index.ts
```

Its shape (do NOT change the header/decorators — only add methods inside the class):

```ts
import { Injectable } from '@angular/core';
import { Cordova, AwesomeCordovaNativePlugin, Plugin } from '@awesome-cordova-plugins/core';

declare let clevertap: any;

/**
 * @name CleverTap
 * @description Cordova Plugin that wraps CleverTap SDK for Android and iOS
 * @usage
 * ```typescript
 * import { CleverTap } from '@awesome-cordova-plugins/clevertap/ngx';
 * constructor(private clevertap: CleverTap) { }
 * ```
 */
@Plugin({
  pluginName: 'CleverTap',
  plugin: 'clevertap-cordova',
  pluginRef: 'CleverTap',
  repo: 'https://github.com/CleverTap/clevertap-cordova',
  platforms: ['Android', 'iOS'],
})
@Injectable()
export class CleverTap extends AwesomeCordovaNativePlugin {
  // ... one @Cordova() method per public plugin API, grouped under banner comments ...
}
```

Methods are grouped under banner comments (`/*** Events ***/`, `/**** Custom Templates methods ****/`,
…). Add each new method under the banner that matches its feature; create a new banner only for a
genuinely new feature area.

## Per-method pattern

Every method in this file is a `@Cordova()`-decorated stub that returns `Promise<any>` — the
decorator wires it to `cordova.exec` under the hood; the body is just `return;`. **No decorator
options are used anywhere in the CleverTap plugin — always bare `@Cordova()`.**

```ts
/**
 * <one-line description>
 * @param {string} name - <description>
 * @returns {Promise<any>}
 */
@Cordova()
methodName(name: string): Promise<any> {
  return;
}
```

## Mapping `www/CleverTap.js` → typings

1. **Drop the success callback.** In the Cordova plugin a getter looks like
   `CleverTap.prototype.foo = function (arg, successCallback) { cordova.exec(successCallback, …) }`.
   In the typings the callback **disappears** — the value comes back via the returned `Promise`.
   (Confirmed in the live file: `getCleverTapID()`, `profileGetProperty(propertyName)`, and
   `variants()` are all callback-less `Promise<any>`.)
2. **Mirror the method name exactly** — it is the `cordova.exec` action string and the
   `pluginRef`'s method; a typo means the call won't reach native.
3. **Type the remaining args** JS → TS: `string` / `number` / `boolean`; an object → `any`
   (or a named interface if one already exists in the file); an array → `any[]`. Return is always
   `Promise<any>`.
4. **Deprecations:** if the plugin removed/deprecated a method, mark the typings method with
   `@deprecated <reason / replacement>` in its JSDoc — do NOT delete it (the file already keeps
   deprecated entries like the old attribution-id getters).

### Worked examples

Fire-and-forget, no args — `CleverTap.prototype.unmute = function () { cordova.exec(null, null, "CleverTapPlugin", "unmute", []) }`:
```ts
/**
 * Resumes event tracking, overriding any active mute period.
 * @returns {Promise<any>}
 */
@Cordova()
unmute(): Promise<any> {
  return;
}
```

Getter with a success callback — `CleverTap.prototype.variants = function (successCallback) { cordova.exec(successCallback, null, "CleverTapPlugin", "variants", []) }`:
```ts
/**
 * Fetches the active A/B experiment variants for the current user.
 * @returns {Promise<any>}
 */
@Cordova()
variants(): Promise<any> {
  return;       // the successCallback is dropped — its data resolves the Promise
}
```

Method with an argument — `recordEventWithName(eventName)`:
```ts
/**
 * Records an event with the given name.
 * @param {string} eventName - the name of the event
 * @returns {Promise<any>}
 */
@Cordova()
recordEventWithName(eventName: string): Promise<any> {
  return;
}
```

## Checklist (per method)

- [ ] Added to `src/@awesome-cordova-plugins/plugins/clevertap/index.ts` (the only file to touch).
- [ ] Method name matches `www/CleverTap.js` exactly.
- [ ] `@Cordova()` (bare) + returns `Promise<any>` + body is `return;`.
- [ ] Success-callback param dropped; other args typed.
- [ ] JSDoc with `@param`/`@returns`; `@deprecated` kept where applicable.
- [ ] Placed under the matching banner section.
- [ ] PR #1 opened against `CleverTap/ionic-native` `master`; upstream PR left as the human step.

## Common mistakes

❌ Keeping the `successCallback` parameter in the typings signature
✅ Drop it — the Cordova decorator resolves the `Promise` with the callback's value

❌ `@Cordova({ ... })` with options / a different return type
✅ Bare `@Cordova()` returning `Promise<any>` — matches every method in this file

❌ Editing the `@Plugin` block, imports, or `declare let clevertap` to add a method
✅ Only add methods inside the class body

❌ Deleting a method the plugin deprecated
✅ Keep it, add `@deprecated` to its JSDoc

❌ Opening the upstream PR via automation
✅ Automation opens PR #1 into the fork; a maintainer opens the upstream PR by hand
