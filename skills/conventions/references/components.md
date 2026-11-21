# Components & UI

## Shape of a component file

- Named arrow-function `const` with a named export. Default exports only where Next.js requires them (`page.tsx`, `layout.tsx`, `sitemap.ts`).
- `'use client'` or `'use server'` is the first line when needed.
- Props are a declared `interface <Component>Props`, destructured in the signature, local helper components included. Declare only the fields the component reads: two fields, not the whole entity (`Pick<User, 'id' | 'name'>`). `children: React.ReactNode`.
- The file holds the component and everything only it uses, in dependency order (red lines in `SKILL.md`): private constants, variant maps, and sub-components above the exported component's props interface and the component. A piece moves to its own file when a second file imports it, or when the component outgrows its file and becomes a directory (`agent-craft:feature` skill).

```tsx
// pricing-table.tsx — the whole picture in one file
const TIER_BADGE = { free: 'secondary', pro: 'default', team: 'outline' } as const;

type PlanSummary = Pick<Plan, 'id' | 'name' | 'tier'>;

interface PlanRowProps {
  plan: PlanSummary;
  onSelect: (planId: string) => void;
}

const PlanRow = ({ plan, onSelect }: PlanRowProps) => (
  <li className="flex items-center gap-2">
    <Badge variant={TIER_BADGE[plan.tier]}>{plan.tier}</Badge>
    <Button onClick={() => onSelect(plan.id)}>{plan.name}</Button>
  </li>
);

interface PricingTableProps {
  plans: PlanSummary[];
  onSelect: (planId: string) => void;
}

export const PricingTable = ({ plans, onSelect }: PricingTableProps) => (
  <ul className="flex flex-col gap-2">
    {plans.map((plan) => (
      <PlanRow key={plan.id} plan={plan} onSelect={onSelect} />
    ))}
  </ul>
);
```

- Static content is a typed `as const` array in the same file, mapped to markup. A new entry is a new array item, not new JSX.
- Variants are a typed union prop backed by a lookup map or CVA, never a pile of booleans (`isPrimary`, `isLarge`).
- `memo`, `useMemo`, `useCallback` only on measured hot paths, and not at all under React Compiler.

## Compound components

A compound component is a root plus parts that consumers arrange (`Tabs`, `TabsList`, `TabsTrigger`, `TabsContent`).

- **Only when consumers arrange several slots.** A component with two props stays one component; a fixed layout stays one component with slot props.
- **One file, flat named exports.** Every part's name starts with the root's (`Stepper`, `StepperStep`), and every part is exported from the root's file, never as statics (`Stepper.Step`). If the file outgrows itself, it becomes a concept directory whose entry still exports every part.
- **The root owns the shared state.** Parts read it through a context and a guard hook declared privately in the same file, above every part that uses them. A part used outside its root throws a developer-facing message.
- **Parts without shared state are plain styled slots** (`Card`, `CardHeader`) with no context and no `'use client'`.
- Parts take only their own props; nothing passes from root to part by prop.

```tsx
'use client';

// stepper.tsx — root, parts, and their context in one file
interface StepperContextValue {
  activeStep: number;
  onStepChange: (step: number) => void;
}

const StepperContext = createContext<StepperContextValue | null>(null);

const useStepper = () => {
  const value = useContext(StepperContext);
  if (!value) throw new Error('Stepper parts must be used inside <Stepper>');
  return value;
};

interface StepperProps {
  children: React.ReactNode;
}

export const Stepper = ({ children }: StepperProps) => {
  const [activeStep, setActiveStep] = useState(0);

  return (
    <StepperContext.Provider value={{ activeStep, onStepChange: setActiveStep }}>
      <ol className="flex flex-col gap-2">{children}</ol>
    </StepperContext.Provider>
  );
};

interface StepperStepProps {
  step: number;
  children: React.ReactNode;
}

export const StepperStep = ({ step, children }: StepperStepProps) => {
  const { activeStep, onStepChange } = useStepper();

  return (
    <li>
      <Button
        variant={step === activeStep ? 'default' : 'ghost'}
        onClick={() => onStepChange(step)}
      >
        {children}
      </Button>
    </li>
  );
};

// consumer
<Stepper>
  <StepperStep step={0}>Account</StepperStep>
  <StepperStep step={1}>Billing</StepperStep>
</Stepper>;
```

## Flat trees

- Page → section → primitive. Anything deeper needs a reason.
- No pass-through components: a component that only forwards props to one child is deleted.
- No prop drilled past one intermediate. Instead, in order of preference: compose with `children` or slot props, read the data where it's used (a Server Component fetches its own data), lift state only to the lowest common owner.

```tsx
// Bad — `user` travels Dashboard → Sidebar → Menu → Avatar
<Dashboard user={user} />

// Good — compose at the top; each piece gets its data directly
<Dashboard
  sidebar={
    <Sidebar>
      <UserMenu user={user} />
    </Sidebar>
  }
/>
```

## Server or client

- Server Component by default. `'use client'` is justified only by event handlers, browser APIs, stateful hooks, or client-only libraries, and goes on the smallest leaf that needs it. Never on a page or a section.
- Animation comes from `components/motion.tsx`, a `'use client'` re-export of the motion library, so sections stay server components.
- Icons render on the server; they need no boundary.

## shadcn/ui composition

The base UI layer is shadcn/ui when the project uses it, added with the CLI (`npx shadcn@latest add <component>`), never hand-written or swapped for another library. Features compose these primitives and never redefine them. For version-specific APIs, check the current shadcn docs.

- Use the full anatomy: `Card` → `CardHeader` (`CardTitle`, `CardDescription`) → `CardContent` → `CardFooter`.
- Items live inside their group: `SelectItem` in `SelectGroup`, `DropdownMenuItem` in `DropdownMenuGroup`, `TabsTrigger` in `TabsList`, `CommandItem` in `CommandGroup`.
- An existing primitive beats custom markup:

| Instead of | Use |
| --- | --- |
| `<hr>` or `<div className="border-t">` | `<Separator />` |
| `animate-pulse` divs | `<Skeleton className="h-4 w-3/4" />` |
| a styled status `<span>` | `<Badge variant="secondary">` |
| a styled callout `div` | `<Alert>` + `AlertTitle` + `AlertDescription` |

- Overlays:

| Use case | Component |
| --- | --- |
| Focused task that needs input | `Dialog` |
| Confirm a destructive action | `AlertDialog` |
| Side panel with details or filters | `Sheet` |
| Mobile bottom panel | `Drawer` |
| Quick info on hover | `HoverCard` |
| Small content on click | `Popover` |

- `Dialog`, `Sheet`, and `Drawer` always have a title (`className="sr-only"` if hidden). `Avatar` always has `AvatarFallback`.
- `Button` has no `isLoading` prop; compose it: `<Button disabled><Spinner /> Saving…</Button>`.

## Styling

- Tailwind v4 (or the repo's utility system), CSS-first config. No inline `style`.
- Every conditional or merged class string goes through `cn()`:

```tsx
className={cn('cursor-grab p-1', isDragging ? 'cursor-grabbing bg-primary/10' : 'hover:bg-muted')}
```

- Semantic tokens (`bg-muted`, `text-primary`, `text-muted-foreground`), never arbitrary hex or raw palette colors. No manual `dark:` color overrides; tokens cover both themes.
- Customizing a component, stop at the first step that works: built-in variant → `className` for layout only (`mx-auto max-w-md`, never recoloring) → a theme token → a new variant in the component source → a wrapper component.
- `gap-*` over `space-*`; `size-*` over equal `w-*` + `h-*`; `truncate`; no manual `z-index` on overlays.
- A class string repeated inside a file becomes a local `const` in that file.
- Mobile-first breakpoints (`md:`, `lg:`); logical utilities (`ms-2`, `pe-4`).

## Icons

- Import directly from the project's icon library (check which one; never assume).
- Pass icons as component objects, never as string keys into a lookup map:

```tsx
interface StatusBadgeProps {
  icon: React.ComponentType;
}

const StatusBadge = ({ icon: Icon }: StatusBadgeProps) => <Icon />;

<StatusBadge icon={CheckIcon} />;
```

- No sizing or spacing classes on an icon inside a component that sizes its own (`Button`, `DropdownMenuItem`, `Alert`): `<Button><SearchIcon /> Search</Button>`. A standalone icon is sized by you: `<CheckIcon className="size-5 text-primary" />`.

## Accessibility

`focus-visible:` styles, `sr-only` text, `aria-hidden` on decoration, an `aria-label` on icon-only triggers, `role="alert"` on error output, `reducedMotion="user"` on animation config.

## UX

- Reuse existing patterns, tokens, and layouts before inventing a new spacing value, color, or layout shape.
- Size, weight, and color track importance; the primary action is the clearest thing on screen.
- Spacing comes from the scale. Elements align to a shared edge.
- Responsive behavior is stated per breakpoint, not left to reflow; check narrow screens for clipping and hidden actions.
- Every interactive surface has its states designed: hover, focus, active, disabled, loading, empty, error, success.
