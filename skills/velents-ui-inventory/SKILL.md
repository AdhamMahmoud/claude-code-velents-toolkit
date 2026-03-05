---
name: velents-ui-inventory
description: "MANDATORY: Complete inventory of existing UI components, colors, patterns. REUSE ONLY — never create new components when existing ones work. No over-designing."
user-invocable: false
---

# VelentsAI UI Component Inventory — REUSE FIRST

> **RULE #1**: NEVER create a new component when an existing one can do the job.
> **RULE #2**: NEVER invent new colors, spacing, or patterns. Use what exists.
> **RULE #3**: Keep it simple. No over-designing. No over-engineering.

## shadcn/ui Components (46 installed — USE THESE)

### Layout & Structure
| Component | Import | Use For |
|---|---|---|
| `Card` | `@/components/ui/card` | Card, CardHeader, CardTitle, CardDescription, CardAction, CardContent, CardFooter |
| `Separator` | `@/components/ui/separator` | Visual dividers |
| `ScrollArea` | `@/components/ui/scroll-area` | Scrollable containers |
| `Tabs` | `@/components/ui/tabs` | Tabs, TabsList, TabsTrigger, TabsContent |
| `Accordion` | `@/components/ui/accordion` | Collapsible sections |
| `Collapsible` | `@/components/ui/collapsible` | Single collapsible |
| `ResizablePanel` | `@/components/ui/resizable` | Resizable panels |
| `AspectRatio` | `@/components/ui/aspect-ratio` | Fixed aspect ratio containers |

### Navigation
| Component | Import | Use For |
|---|---|---|
| `Sidebar` | `@/components/ui/sidebar` | App sidebar (already built — DO NOT rebuild) |
| `Breadcrumb` | `@/components/ui/breadcrumb` | Page breadcrumbs |
| `NavigationMenu` | `@/components/ui/navigation-menu` | Top navigation |
| `Pagination` | `@/components/ui/pagination` | Page pagination |
| `Menubar` | `@/components/ui/menubar` | Menu bars |

### Forms & Input
| Component | Import | Use For |
|---|---|---|
| `Form` | `@/components/ui/form` | Form, FormField, FormItem, FormLabel, FormControl, FormDescription, FormMessage |
| `Input` | `@/components/ui/input` | Text input |
| `Textarea` | `@/components/ui/textarea` | Multi-line text |
| `Select` | `@/components/ui/select` | Dropdown select |
| `Checkbox` | `@/components/ui/checkbox` | Checkboxes |
| `RadioGroup` | `@/components/ui/radio-group` | Radio buttons |
| `Switch` | `@/components/ui/switch` | Toggle switch |
| `Slider` | `@/components/ui/slider` | Range slider |
| `Calendar` | `@/components/ui/calendar` | Date picker calendar |
| `InputOTP` | `@/components/ui/input-otp` | OTP input |

### Buttons & Actions
| Component | Import | Use For |
|---|---|---|
| `Button` | `@/components/ui/button` | Variants: `default`, `destructive`, `outline`, `secondary`, `ghost`, `link`. Sizes: `default`, `sm`, `lg`, `icon` |
| `Toggle` | `@/components/ui/toggle` | Toggle button |
| `ToggleGroup` | `@/components/ui/toggle-group` | Group of toggles |

### Overlay & Dialogs
| Component | Import | Use For |
|---|---|---|
| `Dialog` | `@/components/ui/dialog` | Standard modal dialogs |
| `AlertDialog` | `@/components/ui/alert-dialog` | Destructive confirmation dialogs |
| `Sheet` | `@/components/ui/sheet` | Side panel drawers |
| `Drawer` | `@/components/ui/drawer` | Bottom drawer (vaul) |
| `Popover` | `@/components/ui/popover` | Floating content |
| `HoverCard` | `@/components/ui/hover-card` | Hover previews |
| `Tooltip` | `@/components/ui/tooltip` | Tooltips (with TooltipProvider) |
| `ContextMenu` | `@/components/ui/context-menu` | Right-click menus |
| `DropdownMenu` | `@/components/ui/dropdown-menu` | Action menus |
| `Command` | `@/components/ui/command` | Command palette / search |

### Data Display
| Component | Import | Use For |
|---|---|---|
| `Table` | `@/components/ui/table` | Data tables (with @tanstack/react-table) |
| `Badge` | `@/components/ui/badge` | Status badges. Variants: `default`, `secondary`, `warning`, `info`, `success`, `destructive`, `outline` |
| `Avatar` | `@/components/ui/avatar` | User/agent avatars |
| `Progress` | `@/components/ui/progress` | Progress bars |
| `Skeleton` | `@/components/ui/skeleton` | Loading placeholders |
| `Alert` | `@/components/ui/alert` | Alert messages |
| `Label` | `@/components/ui/label` | Form labels |

### Feedback
| Component | Import | Use For |
|---|---|---|
| `Toaster` (sonner) | `@/components/ui/sonner` | Toast notifications — ALWAYS use `import { toast } from "sonner"` |

### Charts
| Component | Import | Use For |
|---|---|---|
| `ChartContainer` | `@/components/ui/chart` | Recharts wrapper with theme-aware colors |
| `ChartTooltip` | `@/components/ui/chart` | Chart tooltips |
| `ChartLegend` | `@/components/ui/chart` | Chart legends |

### Special
| Component | Import | Use For |
|---|---|---|
| `Kanban` | `@/components/ui/kanban` | Drag-and-drop kanban boards |
| `Carousel` | `@/components/ui/carousel` | Image/content carousels |

## Custom Components (already built — REUSE)

| Component | Import | Purpose |
|---|---|---|
| `Spinner` | `@/components/ui/spinner` | Loading spinner |
| `SkeletonLoader` | `@/components/ui/skeleton-loader` | Pre-built skeletons: `AgentsSkeleton`, `ConversationsSkeleton`, `PhoneCallsSkeleton`, `BatchCallsSkeleton` |
| `Icon` | `@/components/icon` | Dynamic icon by name: `<Icon name="Settings" className="size-4" />` |
| `LanguageSwitcher` | `@/components/LanguageSwitcher` | EN/AR language toggle |
| `ExportButton` | `@/components/CardActionMenus` | Export dropdown (Excel/PDF) |
| `CustomDateRangePicker` | `@/components/custom-date-range-picker` | Date range with presets |
| `DateTimePicker` | `@/components/date-time-picker` | Date-time picker |
| `NoCreditModal` | `@/components/no-creidts-modal` | No credits dialog |
| `MinimalTiptap` | `@/components/ui/custom/minimal-tiptap` | Rich text editor |
| `CountAnimation` | `@/components/ui/custom/count-animation` | Animated number counter |
| `PermissionGate` | `@/components/auth/permission-gate` | RBAC-gated UI rendering |
| `PermissionTooltip` | `@/components/auth/permission-tooltip` | Tooltip for missing permissions |
| `ReadOnlyBadge` | `@/components/auth/read-only-badge` | Read-only indicator |
| `AuthGuard` | `@/components/auth/auth-guard` | Auth redirect guard |
| `ForbiddenHandler` | `@/components/auth/forbidden-handler` | Global 403 handler |

### Chat UI Primitives (in `components/ui/custom/prompt/`)
| Component | Purpose |
|---|---|
| `ChatContainer` | Chat message container |
| `ChatInput` | Message input |
| `ChatMessage` | Single message bubble |
| `ChatMarkdown` | Markdown renderer |
| `CodeBlock` | Syntax-highlighted code |
| `ChatLoader` | Typing indicator |
| `ScrollButton` | Scroll-to-bottom |
| `Suggestion` | Quick reply suggestions |

## Color System — USE SEMANTIC TOKENS ONLY

```
NEVER use raw colors like `bg-blue-500` or `text-gray-700`.
ALWAYS use semantic tokens:
```

| Token | Use For |
|---|---|
| `bg-background` / `text-foreground` | Page background / main text |
| `bg-card` / `text-card-foreground` | Card backgrounds |
| `bg-primary` / `text-primary-foreground` | Primary buttons, key actions |
| `bg-secondary` / `text-secondary-foreground` | Secondary elements |
| `bg-muted` / `text-muted-foreground` | Subdued text, helper text |
| `bg-accent` / `text-accent-foreground` | Hover states, highlights |
| `bg-destructive` / `text-destructive` | Errors, delete actions |
| `border-border` | All borders |
| `border-input` | Form input borders |
| `ring-ring` | Focus rings |
| `bg-sidebar` | Sidebar background |
| `--chart-1` through `--chart-5` | Chart colors (via ChartContainer) |

### Badge Variants (for status display)
| Status | Variant |
|---|---|
| Active / Success | `<Badge variant="success">` |
| Warning / Pending | `<Badge variant="warning">` |
| Info / Processing | `<Badge variant="info">` |
| Error / Failed | `<Badge variant="destructive">` |
| Default | `<Badge variant="default">` |
| Inactive | `<Badge variant="secondary">` |

## Layout Patterns — FOLLOW EXACTLY

### Page Header Pattern
```tsx
<div className="space-y-4">
  <div className="flex flex-row items-center justify-between">
    <div className="flex flex-col items-start space-y-2">
      <h1 className="text-xl font-bold tracking-tight lg:text-2xl">{t("title")}</h1>
      <p className="text-muted-foreground max-w-xl text-sm">{t("description")}</p>
    </div>
    <div className="flex items-center space-x-2">
      {/* Action buttons */}
    </div>
  </div>
  {/* Page content */}
</div>
```

### Form Pattern
```tsx
const form = useForm<z.infer<typeof schema>>({
  resolver: zodResolver(schema),
  defaultValues: { ... }
});

<Form {...form}>
  <form onSubmit={form.handleSubmit(onSubmit)} className="space-y-4">
    <FormField control={form.control} name="field" render={({ field }) => (
      <FormItem>
        <FormLabel>{t("label")}</FormLabel>
        <FormControl><Input {...field} /></FormControl>
        <FormMessage />
      </FormItem>
    )} />
  </form>
</Form>
```

### Data Table Pattern
```tsx
// Use @tanstack/react-table + shadcn Table components
// Support table/grid view toggle (localStorage persisted)
// Pagination: prev/next with API meta { current_page, last_page, total }
// Row actions: DropdownMenu with MoreHorizontal trigger
```

### Dialog Pattern
```tsx
<Dialog open={open} onOpenChange={setOpen}>
  <DialogContent className="sm:max-w-md">
    <DialogHeader>
      <DialogTitle>{title}</DialogTitle>
      <DialogDescription>{description}</DialogDescription>
    </DialogHeader>
    {/* body */}
    <DialogFooter>
      <Button variant="outline" onClick={() => setOpen(false)}>Cancel</Button>
      <Button onClick={handleAction}>Confirm</Button>
    </DialogFooter>
  </DialogContent>
</Dialog>
```

### Destructive Confirmation Pattern
```tsx
// Use AlertDialog, NOT Dialog
<AlertDialog>
  <AlertDialogContent>
    <AlertDialogHeader>
      <AlertDialogTitle>Are you sure?</AlertDialogTitle>
      <AlertDialogDescription>This action cannot be undone.</AlertDialogDescription>
    </AlertDialogHeader>
    <AlertDialogFooter>
      <AlertDialogCancel>Cancel</AlertDialogCancel>
      <AlertDialogAction onClick={handleDelete}>Delete</AlertDialogAction>
    </AlertDialogFooter>
  </AlertDialogContent>
</AlertDialog>
```

## RTL Rules — MANDATORY

1. Check locale: `const isRTL = locale === "ar"`
2. Use CSS logical properties: `ms-*` `me-*` `ps-*` `pe-*` `start-*` `end-*`
3. Sidebar side: `side={isRTL ? "right" : "left"}`
4. Dropdown direction: `side={isRTL ? "left" : "right"}`
5. Icon ordering: `className={isRTL ? "order-2" : ""}` on icons in nav items
6. All text from `useTranslations()` — NEVER hardcode English strings

## Icon Sizing Convention
- `size-4` (16px) — inline icons, button icons
- `size-5` (20px) — nav icons, medium emphasis
- `size-6` (24px) — card icons, headers
- `size-8` (32px) — empty states, hero icons

## FORBIDDEN — Do NOT Do These

1. **DO NOT** create new Button components — use `<Button>` with existing variants
2. **DO NOT** create new Modal components — use `Dialog` or `AlertDialog`
3. **DO NOT** create new Toast systems — use `toast` from `sonner`
4. **DO NOT** create new Table components — use `@tanstack/react-table` + shadcn Table
5. **DO NOT** create new Form wrappers — use `react-hook-form` + `zod` + shadcn Form
6. **DO NOT** create new Loading states — use `SkeletonLoader` or `Spinner`
7. **DO NOT** create new Icon components — use `<Icon name="..." />` or direct lucide import
8. **DO NOT** use raw Tailwind colors (`bg-blue-500`) — use semantic tokens (`bg-primary`)
9. **DO NOT** create new color themes — use existing theme presets
10. **DO NOT** add new CSS files — extend `globals.css` if absolutely needed
11. **DO NOT** create new date pickers — use `CustomDateRangePicker` or `Calendar`
12. **DO NOT** build custom dropdowns — use `Select` or `DropdownMenu`
13. **DO NOT** create new chart wrappers — use `ChartContainer` from `@/components/ui/chart`
14. **DO NOT** build custom permission checks — use `<PermissionGate>`
