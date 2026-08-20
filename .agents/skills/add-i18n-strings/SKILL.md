---
name: add-i18n-strings
description: >
  Workflow for adding or modifying user-facing text. Use whenever adding UI
  strings, messages, or errors; creating or editing i18n keys or locale files;
  or formatting dates, numbers, or currencies for users. Covers key
  conventions, locale-file parity, interpolation/pluralization, the
  key-change gate, and i18n testing. Library-agnostic; supplements
  docs/06-i18n.md.
---

# Add i18n Strings

Every string a user can see comes from the i18n system. A change with a
hardcoded user-facing string is not done. This skill covers the workflow for
adding text correctly: key first, all locales in the same change, then tests.
The library is chosen by the stack (`docs/10-stack-guidance.md`); the rules
here hold regardless of library.

## 1. Pre-flight

Before writing code, confirm:

- **Namespace** — which feature/domain do the keys belong to
  (`auth.login.title`, `checkout.cart.empty`, `errors.notFound`)? New feature
  gets its own namespace; shared error messages go under `errors.*`.
- **New key vs. existing key** — adding new keys follows the normal workflow
  below. Renaming or removing an existing key routes to the key-change gate
  (section 8) before anything else.
- **Locales** — list the locale files that exist (`messages/*.json`). Every
  one of them is updated in this change; there is no "add the translation
  later" path.
- **Library** — the project's i18n solution per `docs/10-stack-guidance.md`
  (e.g. `next-intl`, `react-i18next`, `@nuxtjs/i18n`). Use its interpolation,
  pluralization, and hooks; do not invent parallel mechanisms.

## 2. Key conventions

- **Message keys, not English text.** `auth.login.title` is correct; `"Log in"`
  as a key is not. Keys are stable identifiers, part of the public contract.
- **Namespaced by feature/domain** — `checkout.total`, `profile.avatar.upload`,
  `errors.notFound`.
- **Typed keys.** Keys are defined in a single source of truth (a typed key
  map or generated types, typically in `packages/shared`) so that referencing
  a missing key is a compile-time error. If the project lacks this, note it
  and ask before proceeding.
- **No key duplication.** Before adding a key, check whether an equivalent key
  already exists in the namespace. Reuse over re-add.

## 3. Locale files

- One file per locale: `messages/en.json`, `messages/fr.json`, etc. All files
  share the same structure.
- **Default locale (`en`) is the reference.** Add keys there first; every
  other locale must match its key set exactly.
- Add the key to **all locale files in the same change**. If a translation is
  genuinely unavailable, copy the default-locale value and flag it — never
  omit the key (a missing key breaks parity and renders nothing).
- Parity is verified by a CI check or type generation. If the project has a
  parity test, run it; if it does not, add the key-parity check to the test
  suite as part of this change.

## 4. Interpolation and pluralization

- Dynamic values use the library's interpolation: `checkout.total` →
  `"Total: {total}"`. Never splice strings into translated text.
- Plurals use the library's plural rules: `items.count` →
  `"{count} item"` / `"{count} items"`. Never branch on count in the
  component.
- **Never build sentences by concatenating translated fragments.** Word order
  differs across languages; a full-sentence key with placeholders is the only
  safe shape. If two fragments are always shown together, they are one key.

## 5. Formatting

Dates, times, numbers, and currencies are user-facing content, not
implementation details:

- Use the platform's locale-aware formatters: `Intl.DateTimeFormat`,
  `Intl.NumberFormat`, `Intl.RelativeTimeFormat` — with the active locale.
- Never hardcode date or number formats in components (`"YYYY-MM-DD"`,
  `toFixed(2)`, manual currency symbols).
- If the i18n library provides formatting helpers, prefer them so the locale
  resolution stays consistent with message rendering.

## 6. Server/client consistency

- Server-rendered and client-rendered content use the **same message catalog
  and the same locale resolution**.
- The locale used for the server render must match the client hydration;
  a mismatch produces hydration errors. Resolve the locale once per request
  and pass it through.

## 7. API error mapping

- API error `code`s are stable machine-readable strings (see the
  `add-api-endpoint` skill and `docs/04-api-design.md`). The client maps
  `code` → i18n key (e.g. `errors.notFound`, `errors.validation`).
- API `message` fields are developer-facing and are **never rendered to
  users**. If user-facing text is needed for an error, it is a new key under
  `errors.*` added through this workflow.

## 8. Key-change gate

Renaming or removing an existing key is a **public-contract change** — it
breaks every consumer of the key, including shipped clients and locale files
outside the default. It must not happen silently.

Before renaming or removing a key:

1. Call it out in the plan phase, before any code is written.
2. Present to the user: the affected key(s), every usage site, and every
   locale file touched.
3. Get explicit confirmation. Only then apply the change — key updated at all
   usage sites and in **all** locale files in the same change.

Additive changes (new keys, new values for existing keys) do not require the
gate.

## 9. Testing

- Unit tests assert on **i18n keys or rendered translated output**, never on
  hardcoded English strings.
- The locale-parity test verifies every locale file has the same key set as
  the default locale. Extend it if new namespaces are added.
- E2E tests run in at least the default locale and one additional locale;
  user-visible flows should be exercised in both to catch missing
  translations (`docs/03-tooling.md`).

## Checklist

Before considering the strings done, verify:

- [ ] No hardcoded user-facing strings anywhere in the change.
- [ ] Keys are message keys (not English text), namespaced by feature/domain.
- [ ] Keys are typed — a missing key is a compile-time error.
- [ ] Keys added to the default locale and **all** other locale files in the
      same change.
- [ ] Locale key sets are consistent (parity check/test passes).
- [ ] Interpolation and pluralization use the library, not concatenation.
- [ ] Dates, numbers, and currencies use `Intl` formatters with the active
      locale.
- [ ] Any key rename/removal passed the key-change gate with explicit user
      confirmation.
- [ ] Tests assert on keys or translated output, not English strings.
- [ ] Full verification pipeline passes (format, lint, typecheck, test,
      build).
