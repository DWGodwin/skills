# Validate: Project Health Check

Run comprehensive validation of the project.

Execute the following commands in sequence and report results:

## 1. Linting

```bash
npm run lint
```

**Expected:** Clean output or "All checks passed!"

## 2. Type Checking & Build

```bash
npm run build:development
```

**Expected:** Successful compilation and contract generation.

## 3. Unit & Functional Tests

```bash
npm run test:functional:development
```

**Expected:** All service and utility tests pass.

## 4. System Tests (Optional)

```bash
npm run test:system:development
```

**Expected:** Platform-level integration tests pass.

## 5. Summary Report

After all validations complete, provide a summary report with:

- Linting status
- Build/Type-check status
- Tests passed/failed
- Overall health assessment (PASS/FAIL)

**Format the report clearly with sections and status indicators.**
