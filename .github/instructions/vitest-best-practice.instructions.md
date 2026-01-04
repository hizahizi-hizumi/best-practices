---
description: 'Best practices and recommended patterns for writing test code using Vitest'
applyTo: '**/*.test.ts, **/*.spec.ts, **/*.test.tsx, **/*.spec.tsx, **/*.test.js, **/*.spec.js, **/*.test.jsx, **/*.spec.jsx, **/tests/**/*.ts, **/tests/**/*.tsx, **/tests/**/*.js, **/tests/**/*.jsx, **/test/**/*.ts, **/test/**/*.tsx, **/test/**/*.js, **/test/**/*.jsx'
---

# Vitest Best Practices

Defines recommended patterns and conventions for writing test code using Vitest.

## Scope and Context

- Framework: Vitest 1.0 or later
- Environment: Node.js / jsdom / happy-dom
- Leverages compatibility with Jest API

## File Organization

### Naming Conventions

- Use `.test.{ts,tsx,js,jsx}` or `.spec.{ts,tsx,js,jsx}`
- Place in the same directory as the test target or under `tests/`

### Test Grouping

- Group related tests with `describe`
- Clarify functionality and method units with hierarchical structure

```typescript
// Recommended: Hierarchical grouping
describe('Calculator', () => {
  describe('add', () => {
    test('adds positive numbers', () => {
      expect(add(1, 2)).toBe(3)
    })
  })
})

// Not Recommended: Flat tests
test('add positive numbers', () => {
  expect(add(1, 2)).toBe(3)
})
```

### Parameterized Tests

- Use `test.each` for testing the same logic with different data

```typescript
// Recommended
test.each([
  [1, 1, 2],
  [1, 2, 3],
  [-1, 1, 0],
])('add(%i, %i) -> %i', (a, b, expected) => {
  expect(a + b).toBe(expected)
})

// Not Recommended: Duplicate tests
test('add(1, 1)', () => expect(1 + 1).toBe(2))
test('add(1, 2)', () => expect(1 + 2).toBe(3))
```

## Writing Tests

### Test Names

- Clearly describe the test target and expected behavior
- Include conditions and results (e.g., `returns null when user does not exist`)

```typescript
// Recommended: Describe specific behavior
test('returns null when user does not exist', () => {
  expect(getUser('unknown')).toBeNull()
})

// Not Recommended: Ambiguous name
test('getUser', () => {
  expect(getUser('unknown')).toBeNull()
})
```

### AAA Pattern

- Structure tests into 3 sections with Arrange-Act-Assert
- Separate each section with blank lines

```typescript
test('can create user', () => {
  // Arrange
  const userData = { name: 'John', age: 30 }
  // Act
  const user = createUser(userData)
  // Assert
  expect(user.name).toBe('John')
})
```

### Single Responsibility Principle

- Test only one concept per test
- Multiple assertions are acceptable only when related

```typescript
// Recommended: Focused test
test('sets user name correctly', () => {
  expect(createUser({ name: 'John' }).name).toBe('John')
})

// Not Recommended: Multiple independent validations
test('creates user correctly', () => {
  const user = createUser({ name: 'John', age: 30 })
  expect(user.name).toBe('John')
  expect(user.age).toBe(30)
  expect(user.isActive).toBe(true)
  expect(user.createdAt).toBeDefined()
})
```

### test.skip and test.only

- Use `test.skip` to temporarily skip tests for unimplemented features
- Use `test.only` only during debugging
- Never commit `test.only`

```typescript
test.skip('unimplemented feature', () => { /* TODO */ })
test.only('debugging', () => { /* local only */ })
```

## Assertions

### Matcher Selection

- Use matchers that provide the clearest error messages
- Prefer purpose-specific matchers

```typescript
// Recommended: Appropriate matchers
expect(value).toBe(true)          // Primitives
expect(obj).toEqual(expected)     // Objects/arrays
expect(arr).toContain(item)       // Array elements
expect(fn).toThrow(Error)         // Exceptions
expect(num).toBeGreaterThan(0)    // Numeric comparisons

// Not Recommended: Verbose or unclear
expect(value === true).toBe(true)
expect(JSON.stringify(obj)).toBe(JSON.stringify(expected))
```

### Custom Messages and Soft Assertions

- Add custom error messages for complex tests
- Use `expect.soft` for bulk validation of multiple related assertions

```typescript
// Custom message
expect(
  calculateDiscount(price, percentage),
  `Discount calculation error: price=${price}, percentage=${percentage}`
).toBe(expectedDiscount)

// Soft assertions
test('validate multiple properties', () => {
  expect.soft(user.name).toBe('John')
  expect.soft(user.age).toBe(30)
})
```

## Mocking

### Mock Functions

- Create mock functions with `vi.fn()`
- Spy on existing methods with `vi.spyOn()`
- Mock entire modules with `vi.mock()`

```typescript
// Mock function
const callback = vi.fn()
expect(callback).toHaveBeenCalledTimes(3)

// Spy
const spy = vi.spyOn(utils, 'formatDate')
spy.mockRestore()

// Module mock
vi.mock('./api', () => ({
  fetchUser: vi.fn(() => Promise.resolve({ id: '1' }))
}))
```

### Mock Cleanup

- Enable `clearMocks` and `restoreMocks` in config file
- Prevent state leakage between tests

```typescript
// vitest.config.ts (recommended)
export default defineConfig({
  test: {
    clearMocks: true,
    restoreMocks: true,
  },
})

// Or in each test file
beforeEach(() => vi.clearAllMocks())
afterEach(() => vi.restoreAllMocks())
```

## Async Tests

### async/await

- Always use `async/await` for async tests
- Do not use Promise chains
- Always add `await` to `resolves` / `rejects` matchers

```typescript
// Recommended
test('fetch async data', async () => {
  const data = await fetchData()
  expect(data).toBeDefined()
})

test('Promise resolves', async () => {
  await expect(fetchUser('123')).resolves.toEqual({ id: '123' })
})

// Not Recommended: without await
test('async', () => {
  fetchData().then(data => expect(data).toBeDefined())
})
```

### Assertion Count Guarantee

- Use `expect.assertions()` for assertions inside callbacks

```typescript
test('validate inside callback', async () => {
  expect.assertions(2)
  await processAsync(data => {
    expect(data).toBeTruthy()
    expect(data.id).toBeDefined()
  })
})
```

## Snapshot Testing

### Basic Snapshots

```typescript
// ✅ Good example - Small, focused snapshot
test('formats user info correctly', () => {
  const formatted = formatUser({ name: 'John', age: 30 })
  expect(formatted).toMatchSnapshot()
})
```

### Inline Snapshots

Use inline snapshots for small snapshots.

```typescript
// ✅ Good example
test('formats data correctly', () => {
  expect(formatData({ name: 'John' })).toMatchInlineSnapshot(`
    {
      "name": "John",
    }
  `)
})
```

### Snapshot Best Practices

- Keep snapshots small and focused
- Don't snapshot frequently changing parts
- Exclude or mock dynamic values (dates, IDs, etc.)

```typescript
// ✅ Good example - Exclude dynamic values
test('create user', () => {
  const user = createUser({ name: 'John' })
  
  expect(user).toMatchSnapshot({
    id: expect.any(String),
    createdAt: expect.any(Date),
  })
})

// ❌ Avoid - Contains dynamic values
test('create user', () => {
  const user = createUser({ name: 'John' })
  expect(user).toMatchSnapshot() // id and createdAt change every time
})
```

## Setup and Cleanup

### Lifecycle Hooks

- `beforeEach` / `afterEach`: Run before/after each test
- `beforeAll` / `afterAll`: Run once before/after all tests
- `onTestFinished`: Automatic cleanup on test completion

```typescript
// Per-test setup
beforeEach(async () => {
  db = await createTestDatabase()
})
afterEach(async () => {
  await db.close()
})

// Global setup
beforeAll(async () => {
  server = await startTestServer()
})
afterAll(async () => {
  await server.close()
})

// Automatic cleanup
test('test', async () => {
  const db = connectDatabase()
  onTestFinished(() => db.close())
})
```

## UI Component Testing

### Testing Library Integration

- Use with React Testing Library / Vue Testing Library
- Prefer `screen.getByRole()` for accessible element selection
- Use `userEvent` for user interactions

```typescript
// React
import { render, screen } from '@testing-library/react'
import userEvent from '@testing-library/user-event'

test('button click', async () => {
  const handleClick = vi.fn()
  render(<Button onClick={handleClick} />)
  await userEvent.setup().click(screen.getByRole('button'))
  expect(handleClick).toHaveBeenCalledTimes(1)
})

// Vue
import { render } from '@testing-library/vue'

test('component', () => {
  const { getByText } = render(MyComponent, { props: { message: 'Hello' } })
  expect(getByText('Hello')).toBeTruthy()
})
```

## Anti-Patterns

### Patterns to Avoid

- **Omitting await**: Always add `await` to `resolves` / `rejects`
- **Committing test.only**: Never commit it
- **Global state**: Reset state in `beforeEach`
- **Test order dependency**: Make each test independent

```typescript
// Not Recommended: Shared state
let counter = 0
test('test 1', () => counter++)
test('test 2', () => expect(counter).toBe(1)) // fails

// Recommended: Independent tests
describe('tests', () => {
  let counter: number
  beforeEach(() => { counter = 0 })
  test('test 1', () => { counter++; expect(counter).toBe(1) })
  test('test 2', () => { counter++; expect(counter).toBe(1) })
})
```

## Performance

### Parallel Execution and Timeouts

- Run independent tests in parallel with `test.concurrent`
- Use `test.sequential` for order-dependent tests
- Set appropriate timeouts for long-running tests

```typescript
// Parallel execution
test.concurrent('parallel 1', async () => await asyncOp())
test.concurrent('parallel 2', async () => await asyncOp())

// Sequential execution
test.sequential('sequential 1', async () => await sequentialOp())

// Timeout
test('long-running', async () => await longOp(), 30000)
```

## Configuration

### Recommended Configuration

- Use `globals: true` to eliminate imports
- Enable `clearMocks` and `restoreMocks`
- Set coverage thresholds (80% recommended)
- Specify environment (`node` / `jsdom` / `happy-dom`)

```typescript
// vitest.config.ts
export default defineConfig({
  test: {
    globals: true,
    environment: 'node',
    setupFiles: ['./tests/setup.ts'],
    clearMocks: true,
    restoreMocks: true,
    testTimeout: 10000,
    coverage: {
      provider: 'v8',
      reporter: ['text', 'html'],
      thresholds: { branches: 80, functions: 80, lines: 80, statements: 80 },
    },
  },
})

// tsconfig.json
{ "compilerOptions": { "types": ["vitest/globals"] } }
```

## Checklist

### Pre-Commit Verification

- [ ] Test names are clear and specific
- [ ] Each test can run independently
- [ ] `await` is used for async operations
- [ ] `test.only` / `test.skip` removed
- [ ] Mocks are properly cleaned up
- [ ] Snapshots don't contain dynamic values
- [ ] Appropriate matchers are used
- [ ] Error cases are tested

## Priorities

1. User's explicit instructions take highest priority
2. Test independence and cleanup
3. Proper handling of async operations
4. Clear test names and appropriate matchers
