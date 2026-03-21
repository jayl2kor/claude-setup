# Accessibility Patterns — Full Reference

Full ARIA patterns, focus management, keyboard navigation, and form accessibility. See [../SKILL.md](../SKILL.md) for the summary rules.

---

## ARIA Roles and Labelling

### Icon-only buttons

```tsx
// ✅ Icon-only button MUST have an accessible name
<button aria-label="Close dialog">
  <XIcon aria-hidden="true" />   {/* hide decorative icon from AT */}
</button>

// ❌ BAD — screen readers announce nothing meaningful
<button onClick={onClose}>
  <XIcon />
</button>
```

### Dynamic live regions for async updates

```tsx
// ✅ Announce async changes to screen readers without moving focus
<div role="status" aria-live="polite" aria-atomic="true">
  {isLoading ? 'Loading results…' : `${count} results found`}
</div>

// Use aria-live="assertive" only for critical errors that interrupt the user
<div role="alert" aria-live="assertive">
  {criticalError}
</div>
```

### Linking error messages to inputs

```tsx
// ✅ Error messages linked to their input via aria-describedby
<div>
  <label htmlFor="email">Email</label>
  <input
    id="email"
    type="email"
    aria-describedby={error ? 'email-error' : undefined}
    aria-invalid={!!error}
  />
  {error && <span id="email-error" role="alert">{error}</span>}
</div>
```

---

## Keyboard Navigation

Every interactive element must be reachable by keyboard and have visible focus styles. Custom widgets must implement the ARIA keyboard pattern for that widget type.

### Listbox keyboard pattern

The ARIA listbox pattern uses arrow keys to navigate options, Enter or Space to select, Home/End for first/last:

```tsx
function Listbox({ options, value, onChange }: ListboxProps) {
  const [activeIndex, setActiveIndex] = useState(
    options.findIndex(o => o.value === value)
  )

  const handleKeyDown = (e: KeyboardEvent<HTMLUListElement>) => {
    const handlers: Record<string, () => void> = {
      ArrowDown: () => setActiveIndex(i => Math.min(i + 1, options.length - 1)),
      ArrowUp:   () => setActiveIndex(i => Math.max(i - 1, 0)),
      Home:      () => setActiveIndex(0),
      End:       () => setActiveIndex(options.length - 1),
      Enter:     () => onChange(options[activeIndex].value),
      ' ':       () => onChange(options[activeIndex].value),
    }
    handlers[e.key]?.()
    if (e.key !== 'Tab') e.preventDefault()
  }

  return (
    <ul
      role="listbox"
      tabIndex={0}
      onKeyDown={handleKeyDown}
      aria-activedescendant={`option-${options[activeIndex]?.value}`}
    >
      {options.map((option, i) => (
        <li
          key={option.value}
          role="option"
          aria-selected={option.value === value}
          id={`option-${option.value}`}
          style={{ background: i === activeIndex ? 'highlight' : undefined }}
        >
          {option.label}
        </li>
      ))}
    </ul>
  )
}
```

### Tab keyboard pattern (ARIA tabs)

```tsx
function Tabs({ tabs }: { tabs: { id: string; label: string; content: ReactNode }[] }) {
  const [activeTab, setActiveTab] = useState(tabs[0].id)

  const handleTabKeyDown = (e: KeyboardEvent<HTMLButtonElement>, index: number) => {
    const tabCount = tabs.length
    if (e.key === 'ArrowRight') setActiveTab(tabs[(index + 1) % tabCount].id)
    if (e.key === 'ArrowLeft')  setActiveTab(tabs[(index - 1 + tabCount) % tabCount].id)
    if (e.key === 'Home') setActiveTab(tabs[0].id)
    if (e.key === 'End')  setActiveTab(tabs[tabCount - 1].id)
  }

  return (
    <div>
      <div role="tablist">
        {tabs.map((tab, i) => (
          <button
            key={tab.id}
            role="tab"
            aria-selected={tab.id === activeTab}
            aria-controls={`panel-${tab.id}`}
            id={`tab-${tab.id}`}
            tabIndex={tab.id === activeTab ? 0 : -1}
            onClick={() => setActiveTab(tab.id)}
            onKeyDown={e => handleTabKeyDown(e, i)}
          >
            {tab.label}
          </button>
        ))}
      </div>
      {tabs.map(tab => (
        <div
          key={tab.id}
          role="tabpanel"
          id={`panel-${tab.id}`}
          aria-labelledby={`tab-${tab.id}`}
          hidden={tab.id !== activeTab}
        >
          {tab.content}
        </div>
      ))}
    </div>
  )
}
```

---

## Focus Management for Modals and Dialogs

When a modal opens, focus must move inside it. When it closes, focus must return to the trigger element.

```tsx
export function Modal({ isOpen, onClose, triggerRef, children }: ModalProps) {
  const modalRef = useRef<HTMLDivElement>(null)

  useEffect(() => {
    if (!isOpen) return

    // Move focus into the modal on open
    const firstFocusable = modalRef.current?.querySelector<HTMLElement>(
      'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
    )
    firstFocusable?.focus()

    // Trap focus inside while open
    const handleKeyDown = (e: KeyboardEvent) => {
      if (e.key === 'Escape') { onClose(); return }
      if (e.key !== 'Tab') return

      const focusable = [...modalRef.current!.querySelectorAll<HTMLElement>(
        'button:not([disabled]), [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
      )]
      const first = focusable[0]
      const last  = focusable[focusable.length - 1]

      if (e.shiftKey && document.activeElement === first) {
        e.preventDefault()
        last.focus()
      } else if (!e.shiftKey && document.activeElement === last) {
        e.preventDefault()
        first.focus()
      }
    }

    document.addEventListener('keydown', handleKeyDown)
    return () => {
      document.removeEventListener('keydown', handleKeyDown)
      // Restore focus to the trigger on close
      triggerRef.current?.focus()
    }
  }, [isOpen, onClose, triggerRef])

  if (!isOpen) return null

  return (
    <div
      ref={modalRef}
      role="dialog"
      aria-modal="true"
      aria-labelledby="modal-title"
    >
      {children}
    </div>
  )
}

// Usage
function App() {
  const [isOpen, setIsOpen] = useState(false)
  const triggerRef = useRef<HTMLButtonElement>(null)

  return (
    <>
      <button ref={triggerRef} onClick={() => setIsOpen(true)}>
        Open dialog
      </button>
      <Modal isOpen={isOpen} onClose={() => setIsOpen(false)} triggerRef={triggerRef}>
        <h2 id="modal-title">Confirm action</h2>
        <p>Are you sure?</p>
        <button onClick={() => setIsOpen(false)}>Cancel</button>
        <button onClick={handleConfirm}>Confirm</button>
      </Modal>
    </>
  )
}
```

---

## Form Accessibility

### Controlled forms with validation (react-hook-form + Zod)

```tsx
import { useForm } from 'react-hook-form'
import { zodResolver } from '@hookform/resolvers/zod'
import { z } from 'zod'

const schema = z.object({
  email:    z.string().email('Enter a valid email address'),
  password: z.string().min(8, 'Password must be at least 8 characters'),
})
type FormValues = z.infer<typeof schema>

function LoginForm({ onSubmit }: { onSubmit: (v: FormValues) => Promise<void> }) {
  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting },
  } = useForm<FormValues>({ resolver: zodResolver(schema) })

  return (
    <form onSubmit={handleSubmit(onSubmit)} noValidate>
      <div>
        <label htmlFor="email">Email</label>
        <input
          id="email"
          type="email"
          {...register('email')}
          aria-describedby={errors.email ? 'email-error' : undefined}
          aria-invalid={!!errors.email}
        />
        {errors.email && (
          <span id="email-error" role="alert">{errors.email.message}</span>
        )}
      </div>

      <div>
        <label htmlFor="password">Password</label>
        <input
          id="password"
          type="password"
          {...register('password')}
          aria-describedby={errors.password ? 'pw-error' : undefined}
          aria-invalid={!!errors.password}
        />
        {errors.password && (
          <span id="pw-error" role="alert">{errors.password.message}</span>
        )}
      </div>

      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? 'Signing in…' : 'Sign in'}
      </button>
    </form>
  )
}
```

### Form accessibility checklist

- Use `<label htmlFor>` on every input — never rely on placeholder as the label
- Set `noValidate` on the form to disable native browser popups in favour of custom errors
- Set `aria-invalid="true"` on inputs that have errors
- Link error messages with `aria-describedby` pointing to the error span's `id`
- Use `role="alert"` on error spans so screen readers announce them immediately when they appear
- Disable the submit button (`disabled={isSubmitting}`) while the form is submitting
- Group related checkboxes/radios in `<fieldset>` with `<legend>`

---

## WCAG Quick Reference

| Criterion | Requirement |
|---|---|
| 1.1.1 Non-text content | All images have `alt`; decorative images have `alt=""` |
| 1.4.1 Use of color | Color alone is never the only way to convey information |
| 1.4.2 Audio control | Auto-play audio can be paused or stopped |
| 2.1.1 Keyboard | All functionality available by keyboard |
| 2.4.7 Focus visible | Keyboard focus is always visible |
| 4.1.2 Name, role, value | All UI components have accessible name, role, and state |
| 4.1.3 Status messages | Status changes announced without moving focus (aria-live) |
