# Naming & TypeScript

## Files

- kebab-case for every filename, components included: `submit-button.tsx` exports `SubmitButton`, `use-countdown.ts` exports `useCountdown`. The filename mirrors the main export.
- One public concept per file: a component with its compound parts, a hook, or a service. Everything private to that concept stays in the file.
- `.tsx` if and only if the file contains JSX.
- No file named for a category instead of a job: `utils.ts`, `helpers.ts`, `common.ts`, `misc.ts`. Name the file for what it does (`format.ts`, `invoice-filters.ts`).

## Identifiers

- camelCase for values and functions; PascalCase for components, types, and interfaces. No `I` prefix.
- Booleans read as questions: `isLoading`, `hasNewStep`, `shouldBlock`.
- Internal event handlers are `handle*` (`handleFileSelect`); callback props are `on*` (`onUploadComplete`).
- Name the business meaning, not the technical accident: `calculateDiscount`, `isEligibleForRenewal`, not `doAction`, `checkFlag2`, `processData`, `manager`.

## Declaring types

- `interface` for object shapes; `type` for unions and aliases. Props are `interface <Component>Props`.
- **Declare, don't inline.** A new object shape or union gets a name above its first use, then is referenced. Never `({ a }: { a: string })`, never `useState<'idle' | 'loading'>()`. Composing declared names inline is fine: `Result<Invoice, InvoiceErrorKey>`, `Pick<User, 'id' | 'name'>`.
- String-literal unions, or an `as const` object with `keyof typeof` when the values carry data. Never `enum`.

```ts
const uploadLimits = {
  image: { maxSizeMb: 5, accept: ['image/png', 'image/jpeg'] },
  document: { maxSizeMb: 20, accept: ['application/pdf'] },
} as const;

type UploadKind = keyof typeof uploadLimits;
```

- **Derive, don't restate.** A shape that exists somewhere is derived from it: `z.infer<typeof schema>`, `keyof typeof`, `ReturnType`, `Pick`, `Omit`. A second hand-written copy drifts.
- **Annotate contracts, infer the rest.** Write types on parameters, exported signatures, and a parser's return (so a wrong field errors at the boundary). Let initialized consts and obvious returns infer. Annotate empty-array seeds and literals that would widen.

## Making invalid states unrepresentable

- Async lifecycle is one `status` union, never parallel booleans (details in `state.md`).
- Split entity shapes by lifecycle stage instead of making fields optional:

```ts
interface DraftUser {
  email: string;
  role: UserRole;
}

interface SavedUser {
  id: string;
  email: string;
  role: UserRole;
  status: UserStatus;
}
```

- `unknown` for open values, narrowed with a type guard or a zod schema. Never `any`.

```ts
export const isNotNull = <T>(value: T | null): value is T => value !== null;
```

## Misc

- Array syntax (`T[]` or `Array<T>`) follows the repo; never rewrite one into the other.
- `readonly` where immutability is intended.
- JSDoc only where the signature doesn't already say it; no `@param` lines restating the types.
