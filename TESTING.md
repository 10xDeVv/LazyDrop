# LazyDrop Testing Guide

## Overview

This document covers the testing setup and practices for LazyDrop backend and frontend.

---

## Backend Testing

### Technology Stack
- **Test Framework:** JUnit 5
- **Mocking:** Mockito
- **Assertions:** AssertJ
- **Integration Testing:** TestContainers (PostgreSQL)
- **Code Coverage:** JaCoCo
- **Code Quality:** SonarQube, OWASP Dependency-Check

### Running Tests

#### All Tests
```bash
cd apps/backend
mvn clean test
```

#### Specific Test Class
```bash
mvn test -Dtest=DropSessionServiceTest
```

#### With Coverage Report
```bash
mvn clean test jacoco:report
# Report generated at: target/site/jacoco/index.html
```

#### Integration Tests Only
```bash
mvn test -Dgroups=integration
```

### Test Structure

Tests are organized by module following the main source structure:

```
src/test/java/com/lazydrop/
├── modules/
│   ├── session/
│   │   ├── core/
│   │   │   ├── service/DropSessionServiceTest.java
│   │   │   └── controller/DropSessionControllerTest.java
│   │   ├── file/
│   │   ├── participant/
│   │   └── note/
│   ├── billing/
│   │   ├── service/PaymentWebhookServiceTest.java
│   │   └── controller/PaymentWebhookControllerTest.java
│   ├── subscription/
│   │   └── service/SubscriptionServiceTest.java
│   ├── user/
│   │   └── service/UserServiceTest.java
│   └── storage/
├── common/
└── config/
```

### Test Categories

#### 1. Unit Tests
- Test service layer logic in isolation
- Use `@ExtendWith(MockitoExtension.class)`
- Mock all external dependencies

**Example:**
```java
@ExtendWith(MockitoExtension.class)
class DropSessionServiceTest {
    @Mock
    private DropSessionRepository dropSessionRepository;
    @InjectMocks
    private DropSessionService dropSessionService;

    @Test
    void testCreateDropSession() {
        // Arrange, Act, Assert
    }
}
```

#### 2. Controller Tests
- Test REST endpoint handling
- Use `@WebMvcTest` for isolated testing
- Mock services and dependencies

**Example:**
```java
@WebMvcTest(DropSessionController.class)
class DropSessionControllerTest {
    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private DropSessionService dropSessionService;

    @Test
    void testCreateSession() throws Exception {
        mockMvc.perform(post("/sessions"))
            .andExpect(status().isCreated());
    }
}
```

#### 3. Integration Tests
- Test with real database (TestContainers)
- Verify data persistence and transactions
- Use `@SpringBootTest` with `@Testcontainers`

**Example:**
```java
@SpringBootTest
@Testcontainers
class DropSessionIntegrationTest {
    @Container
    static PostgreSQLContainer<?> postgres = 
        new PostgreSQLContainer<>("postgres:16");

    @Test
    void testSessionCreationWithDB() {
        // Full integration test
    }
}
```

### Best Practices

#### Naming Conventions
```java
// Test class names
[ClassUnderTest]Test.java

// Test method names - Given-When-Then pattern
@Test
void testCreateSessionWhenValidUserThenSessionIsCreated() { }

// Or - Scenario-based naming
@Test
void shouldCreateSessionWithCorrectProperties() { }
```

#### Assertions
Use AssertJ for readable, fluent assertions:

```java
// Good - Fluent and readable
assertThat(session)
    .isNotNull()
    .satisfies(s -> {
        assertThat(s.getCode()).isEqualTo("ABC12345");
        assertThat(s.getStatus()).isEqualTo(DropSessionStatus.OPEN);
    });

// Avoid - Basic JUnit assertions
assertTrue(session != null);
assertEquals("ABC12345", session.getCode());
```

#### Mocking Best Practices
```java
// Arrange - Set up expectations
when(repository.findById(id)).thenReturn(Optional.of(entity));

// Act - Execute the code
Entity result = service.getEntity(id);

// Assert - Verify results
assertThat(result).isEqualTo(entity);

// Verify interactions
verify(repository).findById(id);
verify(repository, times(2)).save(any());
verify(repository, never()).delete(any());
```

### Coverage Goals

- **Overall:** Aim for >80% code coverage
- **Critical paths:** 100% coverage (session creation, billing, security)
- **Utilities:** >90% coverage
- **Controllers:** >85% coverage

### Code Quality Commands

#### SonarQube Analysis
```bash
mvn clean verify sonar:sonar \
  -Dsonar.projectKey=lazydrop-backend \
  -Dsonar.host.url=http://sonarqube:9000 \
  -Dsonar.login=<token>
```

#### Dependency Vulnerability Check
```bash
mvn dependency-check:check
# Report: target/dependency-check-report.html
```

---

## Frontend Testing

### Technology Stack
- **Test Framework:** Vitest
- **Component Testing:** React Testing Library
- **Coverage:** Istanbul (V8)

### Setup

#### Installation
```bash
cd apps/frontend
npm ci
```

#### Configuration Files
- `vitest.config.js` - Test runner configuration
- `src/__tests__/setup.js` - Global test setup and mocks

### Running Tests

#### All Tests
```bash
npm test
```

#### Watch Mode (Auto-rerun on changes)
```bash
npm test -- --watch
```

#### Coverage Report
```bash
npm run test:coverage
# Coverage: coverage/
```

#### UI Mode (Visual test runner)
```bash
npm run test:ui
```

### Test Structure

```
src/
├── components/
│   ├── Toast.jsx
│   └── Toast.test.jsx
├── lib/
│   ├── api.js
│   ├── api.test.js
│   ├── auth-token.js
│   └── auth-token.test.js
├── context/
│   ├── UserContext.js
│   └── UserContext.test.js
└── __tests__/
    └── setup.js
```

### Test Categories

#### 1. Component Tests
Test React components in isolation using React Testing Library:

```javascript
describe('Toast Component', () => {
  it('should render toast message', () => {
    render(
      <Toast
        message="Test message"
        type="info"
        onClose={() => {}}
      />
    );

    expect(screen.getByText('Test message')).toBeInTheDocument();
  });
});
```

#### 2. Hook Tests
Test custom React hooks:

```javascript
describe('useSession', () => {
  it('should fetch session data', async () => {
    const { result } = renderHook(() => useSession('session-id'));
    
    await waitFor(() => {
      expect(result.current.session).toBeDefined();
    });
  });
});
```

#### 3. Utility Tests
Test pure functions and utilities:

```javascript
describe('auth-token module', () => {
  it('should retrieve token from localStorage', () => {
    localStorage.setItem('sb-access-token', 'test_token');
    
    const token = getSupabaseAccessToken();
    
    expect(token).toBe('test_token');
  });
});
```

#### 4. API Tests
Mock API calls and test request/response handling:

```javascript
describe('ApiService', () => {
  it('should handle successful API request', async () => {
    mockFetch.mockResolvedValueOnce({
      ok: true,
      json: async () => ({ id: '123' })
    });

    const result = await apiService.request('/test');

    expect(result.id).toBe('123');
  });
});
```

### Best Practices

#### Testing Library Queries (Priority Order)
```javascript
// 1. Queries most like user interactions
getByRole('button', { name: /submit/i })
getByLabelText('Username')
getByPlaceholderText('Email')

// 2. Semantic Queries
getByText('Welcome')

// 3. Test IDs (last resort)
getByTestId('submit-button')

// Avoid querying by CSS classes or IDs directly
```

#### User Interactions
```javascript
// Simulate user actions like they would interact
fireEvent.click(screen.getByRole('button'))
userEvent.type(screen.getByRole('textbox'), 'hello')
await userEvent.click(screen.getByRole('button'))
```

#### Async Testing
```javascript
// Use waitFor for async operations
await waitFor(() => {
  expect(screen.getByText('Loaded')).toBeInTheDocument();
});

// Use userEvent which is async by default
await userEvent.type(input, 'text');
```

#### Mocking
```javascript
// Mock modules
vi.mock('@/lib/api', () => ({
  ApiService: vi.fn()
}));

// Mock functions
const mockFetch = vi.fn();
global.fetch = mockFetch;

// Clear mocks between tests
vi.clearAllMocks();
```

### Coverage Goals

- **Overall:** Aim for >75% code coverage
- **Components:** >80% coverage
- **Utilities:** >90% coverage
- **API/Hooks:** >85% coverage

---

## CI/CD Integration

### GitHub Actions Workflows

#### Backend Tests
**File:** `.github/workflows/backend-test.yml`

- Runs on push to main/develop or PR
- Executes Maven tests with PostgreSQL container
- Generates code coverage report
- Uploads coverage to Codecov
- Performs SonarQube analysis
- Checks for vulnerable dependencies

#### Frontend Tests
**File:** `.github/workflows/frontend-test.yml`

- Runs on push to main/develop or PR
- Executes Vitest tests
- Runs ESLint
- Performs npm audit

#### Docker Build
**File:** `.github/workflows/docker-build.yml`

- Builds Docker images after tests pass
- Scans images with Trivy for vulnerabilities
- Pushes to Docker registry (if credentials provided)

### Required Secrets

For CI/CD pipelines to work, add these secrets to GitHub:

**Backend:**
- `DATABASE_URL` - Test database URL
- `SUPABASE_URL` - Supabase project URL
- `SUPABASE_ANON_KEY` - Supabase anon key
- `SONAR_HOST_URL` - SonarQube host URL
- `SONAR_LOGIN` - SonarQube token

**Frontend:**
- `NEXT_PUBLIC_SUPABASE_URL` - Supabase project URL
- `NEXT_PUBLIC_SUPABASE_ANON_KEY` - Supabase anon key
- `NEXT_PUBLIC_API_URL` - Backend API URL
- `SNYK_TOKEN` - Snyk token (optional)

**Docker:**
- `DOCKER_USERNAME` - Docker Hub username
- `DOCKER_PASSWORD` - Docker Hub password

---

## Local Development

### Pre-commit Hook

Create `.git/hooks/pre-commit` to run tests before commits:

```bash
#!/bin/bash

echo "Running tests..."

# Backend
cd apps/backend
mvn clean test -q || exit 1

# Frontend
cd ../frontend
npm test -- --run || exit 1

echo "All tests passed!"
```

### Watch Mode

Run tests in watch mode while developing:

**Backend:**
```bash
cd apps/backend
mvn test -DreuseForks=false -Dparallel=methods -DthreadCount=4
```

**Frontend:**
```bash
cd apps/frontend
npm test -- --watch
```

---

## Debugging Tests

### Backend
```bash
# Run single test in debug mode
mvn test -Dtest=DropSessionServiceTest#testCreateDropSession -X

# Run with Java debugger
mvn -Dmaven.surefire.debug test
```

### Frontend
```bash
# Run tests with Node inspector
node --inspect-brk ./node_modules/.bin/vitest run

# Run in UI mode for visual debugging
npm run test:ui
```

---

## Continuous Improvement

### Metrics to Track
- Code coverage percentage (aim for 80%+)
- Test execution time (keep <5 minutes for PRs)
- Flaky test rate (should be <2%)
- Mutation test score (if added)

### Adding New Tests
1. Write test first (TDD approach)
2. Ensure it fails
3. Write minimal code to pass
4. Refactor and improve coverage
5. Run full test suite to verify

### Test Review Checklist
- [ ] Tests are focused and isolated
- [ ] No test dependencies (order-independent)
- [ ] Meaningful assertions and messages
- [ ] Proper cleanup (mocks cleared)
- [ ] Performance acceptable (<1s per test)
- [ ] Coverage meets minimum threshold
