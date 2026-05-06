# Testing Standards

## Test Philosophy
Tests assert **what the user sees and does**, not DOM existence.
A test that passes when the component is broken is worse than no
test at all. A test that survives a meaningful refactor without
modification is a good test.

## Conformance Target
- Unit/integration tests: Vitest + React Testing Library
- E2E tests: Playwright
- Coverage target: ≥ 90% branch coverage on domain/business logic
- Component coverage target: every interactive path exercised

## TDD Discipline

1. Write the failing test
2. Run it — confirm the **right** failure (not a typo or
   import error). If all tests pass before the component
   exists, tests aren't asserting correctly.
3. Write the minimum implementation to make the test pass
4. Run the tests — confirm green
5. Commit (test + impl together if test was scaffold-only;
   separate commits if test exposes a real behavior gap)

## Assertion Hierarchy (priority order)

When writing a test assertion, prefer the highest-priority
form that fits:

1. **Behavior** — what callbacks fire, with what arguments,
   how often. The strongest tests.
   ```ts
   expect(onSubmit).toHaveBeenCalledOnce();
   expect(onLog).toHaveBeenCalledWith({ amountOz: 5, ... });
   ```

2. **User-visible content** — text, accessible names, labels.
   ```ts
   expect(screen.getByRole("button")).toHaveTextContent("Save");
   expect(button).toHaveAccessibleName("Sign in with Google");
   ```

3. **Visibility** — when you genuinely need to assert "the
   user can perceive this." Stronger than presence; also
   checks `display`, `visibility`, `opacity`, `hidden`,
   ancestor visibility.
   ```ts
   expect(screen.getByText("Welcome")).toBeVisible();
   ```

4. **Attribute / value** — when the specific DOM state matters.
   ```ts
   expect(input).toHaveValue("07:00");
   expect(button).toHaveAttribute("aria-pressed", "true");
   ```

5. **Absence** — when an element should NOT exist.
   ```ts
   expect(screen.queryByRole("dialog")).toBeNull();
   ```

## Banned Assertions

### ❌ `.toBeInTheDocument()`
**Never use it.** Reasons:
- It's tautological: `screen.getByRole(...)` already throws a
  descriptive error if the element doesn't exist, so wrapping
  it in `expect(...).toBeInTheDocument()` adds zero information.
  The test would have failed before reaching the assertion.
- It tests the wrong thing: presence in the DOM ≠ visible to
  the user. An element with `display: none` is "in the document"
  but invisible. `.toBeVisible()` catches both presence AND
  user-perceptibility.
- It encourages weak tests that don't catch regressions.

If you find yourself wanting to assert that something is "there,"
ask: **what about it matters to the user?** Then assert THAT.

```ts
// ❌ Bad — tells you nothing
expect(screen.getByText("Bottle 1")).toBeInTheDocument();

// ✅ Good — let the query be the presence check
screen.getByText("Bottle 1");

// ✅ Better — verify what the user actually sees
expect(screen.getByRole("article", { name: /next bottle/i }))
  .toHaveTextContent("Bottle 1");

// ✅ Best — verify the behavior this triggers
await userEvent.click(screen.getByRole("button", { name: /log/i }));
expect(onLog).toHaveBeenCalledWith({ label: "Bottle 1", ... });
```

### ❌ Mocking internal modules
Mock at the **boundary** (network, repository, hook), not
internal helpers. Mocking internal modules tests your mocks,
not your code.

```ts
// ❌ Bad
vi.mock("./formatLabel", () => ({ formatLabel: () => "Mocked" }));

// ✅ Good
vi.mock("@/repositories/events", () => ({
  watchEvents: vi.fn(),
}));
```

### ❌ `screen.debug()` / `console.log` in committed tests
Useful while debugging, never in committed code.

## Test Quality Verification

Every `it`/`test` block must contain at least one `expect()` call.
A test case with no assertions silently passes regardless of
behavior — worse than no test.

## E2E Test Conventions (Playwright)

Playwright queries do NOT throw on miss, so explicit visibility
assertions ARE idiomatic:

```ts
await expect(page.getByRole("heading", { name: /dashboard/i }))
  .toBeVisible();
```

Other E2E rules:
- E2E tests are **immutable from the component's perspective** —
  if a test fails, fix the component, not the test (unless the
  test itself is wrong)
- Test golden paths, not exhaustive permutations (that's the
  unit/integration test layer's job)
- Use real selectors (`getByRole`, `getByLabel`), never test IDs
  in the DOM unless absolutely necessary

## When in Doubt

Ask: **"if this test passes, what behavior have I actually
verified?"** If the answer is "it didn't crash on render,"
write a better assertion.
