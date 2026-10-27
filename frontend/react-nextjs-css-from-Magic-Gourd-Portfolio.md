# Interview Notes: React, Next.js, CSS Animation, Web Performance

> Every snippet is from my project (Magic Gourd portfolio, Next.js 15 static export, React 19, Tailwind CSS v4).
> For each concept: what it is, the project code, and a likely interview question.

## Contents

- [React](#react)
  1. Native `<dialog>` with `useRef` (a modal without a library)
  2. `useId` for accessible ids
  3. A finite state machine with `useState`
  4. Timers in effects, with cleanup
  5. Changing `key` to remount a component (and restart its animation)
  6. Functional state updates
  7. `useRef` for DOM nodes and imperative browser APIs
  8. Subscribing to browser events in `useEffect` (scroll, resize, `matchMedia`, `visibilitychange`)
  9. `IntersectionObserver`: pause work that is off screen
  10. Two conditions, one result: track each source separately
  11. Content in data files, UI in components
  12. Rendering a list, and keys in nested lists
  13. A tiny parser that returns JSX (`**bold**` → `<strong>`)
  14. Form validation with an object of checks, focus the first error
  15. A honeypot field against spam bots
  16. Accessible custom buttons (`role`, `tabIndex`, `aria-pressed`, Enter/Space)
  17. `aria-disabled` vs `disabled`
  18. One component, two layouts: a `preview` prop
  19. `useLayoutEffect` vs `useEffect`: run before paint
  20. Capture-phase listeners: run before another component's handler
- [Next.js (App Router)](#nextjs-app-router)
  21. Server Components by default, `"use client"` only where needed
  22. Static export (`output: 'export'`)
  23. `next/font`: self-hosted fonts as CSS variables (and variable fonts with `opsz`)
  24. Metadata API
  25. `next/link` vs a plain `<a>` (and `usePathname`)
  26. Layouts persist across navigations, pages remount
  27. Hydration mismatches and `suppressHydrationWarning`
- [CSS and animation](#css-and-animation)
  28. Animate only `transform` and `opacity`
  29. CSS custom properties per element: staggered animations
  30. Playing a keyframe animation in reverse
  31. `prefers-reduced-motion`
  32. `transform-box: fill-box` for SVG transforms
  33. Container query units (`cqw`)
  34. Tailwind v4: `@theme` tokens and `@layer`
  35. Sticky nav: `scroll-padding-top` and smooth scrolling
  36. Scroll-driven animations (`animation-timeline: view()`)
- [Browser APIs](#browser-apis)
  37. Web Audio API: sounds without audio files
  38. Restarting a CSS animation from JavaScript
- [Web performance](#web-performance)
  39. Core Web Vitals: LCP, CLS, INP
  40. Layout shift from web fonts
  41. Measuring before optimizing (Lighthouse, a fair comparison)
  42. Removing an animation library

---

## React

### 1. Native `<dialog>` with `useRef` (a modal without a library)

The HTML `<dialog>` element gives you a real modal for free: `showModal()` adds the backdrop, traps focus inside, closes on Esc, and puts focus back on the button that opened it. React only needs a ref to call it.

```jsx
// components/Dialog.jsx
const Dialog = ({ label, what, project }) => {
  const dialogRef = useRef(null);
  const titleId = useId();

  return (
    <>
      <button type="button" onClick={() => dialogRef.current.showModal()}>
        {label}
      </button>
      {/* a click on the backdrop hits the <dialog> itself, not the card inside it */}
      <dialog
        ref={dialogRef}
        aria-labelledby={titleId}
        onClick={(e) => e.target === e.currentTarget && dialogRef.current.close()}
      >
        <h2 id={titleId}>Not ready yet</h2>
        <p>The {what} for {project} isn&apos;t available yet.</p>
        <button type="button" autoFocus onClick={() => dialogRef.current.close()}>
          Got it
        </button>
      </dialog>
    </>
  );
};
```

```css
/* globals.css: stop the page scrolling behind an open modal */
html:has(dialog[open]) { overflow: hidden; }
```

- `showModal()` = modal (backdrop, focus trap, Esc). `show()` = non-modal.
- `e.target === e.currentTarget` is true only when the click lands on the dialog itself (the backdrop area), not on its children.
- Opening/closing is imperative (a method call), so a ref fits better than state here.

**Interview Q:** *How would you build an accessible modal?* Use the native `<dialog>` with `showModal()`: it handles focus trapping, Esc, the backdrop and returning focus. Label it with `aria-labelledby`. Libraries (Radix, Headless UI) are an option when you need more control, but the native element covers most cases with zero JS weight.

### 2. `useId` for accessible ids

`useId()` makes an id that is unique per component instance and identical on the server and the client, so it is safe with server rendering.

```jsx
const titleId = useId();
<dialog aria-labelledby={titleId}>
  <h2 id={titleId}>Not ready yet</h2>
</dialog>
```

**Interview Q:** *Why not `Math.random()` or a counter for ids?* They differ between the server render and the browser render, which causes a hydration mismatch. A hard-coded id breaks when the component is rendered twice on one page. `useId` solves both.

### 3. A finite state machine with `useState`

Instead of several booleans (`isOpen`, `isAnimating`, …) that can contradict each other, keep one state with a fixed set of values. Derive everything else from it.

```jsx
// components/HeroGourd.jsx
// closed → opening → open → closing → closed
const [state, setState] = useState("closed");
const busy = state === "opening" || state === "closing";   // derived, not stored
const open = state === "open";

const toggle = () => {
  if (busy) return;                 // ignore clicks during the animation
  if (state === "closed") {
    playOpen();
    setState("opening");
  } else {
    playClose();
    setState("closing");
  }
};

<div data-state={state} className="gourd">…</div>   // CSS animates per state
```

```css
.gourd[data-state="opening"] .lid { animation: lid 3.6s ease-in-out 0.2s both; }
.gourd[data-state="closed"] .pop  { opacity: 0; }
```

**Interview Q:** *Why one `state` string instead of `isOpen` + `isAnimating`?* Two booleans allow four combinations, and some are impossible (`isOpen && isClosing`). A single state makes impossible states impossible, and each transition lives in one place. For bigger flows you can use `useReducer` or XState.

### 4. Timers in effects, with cleanup

An effect that starts a timer must clear it in the cleanup. Otherwise the timer can fire after the component unmounted, or after the state already changed.

```jsx
// components/HeroGourd.jsx: move on once the animation has finished
useEffect(() => {
  if (!busy) return;
  const timer = setTimeout(() => {
    if (state === "opening") {
      setState("open");
    } else {
      setState("closed");
      setTech(DEFAULT_TECH);
      setPicks(0);
    }
  }, ANIMATION_MS);
  return () => clearTimeout(timer);
}, [state, busy]);
```

```jsx
// components/Contact.jsx: hide the success message after 3s
useEffect(() => {
  if (status !== "sent") return;
  const timer = setTimeout(() => setStatus(""), SENT_VISIBLE_MS);
  return () => clearTimeout(timer);  // status changed first, or the section unmounted
}, [status]);
```

**Interview Q:** *What happens if you forget `clearTimeout`?* The callback still runs later and sets state on a component that may be gone or in a different state, which causes bugs (and in Strict Mode, effects run twice in development, so a missing cleanup shows up as double timers).

### 5. Changing `key` to remount a component (and restart its animation)

When a `key` changes, React throws away the old element and mounts a new one. A fresh DOM node starts its CSS animation from the beginning. This replaces the classic "remove class, force reflow, add class" trick.

```jsx
// components/HeroGourd.jsx
const [picks, setPicks] = useState(0);   // how many times a chip was clicked

// every pick remounts the lid, so the "pour" animation plays again (even for the same chip)
<g key={picks} className={picks ? "lidwrap pour" : "lidwrap"}>…</g>

// and remounts the code lines, so they fly in again
<g key={picks} className={picks ? "picked font-mono" : "font-mono"}>
  <CodeLines tech={tech} picked={picks > 0} />
</g>
```

**Interview Q:** *How do you reset a component's internal state?* Change its `key`. React treats it as a different component, so all state, refs and DOM are recreated. Common uses: resetting a form when the selected user changes, or replaying an animation.

### 6. Functional state updates

When the next state depends on the previous one, pass a function.

```jsx
setPicks((count) => count + 1);
setMessage((m) => (m + 1) % heroData.caravan.length);   // cycle through messages
setErrors((prev) => ({ ...prev, [name]: false }));      // update one key of an object
```

**Interview Q:** *Why `setCount(c => c + 1)`?* Inside event listeners, timers and effects, the `count` variable can be stale (it is the value from the render that created the closure). The function form always receives the latest value.

### 7. `useRef` for DOM nodes and imperative browser APIs

Refs hold things React doesn't render: DOM nodes, observers, timers. Changing `.current` does not re-render.

```jsx
// components/Caravan.jsx
const strip = useRef(null);
const walker = useRef(null);
const bubble = useRef(null);

useEffect(() => {
  const c = strip.current;            // the real DOM node, available after mount
  const speed = (c.offsetWidth + START_OFFSET + 20) / WALK_SECONDS;
  // …
}, []);

<div ref={strip} className="caravan">…</div>
```

**Interview Q:** *When is `ref.current` set?* After React commits the DOM, so read it in an effect or an event handler, never during render (it is `null` on the first render).

### 8. Subscribing to browser events in `useEffect`

Pattern: subscribe in the effect, run once immediately for the current state, unsubscribe in the cleanup.

```jsx
// components/Navbar.jsx: scroll-spy
useEffect(() => {
  const update = () => {
    setStuck(window.scrollY > 40);
    // …find the section under the nav, setActive(current)
  };
  update();                                                // page may load half-scrolled
  window.addEventListener("scroll", update, { passive: true });
  window.addEventListener("resize", update);
  return () => {
    window.removeEventListener("scroll", update);
    window.removeEventListener("resize", update);
  };
}, [isHome]);
```

```jsx
// components/HeroGourd.jsx: follow the user's motion setting
useEffect(() => {
  const reduce = window.matchMedia("(prefers-reduced-motion: reduce)");
  const sync = () => reduce.matches && setState("open");
  sync();
  reduce.addEventListener("change", sync);
  return () => reduce.removeEventListener("change", sync);
}, []);
```

- `{ passive: true }` promises the listener won't call `preventDefault()`, so the browser can scroll without waiting for JS.
- The cleanup must remove the **same function reference** that was added.

**Interview Q:** *Why can't you read `window` during render in Next.js?* Components also render on the server, where `window` doesn't exist. Browser-only code goes in `useEffect` (runs only in the browser, after hydration) or in event handlers.

### 9. `IntersectionObserver`: pause work that is off screen

`IntersectionObserver` tells you when an element enters or leaves the viewport, without a scroll listener doing layout math on every frame.

```jsx
// components/Caravan.jsx: pause the walking animation while it can't be seen
const observer = new IntersectionObserver(([entry]) => {
  offScreen = !entry.isIntersecting;
  update();
});
observer.observe(c);
return () => observer.disconnect();
```

```css
.caravan[data-paused="true"] * { animation-play-state: paused !important; }
```

**Interview Q:** *Name uses of `IntersectionObserver`.* Lazy-loading images, infinite scroll, scroll-triggered animations, analytics ("was this seen?"), and pausing videos/animations off screen to save CPU and battery.

### 10. Two conditions, one result: track each source separately

The caravan must pause when it is off screen **or** the tab is hidden. If each event simply toggled "paused", returning to the tab would un-pause it even while it's still off screen. Keep each input, compute the output from both.

```jsx
let offScreen = false;
const update = () => {
  c.dataset.paused = offScreen || document.hidden ? "true" : "false";
};
const observer = new IntersectionObserver(([entry]) => {
  offScreen = !entry.isIntersecting;
  update();
});
document.addEventListener("visibilitychange", update);
```

**Interview Q:** *What's the bug in toggling state from two independent events?* The last event wins, so one condition can overwrite the other. Store the inputs and derive the result (same idea as derived state in React).

### 11. Content in data files, UI in components

All text lives in `src/utils/*.js`; components only render it. Changing copy never touches layout code.

```js
// utils/heroData.js
const heroData = {
  roles: ["Front-End", "Full-Stack", "Applied AI"],
  greeting: "Hey There,",
  name: "Akbar",
  gourdButton: { open: "What's inside? Open me", close: "Back into the gourd" },
  caravan: ["Magic gourds for sale!", "Dreams come true inside!"],
};
```

```js
// utils/gourdSnippets.js: each code line is a list of [text, color] parts
const gourdSnippets = {
  Python: {
    file: "db.py",
    lines: [
      [["# Load projects from PostgreSQL", "c"]],
      [["def", "k"], [" get_projects", "f"], ["(db):", "t"]],
      // …
    ],
  },
};
```

**Interview Q:** *Why separate content from components?* Easier edits, reuse (the same data feeds the home preview and the full page), testability, and it's one step from moving the content to a CMS or an API.

### 12. Rendering a list, and keys in nested lists

```jsx
// components/HeroGourd.jsx: render a code snippet from data
{gourdSnippets[tech].lines.map((parts, i) => (
  <text key={i} x="63" y={325 + i * 17} xmlSpace="preserve">
    {parts.map(([part, color], j) => (
      <tspan key={j} fill={codeColors[color]}>{part}</tspan>
    ))}
  </text>
))}
```

Index keys are fine **here** because the lines never reorder, get inserted or deleted within one snippet (a new snippet remounts the whole group through its `key`). For lists that change (filters, sorting, deleting), use a stable id: `key={item.id}`.

**Interview Q:** *When is `key={index}` a bug?* When items can be reordered, inserted or removed: React matches elements by key, so state (inputs, focus, animations) ends up attached to the wrong item.

### 13. A tiny parser that returns JSX (`**bold**` → `<strong>`)

Data text can't contain JSX, and `dangerouslySetInnerHTML` would open an XSS hole. Split the string and map pieces to elements instead.

```jsx
// utils/withBold.jsx
// "a **b** c" → ["a ", <strong>b</strong>, " c"]: every odd piece was inside the asterisks
const withBold = (text) =>
  text.split("**").map((part, i) =>
    i % 2 ? <strong key={i}>{part}</strong> : part
  );
```

**Interview Q:** *Why not `dangerouslySetInnerHTML`?* It injects raw HTML, so any user-controlled text could run scripts (XSS). Returning React elements keeps React's automatic escaping.

### 14. Form validation with an object of checks, focus the first error

```jsx
// components/Contact.jsx
const checks = {                              // in form order
  user_email: validateEmail,
  subject: validateRequired,
  user_message: validateRequired,
};

const handleSubmit = async (e) => {
  e.preventDefault();
  const fields = formRef.current.elements;    // uncontrolled inputs, read on submit
  const nextErrors = {};
  for (const name in checks) {
    if (!checks[name](fields[name].value)) nextErrors[name] = true;
  }
  setErrors(nextErrors);
  const firstInvalid = Object.keys(nextErrors)[0];
  if (firstInvalid) {
    fields[firstInvalid].focus();             // keyboard and screen-reader users land on the problem
    return;
  }
  // …send
};
```

Each input is wired for screen readers:

```jsx
{
  "aria-invalid": errors[name] || undefined,
  "aria-describedby": errors[name] ? `e-${name}` : undefined,
}
{errors[name] && <small id={`e-${name}`}>{form.errors[name]}</small>}
```

**Interview Q:** *Controlled or uncontrolled inputs here?* Uncontrolled: the values are only needed on submit, so reading `form.elements` avoids a re-render on every keystroke. Controlled inputs are better when the UI reacts to each keystroke (live validation, formatting, dependent fields).

### 15. A honeypot field against spam bots

A hidden field that humans never see or fill. Bots fill every field, so a value means "bot": return silently.

```jsx
// HONEYPOT: only bots fill the hidden field; ignore them silently
if (fields._gotcha.value) return;
```

**Interview Q:** *How do you reduce form spam without a CAPTCHA?* A honeypot field, a minimum time-to-submit, server-side rate limiting, and only then an invisible CAPTCHA (Turnstile, reCAPTCHA v3).

### 16. Accessible custom buttons (`role`, `tabIndex`, `aria-pressed`, Enter/Space)

The tech chips are SVG `<g>` elements, not `<button>`s, so everything a button gives for free must be added by hand.

```jsx
// components/HeroGourd.jsx
<g
  role="button"
  tabIndex={open ? 0 : -1}                    // reachable with Tab only while open
  aria-pressed={tech === pill.label}          // a toggle: screen readers say "pressed"
  aria-label={`Show ${pill.label} code`}
  onClick={() => pick(pill.label)}
  onKeyDown={(e) => {
    if (e.key !== "Enter" && e.key !== " ") return;
    e.preventDefault();                       // Space would otherwise scroll the page
    pick(pill.label);
  }}
>
```

**Interview Q:** *What does a `<div onClick>` lack compared with a `<button>`?* Focusability (`tabIndex`), the role announced to screen readers, keyboard activation (Enter and Space), and disabled semantics. Prefer a real `<button>`; when you can't (inside SVG), add all of them.

### 17. `aria-disabled` vs `disabled`

```jsx
// the open/close button while the gourd animates
<button type="button" onClick={toggle} aria-disabled={busy}
  className="aria-disabled:opacity-55 aria-disabled:cursor-default">
```

| | `disabled` | `aria-disabled="true"` |
|---|---|---|
| Can be focused | No (a focused button loses focus) | Yes |
| Click events | Blocked by the browser | Still fire: guard in the handler |
| Screen readers | "dimmed/unavailable" | "dimmed/unavailable" |

Here the button is only busy for ~4s, and a keyboard user who just pressed it would lose their place with `disabled`. So: `aria-disabled` + `if (busy) return;`.

**Interview Q:** *When would you choose `aria-disabled`?* When the control is temporarily unavailable and focus must stay put, or when you want it discoverable (with a tooltip explaining why it's disabled).

### 18. One component, two layouts: a `preview` prop

```jsx
// components/Experience.jsx
const PREVIEW_COUNT = 4;

const Experience = ({ preview = false }) => {
  const items = preview ? experiences.slice(0, PREVIEW_COUNT) : experiences;
  return (
    <section id="experience" aria-labelledby="xp-h">
      {preview ? (
        <h2 id="xp-h">Experience</h2>
      ) : (
        <h2 id="xp-h" className="sr-only">All experience</h2>   // the page header is the visible title
      )}
      {/* …timeline… */}
      {preview && <MoreLink href="/experience">View full experience</MoreLink>}
    </section>
  );
};

// app/page.js:            <Experience preview />
// app/experience/page.js: <Experience />
```

**Interview Q:** *Prop flag or two components?* A flag is fine while the differences are small (count, heading, a link). If the variants keep diverging, split them and share the inner pieces instead.

### 19. `useLayoutEffect` vs `useEffect`: run before paint

The key difference is timing:
- `useEffect` normally runs **after** the browser has painted the update.
- `useLayoutEffect` runs synchronously **after React updates the DOM but before the browser paints** it.

So use `useLayoutEffect` when the effect changes what the user would see in that very frame (measure layout and reposition, or swap a class), otherwise they see one wrong frame: a flash or a jump. Everything else goes in `useEffect`, which doesn't block painting.

```jsx
// components/PageTransition.jsx: when the new page has rendered, swap "leaving" (old page faded out)
// for "entering" (slide-in) BEFORE the browser paints it; with useEffect the new page could
// flash fully visible for one frame before the slide-in starts
useLayoutEffect(() => {
  if (!leaving.current) return; // first load, or the browser's back/forward: no animation
  leaving.current = false;
  const html = document.documentElement;
  html.classList.remove("leaving");
  html.classList.add("entering");
  const timer = setTimeout(() => html.classList.remove("entering"), ENTER_MS);
  return () => {
    clearTimeout(timer);
    html.classList.remove("entering");
  };
}, [pathname]);
```

**Interview Q:** *When would you pick `useLayoutEffect`, and what's the cost?* When you must read layout or change the DOM before the user sees the frame (tooltips positioned from a measurement, avoiding a flash). The cost: it blocks painting until it finishes, so keep it short. It also doesn't run on the server (React warns in SSR), so only use it for browser-only work.

### 20. Capture-phase listeners: run before another component's handler

DOM events travel **down** from `window` to the target (capture phase), then back **up** (bubble phase). React's `onClick` handlers (like the one inside `next/link`) run in the bubble phase. A listener added with `capture: true` on `window` runs first, so it can `preventDefault()` and take over.

```js
// components/PageTransition.jsx: catch clicks on links to another page before next/link does
const onClick = (e) => {
  const a = e.target.closest?.("a[href]");
  if (!a || e.defaultPrevented || e.button !== 0 || e.metaKey || e.ctrlKey || e.shiftKey || e.altKey) return;
  if (a.target === "_blank" || a.hasAttribute("download")) return;
  const url = new URL(a.href, location.href);
  if (url.origin !== location.origin || url.pathname === location.pathname) return;

  e.preventDefault(); // next/link sees this and doesn't navigate; we do it ourselves
  html.classList.add("leaving");
  router.push(url.pathname + url.search + url.hash); // start loading now, during the slide-out
};
window.addEventListener("click", onClick, true); // true = capture phase
```

- **Event delegation:** one listener for every link on every page, instead of wrapping each `<Link>`.
- Respect the user: modified clicks (Ctrl/Cmd/Shift = new tab/window), middle clicks, `target="_blank"` and downloads are left alone.
- Lesson from measuring on a phone: first I waited for the 0.3s slide-out and *then* called `router.push()`, which added ~0.5s to every page change. Starting the navigation at the click, while the animation plays, brought it down to ~0.2s extra.

**Interview Q:** *What's the difference between `stopPropagation()` and `preventDefault()`?* `preventDefault()` cancels the browser's default action (following the link) but the event keeps travelling, so other handlers still run and can check `e.defaultPrevented`. `stopPropagation()` stops the event reaching other elements' handlers but doesn't cancel the default action.

---

## Next.js (App Router)

### 21. Server Components by default, `"use client"` only where needed

In the App Router every component is a Server Component unless the file starts with `"use client"`. Server Components send no JavaScript to the browser.

In this site only 5 of 14 components are client components, each for a reason:

| Component | Why it needs the client |
|---|---|
| `Navbar` | scroll listener, `usePathname` |
| `Contact` | form state, EmailJS |
| `Dialog` | ref + `showModal()` |
| `HeroGourd` | open/close state, chip clicks, sounds |
| `Caravan` | `IntersectionObserver`, bubble text |

`Hero` stays a server component even though it renders the client `HeroGourd` and `Caravan`: a server component can import and render client components.

**Interview Q:** *What does `"use client"` actually do?* It marks a boundary: that module and everything it imports are bundled for the browser and hydrated. It doesn't mean "render only on the client" (client components still render to HTML on the server first).

### 22. Static export (`output: 'export'`)

```js
// next.config.mjs
const nextConfig = {
  output: "export",              // `next build` writes plain HTML/CSS/JS to out/
  images: { unoptimized: true }, // the image optimizer needs a server, so turn it off
};
```

Each route becomes a file (`out/index.html`, `out/experience.html`), which any static host or CDN can serve.

**Interview Q:** *What do you give up with a static export?* Anything that needs a Node server at request time: API routes, server actions, middleware, ISR/on-demand revalidation, cookies/headers in server components, and the built-in image optimization.

### 23. `next/font`: self-hosted fonts as CSS variables (and variable fonts with `opsz`)

```js
// app/layout.js
// variable font + optical-size axis: one file holds every weight (200–800)
const display = Bricolage_Grotesque({ subsets: ["latin"], axes: ["opsz"], variable: "--nf-display" });
const mono = JetBrains_Mono({ subsets: ["latin"], weight: ["400", "500"], variable: "--nf-mono" });

<html lang="en" className={`${display.variable} ${mono.variable}`}>
```

```css
/* globals.css */
@theme {
  --font-display: var(--nf-display), "Segoe UI", system-ui, sans-serif;
  --font-mono: var(--nf-mono), ui-monospace, Menlo, Consolas, monospace;
}
```

`next/font` downloads the fonts at build time and serves them from your own domain (no request to Google, no privacy issue), preloads them, and generates a size-adjusted fallback font to reduce layout shift.

**Interview Q:** *Why load only the weights you use?* Each weight is a separate file. Note the flip side: if CSS asks for a weight you didn't load (e.g. `font-semibold` = 600 with only 400/500 loaded), the browser fakes it ("synthetic bold"), which looks worse and can change text width.

**Variable fonts.** A variable font is one file with *axes* (weight `wght`, width, optical size `opsz`…) instead of one file per weight. The optical-size axis redraws letters for their size: big headings get a tighter, more condensed cut, small text a looser, more readable one. With `font-optical-sizing: auto` (the default) the browser picks the `opsz` value from the `font-size`, no extra CSS. In my hero the big "Hey There, I'm Akbar." got 8% narrower and matched the design exactly once I loaded `opsz`. In `next/font`, `axes` only works on the variable font, so you leave `weight` out.

Gotcha I hit: with only 600 and 800 loaded, `font-bold` (700) used to *snap* to the nearest loaded file (800), and `font-normal` on a heading showed as 600. The variable font draws every weight exactly, so those headings suddenly looked different. Fix: set each one to the weight it was really showing (`font-extrabold`, `font-semibold`).

**Interview Q:** *Static weights or a variable font?* Few weights (1–2): static files are usually smaller. Many weights, or you want an axis like `opsz` or fine-tuned weights: one variable file is simpler and often smaller in total. Either way, check what weights your CSS actually asks for.

### 24. Metadata API

```js
// app/layout.js
export const metadata = {
  title: "Magic Gourd · Akbar Ruzi",
  description: "Akbar Ruzi, software engineer: front-end, full-stack and applied AI.",
  formatDetection: { telephone: false, address: false, email: false },
};

// app/experience/page.js: a page overrides only what differs
export const metadata = { title: "Experience · Akbar Ruzi" };
```

**Interview Q:** *How do you set per-page `<title>` in the App Router?* Export `metadata` (static) or `generateMetadata` (async, from params/data) from a `layout` or `page`. Next merges them from the root down. It only works in Server Components.

### 25. `next/link` vs a plain `<a>` (and `usePathname`)

```jsx
// components/Navbar.jsx
const isHome = usePathname() === "/";
const Anchor = isHome ? "a" : Link;               // pick the element at runtime
const to = (hash) => (isHome ? hash : `/${hash}`);

<Anchor href={to("#about")}>About</Anchor>
```

- On the home page, `#about` is a same-page jump: a plain `<a>` is enough.
- On `/experience`, the link must go to `/#about`: `Link` swaps pages without a full reload.
- Storing a component in a variable (`const Anchor = …`) and rendering `<Anchor>` works because JSX only needs a capitalized name.

**Interview Q:** *What does `next/link` add over `<a>`?* Client-side navigation (no full reload, layout state kept) and automatic prefetching of the target route when the link is visible.

### 26. Layouts persist across navigations, pages remount

In the App Router, going from `/` to `/experience` swaps only the **page** inside the layout. The **layout** doesn't unmount or re-render, so components in it keep their state, refs, timers and event listeners. Put things there that must survive a page change.

```jsx
// app/layout.js: the layout stays a server component; PageTransition is a small client component inside it
<body>
  {children}
  <PageTransition />
</body>
```

The page transition has two halves: on the old page a click marks "leaving" (a ref), and when the new page renders, an effect on `usePathname()` sees that ref and plays the slide-in. Both halves must run in the **same component instance**. Inside each page it would unmount with the old page and lose the ref, so the new page wouldn't know a transition was in progress. (The plain-HTML design does full reloads, so it has to pass a flag through `sessionStorage` instead.)

**Interview Q:** *Where do you keep UI that must survive navigation (a player, a progress bar, a transition)?* In a layout above the pages that change. Also useful: `template.js` is the opposite, a layout that **does** remount on every navigation.

### 27. Hydration mismatches and `suppressHydrationWarning`

Hydration = React attaching to server-rendered HTML. If the first client render differs from the server HTML, React warns (and may re-render).

Two cases in this project:

```jsx
// 1. Browser extensions / Chrome on iOS add attributes before React loads
<html lang="en" suppressHydrationWarning>          // only this tag's attributes, not its children
<input suppressHydrationWarning />                 // autofill attribute added by Chrome on iOS
```

```css
/* 2. Reduced motion is handled in CSS, not with JS state in the first render.
   The server can't know the user's setting, so CSS decides what is visible from the first paint. */
@media (prefers-reduced-motion: no-preference) {
  .gourd[data-state="closed"] .pop { opacity: 0; }
}
```

**Interview Q:** *How do you render something that depends on the browser (window size, a media query, `localStorage`) without a hydration mismatch?* Render the same thing on server and client first, then update in `useEffect`; or let CSS handle it (media queries); or render that part only on the client (`dynamic(..., { ssr: false })`).

---

## CSS and animation

### 28. Animate only `transform` and `opacity`

The browser renders in steps: style → layout → paint → composite. Changing `width`, `top` or `margin` re-runs layout for the page. `transform` and `opacity` can be handled by the compositor (often the GPU) alone, so they stay smooth at 60fps and cause no layout shift.

```css
@keyframes pop-out {
  0%   { opacity: 0; transform: translate(var(--dx), var(--dy)) scale(0.2); }
  45%  { transform: translate(var(--dx), calc(var(--dy) - 46px)) scale(0.75); }
  100% { opacity: 1; transform: none; }
}
```

**Interview Q:** *Why is animating `left` slower than `transform: translateX`?* `left` triggers layout and paint on every frame; `transform` only composites an already-painted layer.

### 29. CSS custom properties per element: staggered animations

One keyframe rule for all chips; each element passes its own start offset and delay as CSS variables.

```jsx
// components/HeroGourd.jsx
const fly = (dx, dy, d) => ({ "--dx": `${dx}px`, "--dy": `${dy}px`, "--d": `${d}s` });

<g className="pop" style={fly(pill.dx, pill.dy, pill.d)}>…</g>
```

```css
.gourd[data-state="opening"] .pop {
  animation: pop-out 0.75s cubic-bezier(0.2, 0.8, 0.3, 1.15) var(--d) both;
}
```

**Interview Q:** *How do you pass a value from React to CSS?* A CSS custom property in `style` (`style={{ "--x": value }}`), then use `var(--x)` in CSS. Better than inline `transform` strings: the animation logic stays in CSS.

### 30. Playing a keyframe animation in reverse

Closing the gourd reuses the opening keyframes with `animation-direction: reverse`. The delays are mirrored with `calc()`, so the last item out goes back in first.

```css
.gourd[data-state="closing"] .pop {
  animation: pop-out 0.75s cubic-bezier(0.7, -0.15, 0.8, 0.2)
             calc(3.05s - var(--d)) reverse both;
}
```

- `both` (fill-mode) holds the first keyframe before the delay and the last one after the end. With `reverse`, "first" is the 100% frame, so items stay visible until their turn.
- Use a different easing for the way back: an ease-out in reverse would feel like an ease-in.

**Interview Q:** *What does `animation-fill-mode: both` do?* Applies the starting keyframe during the delay (`backwards`) and keeps the end keyframe after the animation (`forwards`).

### 31. `prefers-reduced-motion`

Some users get dizzy or nauseous from motion (vestibular disorders) and turn on "Reduce motion" in their OS. Respect it.

```css
/* opt-in: animations only exist when the user hasn't asked for less motion */
@media (prefers-reduced-motion: no-preference) {
  .walker { animation: cross 34s linear infinite; }
}
@media (prefers-reduced-motion: reduce) {
  html { scroll-behavior: auto; }
  *, *::before, *::after { transition: none !important; }
}
```

```jsx
// Tailwind variants for one-off cases
<button className="motion-reduce:hidden">…</button>
<span className="motion-reduce:animate-none animate-spin" />

// components/About.jsx: the skill pills always turn green on hover,
// but only wiggle when motion is OK (variants stack: motion-safe + hover)
<li className="cursor-default transition-colors hover:bg-cta hover:text-white motion-safe:hover:animate-wiggle">
```

Two ways to write it: `motion-reduce:` takes the animation away (opt-out), `motion-safe:` only adds it when the user hasn't asked for less motion (opt-in). Opt-in is the safer default: if you forget a case, nothing moves. Note the pill keeps its color change with reduced motion: a color fade isn't movement.

In this project reduced motion doesn't remove features: the gourd shows open with its chips and code, and the caravan stands still with its speech bubble.

**Interview Q:** *Remove all animation, or reduce it?* Reduce: keep the content and meaning, drop large movement, parallax, auto-playing loops and smooth scrolling. Short fades are usually fine.

### 32. `transform-box: fill-box` for SVG transforms

In SVG, `transform-origin` is measured from the SVG's own top-left corner by default, so `rotate()` on a small shape swings it around the whole drawing. `transform-box: fill-box` makes percentages refer to the shape's own bounding box.

```css
.gourd .lid {
  transform-box: fill-box;
  transform-origin: 100% 100%;   /* the lid's own bottom-right corner, like a hinge */
}
.walker .leg {
  transform-box: fill-box;
  transform-origin: 50% 0;       /* the hip: legs swing from the top */
}
```

**Interview Q:** *Why does my SVG element rotate around the wrong point?* Its transform origin is in SVG user space (the viewport's top-left). Set `transform-box: fill-box` and a `transform-origin` relative to the element.

### 33. Container query units (`cqw`)

`100vw` is the window width; the caravan walks across a card that is narrower than the window on wide screens. With `container-type: inline-size`, `100cqw` is the width of the element itself.

```css
.caravan { container-type: inline-size; overflow: hidden; }

@keyframes cross {
  from { transform: translateX(-330px); }
  to   { transform: translateX(calc(100cqw + 20px)); }   /* exactly across the strip */
}
```

**Interview Q:** *Media queries vs container queries?* Media queries react to the viewport; container queries react to a parent's size, so a component adapts wherever it's placed (sidebar, grid cell, full width).

### 34. Tailwind v4: `@theme` tokens and `@layer`

```css
@import "tailwindcss";

@theme {                          /* design tokens → utilities like bg-cream, text-cta */
  --color-cream: #F5F2E9;
  --color-cta: #29695C;
  --font-mono: var(--nf-mono), ui-monospace, monospace;
  --animate-twinkle: twinkle 2.2s ease-in-out infinite;   /* → animate-twinkle */
}

@layer components {               /* custom CSS that utilities can still override */
  .card-section { padding-inline: max(var(--inner), calc((100% - 1080px) / 2)); }
}
```

- v4 has no `tailwind.config.js`: configuration lives in CSS.
- An `--animate-*` token can reuse keyframes defined anywhere in the CSS: `--animate-wiggle: wiggle 0.5s ease-in-out;` gives `animate-wiggle` for the About pills from the same `@keyframes wiggle` the gourd chips already use (one animation, two places).
- Tailwind only emits the theme variables and utilities you actually use, so unused tokens cost nothing in the final CSS.
- Cascade layers (`@layer`) decide priority by layer order, not selector specificity, so a utility class always beats a component rule.

**Interview Q:** *Why use cascade layers?* To control priority between groups of CSS (reset → base → components → utilities) without specificity wars or `!important`.

### 35. Sticky nav: `scroll-padding-top` and smooth scrolling

```css
html {
  scroll-behavior: smooth;
  scroll-padding-top: 7rem;   /* #section jumps and Tab focus both stop below the sticky nav */
}
```

`scroll-margin-top` on each section only fixes anchor jumps. `scroll-padding-top` on the scroll container also applies when the browser scrolls a focused element into view, so keyboard users never get a focused link hidden under the nav.

**Interview Q:** *Clicking `#contact` hides the heading under the sticky header. Fix?* `scroll-padding-top` on `html` (or `scroll-margin-top` on the targets), equal to the header height plus some space.

### 36. Scroll-driven animations (`animation-timeline: view()`)

A normal CSS animation runs on a clock (e.g. 0.5s). A scroll-driven animation runs on the **scroll position**: `view()` ties its progress to where the element is in the viewport, so scrolling forward plays it and scrolling back rewinds it. No JavaScript, no scroll listener, and the browser can run it off the main thread.

```css
/* globals.css: each section card fades and slides up as it scrolls into view */
@keyframes reveal {
  from { opacity: 0; transform: translateY(20px); }
}

@supports (animation-timeline: view()) {               /* older browsers: cards just show */
  @media (prefers-reduced-motion: no-preference) {
    .page > main > section {
      animation: reveal ease-out both;
      animation-timeline: view();                       /* after the shorthand, which resets it */
      animation-range: entry 0% entry 240px;            /* done once its top is 240px into the screen */
    }
  }
}
```

- **`animation-range`** picks which part of the element's trip through the viewport plays the animation. `entry` is while it comes in from the bottom; `cover` is its whole trip, from first pixel in to last pixel out.
- Gotcha I hit: the usual `entry 0% cover 30%` depends on the element's height. My Experience card is ~1,600px tall, so it was still half faded with its top almost under the nav. A fixed length (`entry 240px`) finishes at the same point for every card.
- `both` (fill mode) keeps the end state for cards already scrolled past, so jumping down with a nav link never leaves a faded card above.
- Progressive enhancement: `@supports` means unsupported browsers get the plain page, never hidden content.

**Interview Q:** *Scroll-driven animations vs an `IntersectionObserver` reveal?* CSS: zero JS, follows the scroll exactly (also backwards), runs on the compositor; but newer browser support. `IntersectionObserver`: works everywhere and can trigger time-based animations once, but needs JS and care so content isn't hidden if the script fails.

---

## Browser APIs

### 37. Web Audio API: sounds without audio files

The open/close sounds are generated: an oscillator gliding in pitch, plus a burst of filtered noise, shaped by a volume envelope.

```js
// utils/gourdSounds.js
const tone = (c, out, { type, f0, f1, t, peak, attack, decay, glide }) => {
  const osc = c.createOscillator();
  const gain = c.createGain();
  osc.type = type;                                          // "sine", "triangle", …
  osc.frequency.setValueAtTime(f0, t);
  osc.frequency.exponentialRampToValueAtTime(f1, t + glide);
  envelope(gain, t, peak, attack, decay);                   // quick rise, then fade
  osc.connect(gain).connect(out);
  osc.start(t);
  osc.stop(t + attack + decay + 0.05);
};

// one shared AudioContext, created on the first click
const audio = () => {
  if (!ctx) ctx = new (window.AudioContext || window.webkitAudioContext)();
  if (ctx.state === "suspended") ctx.resume();
  return ctx;
};

export const playOpen = () => play(cork, 0.67);   // timed to the lid coming off
```

- **Autoplay policy:** browsers block audio until the user interacts with the page, so the context is created inside a click handler.
- Scheduling with `c.currentTime + delay` is sample-accurate, unlike `setTimeout`.
- `exponentialRampToValueAtTime` can't reach 0, so the envelope ramps to `0.0001`.
- Wrapped in `try/catch`: no sound is fine, a crash in the click handler is not.

**Interview Q:** *Why might audio not play on page load?* Autoplay policies: an `AudioContext` created without a user gesture starts `suspended`. Create or `resume()` it in response to a click or key press.

### 38. Restarting a CSS animation from JavaScript

Setting the same `animation` value again does nothing. Remove it, force the browser to apply that, then put it back.

```js
// components/Caravan.jsx
b.style.animation = "none";
void b.offsetWidth;            // reading layout forces a reflow, so "none" takes effect
b.style.animation = "";        // back to the stylesheet's animation: it starts over
b.style.animationDelay = `${delay}s`;
```

In React, changing the element's `key` (concept 5) does the same thing more declaratively.

**Interview Q:** *Why does `void el.offsetWidth` work?* Style changes are batched until the browser next needs layout. Reading a layout property flushes them, so the browser sees "no animation" before it sees the animation again.

---

## Web performance

### 39. Core Web Vitals: LCP, CLS, INP

| Metric | Measures | Good |
|---|---|---|
| **LCP** (Largest Contentful Paint) | when the biggest element in view is painted (loading) | ≤ 2.5s |
| **CLS** (Cumulative Layout Shift) | how much visible content jumps around (visual stability) | ≤ 0.1 |
| **INP** (Interaction to Next Paint) | how fast the page responds to clicks/taps/keys (responsiveness) | ≤ 200ms |

Lighthouse (lab) can't measure INP without interactions, so it reports **TBT** (Total Blocking Time) as a stand-in.

Ways this project avoids CLS: the gourd sits in an `aspect-ratio` box (space reserved before it draws), the open/close button is part of the layout from the first paint, and every animation uses `transform`/`opacity`.

**Interview Q:** *What causes CLS, and how do you fix it?* Images/embeds without dimensions (set `width`/`height` or `aspect-ratio`), content inserted above existing content (reserve space), and web fonts changing text size (see next concept).

### 40. Layout shift from web fonts

Measured in this project: on phones the hero role pill is set in JetBrains Mono. Until that font loads, the fallback font is wider, so the roles wrap to two lines (pill 50px tall). When the real font arrives they fit on one line (34px), and the whole hero jumps up ~17px: CLS 0.12.

```
fallback font:  [● FRONT-END · FULL-STACK ·]     ← 2 lines, 50px
                [  APPLIED AI             ]
web font:       [● FRONT-END · FULL-STACK · APPLIED AI]   ← 1 line, 34px
```

Fixes:
- A fallback font metrically adjusted to match (`size-adjust`, `ascent-override`; `next/font` does this automatically for many fonts).
- Load every weight the CSS uses (here 600 was faked from 400/500).
- Make the layout not depend on the exact text width (no wrap, or reserved height).
- `font-display: optional` (no swap at all, the fallback is kept if the font is late).

**Interview Q:** *What is FOUT, and how do you prevent the shift it causes?* Flash Of Unstyled Text: the fallback shows first, then swaps to the web font. Prevent the jump with a matched fallback (`size-adjust`), preloading, and self-hosting (`next/font`).

### 41. Measuring before optimizing (Lighthouse, a fair comparison)

To check whether the new animations hurt performance, I compared two commits instead of guessing:

1. Built each commit as a static export in its own `git worktree` (so the running dev server's `.next` wasn't touched).
2. Served both the same way and ran mobile Lighthouse 3 times each, taking the median (single runs vary a lot: one run scored 60, the others 73–74).
3. Result: Performance 71 (without) vs 73 (with): no measurable cost; JS +4 kB.

```bash
git worktree add --detach ../lh/without 83d9250
git worktree add --detach ../lh/with    4e7ed26
npx lighthouse http://localhost:4101/ --form-factor=mobile --only-categories=performance
```

Lessons: compare like with like (same server, same throttling), repeat runs, and read the diagnostics. Both versions had the same slow LCP, mostly because the local server didn't compress files (Lighthouse listed 360 KiB of savings), which real hosting does automatically.

**Interview Q:** *Lab data vs field data?* Lab (Lighthouse) is a controlled, repeatable test on simulated slow hardware, which is good for comparing changes. Field data (CrUX, `web-vitals` in real browsers) shows what real users experience. Google ranks on field data.

### 42. Removing an animation library

The old site used three animation libraries (`framer-motion`, `motion`, `motion-plus`), a 296 KB SVG component with ~89 animated paths, and a full-screen page transition that blocked every navigation for 1–1.5s. The redesign removed all of it; every animation is CSS keyframes. Home page First Load JS is now ~113 kB, including the interactive gourd.

**Interview Q:** *When is a JS animation library worth it?* For gesture-driven or physics-based motion (drag, spring, layout animations between states, shared-element transitions). For entrance effects, loops and hover states, CSS keyframes and transitions are lighter and run off the main thread.
