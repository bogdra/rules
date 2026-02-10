# ========================================
# DEVELOPMENT WORKFLOW
# ========================================

---
description: "Clean plugin boundaries - separate routing from business logic"
alwaysApply: true
---

Keep route setup separate from business logic for better maintainability.

**Example:**
```php
// Good: Clean separation
// In service/router.php or service.ts
$router->post('/users', [UserController::class, 'create']);
$router->get('/users/{id}', [UserController::class, 'show']);

// In UserController.php - only HTTP concerns
class UserController {
    public function __construct(
        private UserService $userService,  // Business logic in separate module
    ) {}

    public function create(Request $request): Response {
        $user = $this->userService->createUser($request->getParsedBody());
        return new Response(['id' => $user->getId()], 201);
    }
}

// In UserService.php - business logic module
class UserService {
    public function createUser(array $userData): User {
        // All business logic here, not in controller
    }
}
```

---
description: "Required commit message format with task IDs"
alwaysApply: true
---

All commits must include task ID in uppercase with descriptive present-tense message.

**Format:** `[{TASK-ID}] - {descriptive message}`

**Examples:**
```
[EOSMKP-170113] - Add site module off switch for ShippingTaxProgressBar functionality
[EOSMKP-123456] - Fix user authentication timeout handling
[EOSMKP-789012] - Update product listing pagination to support infinite scroll
```

**Rules:**
- Task ID must be uppercase (EOSMKP-170113, not eosmkp-170113)
- Use present tense ("Add", "Fix", "Update" not "Added", "Fixed", "Updated")
- Be specific about what changes ("Add user validation" not "Update user stuff")

---
description: "Planning and review process for code changes"
alwaysApply: true
---

Always outline implementation plan before making changes.

**Plan must include:**
- What specific files/modules will be modified and why
- Potential breaking changes or side effects
- Dependencies that need to be updated
- Testing strategy for the changes

**Example planning comment:**
```
Plan for adding email notifications:
- Modify: UserService.php (add notification trigger)
- Add: EmailNotificationService.php (new service)
- Update: config/services.yaml (register new service)
- Breaking changes: None expected
- Dependencies: Add symfony/mailer package
- Testing: Unit tests for EmailNotificationService, integration test for registration flow
```

---
description: "Error handling approach - outline problems instead of multiple fix attempts"
alwaysApply: false
---

When encountering issues, present clear problem analysis and solution options rather than attempting multiple fixes.

**Example response pattern:**
```
Problem identified: Authentication middleware fails on API routes but works on web routes.

Root cause analysis:
- API routes use different middleware stack
- JWT token validation missing proper error handling
- Session-based auth conflicts with stateless API design

Possible solutions:
1. Split authentication into separate services (recommended)
2. Add API-specific middleware with proper JWT handling
3. Modify existing middleware to detect request type

Please confirm which approach you'd prefer before implementation.
```
