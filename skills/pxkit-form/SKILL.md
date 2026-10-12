---
name: pxkit-form
description: Build or extend a form the pxkit way — zod schema and error keys first, react-hook-form logic, shadcn Field/FieldGroup markup, a control chooser, and submit through a Result-returning server action. Use whenever the user asks for a form, sign-up/login/checkout/contact/settings screen, input validation, a multi-step wizard, or a submit flow, and whenever a task adds two or more inputs that get submitted together, even if they only say "add a couple of fields".
---

# pxkit Form

Every form rule lives here. Load `pxkit-conventions` for the rules that apply to all code, and `pxkit-service` when the submit action doesn't exist yet.

## Gates

- **Schema and error keys before any markup.** The schema drives the type, the fields, the resolver, and the server validation.
- **Present step 2 and wait for confirmation** before building UI.
- **No raw `div` + `space-y-*` form markup** when the project has Field primitives.
- **No `try/catch` in the submit handler.** It reads a `Result`; the action already caught.
- **No generic form wrapper or field factory.** Each form is written out; two similar forms are cheaper than one configurable one.

## 1. Form library or not

| Situation | Approach |
| --- | --- |
| 2+ fields, validation rules, or submit state | react-hook-form + zod (this skill) |
| A single input (search box, inline rename) | Plain state or a form action; say so and stop |

Reuse the repo's existing form pattern, schema location, and Field components.

## 2. Schema first

- Export a zod schema from the feature's `lib/` (`lib/contact-schema.ts`); the form and the action both import it. Derive the type: `type ContactValues = z.infer<typeof contactSchema>`.
- Field validation messages live in the schema (an i18n key, or the copy itself in a single-language app).
- Submit failures are error keys declared in the action (`pxkit-service`, step 6), named by reason when known.

Present the schema, the fields with their defaults, and the submit error keys. Wait for confirmation.

## 3. Choose controls

| Need | Control |
| --- | --- |
| Simple text | `Input` |
| Dropdown, predefined options | `Select` |
| Searchable dropdown | `Combobox` |
| Boolean in settings | `Switch` |
| Boolean in a form | `Checkbox` |
| Single choice, few options | `RadioGroup` |
| 2–5 options as buttons | `ToggleGroup` |
| Multi-line text | `Textarea` |
| One-time code | `InputOTP` |

- A button inside an input uses `InputGroup` + `InputGroupAddon` with `InputGroupInput`, never a `Button` positioned over an `Input`.
- Related checkboxes, radios, or switches sit in `FieldSet` + `FieldLegend`.

## 4. Build the form

One file, `'use client'`, markup and logic together:

```tsx
// components/contact-form.tsx
'use client';

export const ContactForm = () => {
  const form = useForm<ContactValues>({
    defaultValues: { email: '', message: '' },
    mode: 'onTouched', // validate after first blur, then live; onChange only for per-keystroke feedback
    resolver: zodResolver(contactSchema),
  });
  const { errors, isSubmitting } = form.formState;

  const handleSubmit = async (values: ContactValues) => {
    const result = await submitContact(values);
    if (!result.ok) {
      form.setError('root', { message: errorCopy[result.errorKey] });
      return;
    }
    form.reset();
  };

  return (
    <form onSubmit={form.handleSubmit(handleSubmit)}>
      <FieldGroup>
        <Field data-invalid={!!errors.email}>
          <FieldLabel htmlFor="email">Email</FieldLabel>
          <Input id="email" type="email" aria-invalid={!!errors.email} {...form.register('email')} />
          {errors.email ? <FieldError>{errors.email.message}</FieldError> : null}
        </Field>
        <Field data-invalid={!!errors.message}>
          <FieldLabel htmlFor="message">Message</FieldLabel>
          <Textarea id="message" aria-invalid={!!errors.message} {...form.register('message')} />
          {errors.message ? <FieldError>{errors.message.message}</FieldError> : null}
        </Field>
        {errors.root ? <p role="alert">{errors.root.message}</p> : null}
        <Button type="submit" disabled={isSubmitting}>
          Send
        </Button>
      </FieldGroup>
    </form>
  );
};
```

- Invalid and disabled need both attributes: `data-invalid` / `data-disabled` on `Field` (styles the label and description), `aria-invalid` / `disabled` on the control.
- `register` or `Controller`, whichever the repo uses. Controlled shadcn components (`Select`, `Checkbox`) need `Controller`.
- `isSubmitting` guards against double submit; no extra flag.
- Over 60 lines: move a repeated field block into a private component above the form in the same file.

## 5. File layout

```
components/contact-form.tsx   # 'use client' — markup + useForm + submit
lib/contact-schema.ts         # zod schema + type, shared by form and action
actions/submit-contact.ts     # 'use server' — validate, call http, return Result; declares ContactErrorKey
```

A reader of the form opens at most the schema and the action.

## 6. Verify

- Every field validates per the schema; an invalid field shows on the right `Field` with both attributes set.
- Submit shows the disabled state while `isSubmitting`, then success, and each error key from step 2.
- `rg "toast|setError" <form file>` shows only resolved copy, never a string literal.
- `rg "space-y-" <form file>` returns nothing.
- The repo's `typecheck` and `lint` scripts pass.
