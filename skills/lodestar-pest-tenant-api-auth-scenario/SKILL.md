---
name: lodestar-pest-tenant-api-auth-scenario
description: "For Lodestar: writing a Pest scenario test that exercises admin user update → API login end-to-end against the real tenant DB, not a stubbed user."
---

# Lodestar: Pest end-to-end test for tenant API auth flow

Use when writing a Pest feature test that needs to:
- Create a tenant with a real in-memory tenant DB
- Create/update a user via the admin panel (POST/PATCH `/admin/tenants/{tenant}/users`)
- Log that user in via the central API (`POST /api/auth/login`) with `X-Tenant` header
- Verify the password the admin set is what the API login accepts

The point: prove the admin write path and the API read path agree on the same tenant DB, without mocking the auth flow.

## Why this pattern

The default `ApiLoginTest` pattern in this repo (see `app/Modules/Central/Tests/Feature/ApiLoginTest.php`) uses `User::factory()->create()` in the central DB and bypasses tenancy middleware via `$middlewareToBypass = [InitializeTenancyIfNotCentral, InitializeTenancyByRequestData, PreventAccessFromCentralDomains]`. That works for testing the auth *logic* in isolation but does NOT exercise the real tenant-DB lookup — it just queries the central DB. For the bug where the admin updates a user in the tenant DB and the API login then queries a different DB, that test cannot catch it.

The scenario pattern below DOES exercise the real tenant DB and the real `X-Tenant` resolution middleware.

## The pattern

1. **Set up operator + tenant via existing helpers** — don't reinvent:

   ```php
   use Tests\Support\AdminTestSchema;
   use Tests\Support\TenantUserTestSchema;

   AdminTestSchema::configure();
   $operator = AdminTestSchema::createAdminUser(['tenant.user.manage', 'tenant.view']);
   $tenant = TenantUserTestSchema::createActiveTenantWithSchema();
   ```

   `createActiveTenantWithSchema()` already calls `Tenant::withoutEvents(...)` to bypass auto-provisioning and seeds the in-memory tenant DB with users/roles/model_has_roles.

2. **Add the additional tables the login flow expects.** `TenantUserTestSchema` only sets up users/roles/model_has_roles. The login flow also hits:
   - `employees`
   - `people_operations_general_configurations`
   - `security_configurations`
   - `login_two_factor_challenges`
   - `two_factor_policy_assignments`
   - `personal_access_tokens`
   - `permissions` / `model_has_permissions` / `role_has_permissions`
   - `users.two_factor_enabled`, `users.two_factor_method`, `users.two_factor_secret`, `users.two_factor_confirmed_at` columns

   Define a helper inside the test file (or in `tests/Support/`) and call it after initializing tenancy:

   ```php
   tenancy()->initialize($tenant);
   try {
       ensureTenantLoginTables(); // helper, idempotent Schema::create + hasTable checks
   } finally {
       tenancy()->end();
   }
   ```

   See `app/Modules/Central/Tests/Feature/ApiLoginAdminScenarioTest.php` for the full helper. Wrap in try/finally so `tenancy()->end()` always runs.

3. **Run the admin actions via the real route** (not the repository directly):

   ```php
   $this->actingAs($operator)
       ->post(route('admin.tenants.users.store', $tenant), [
           'name' => 'User',
           'email' => $email,
           'password' => $initialPassword,
           'roles' => ['admin'],
       ])
       ->assertRedirect(route('admin.tenants.users.index', $tenant));
   ```

4. **Verify the row landed correctly in the tenant DB** by re-initializing tenancy and using `DB::table(...)`. Assert `Hash::check($newPassword, $hash)` is true and `Hash::check($oldPassword, $hash)` is false.

5. **Hit the API with the real X-Tenant middleware** — do NOT bypass it:

   ```php
   $resp = $this->withHeader('X-Tenant', $tenant->id)
       ->postJson('/api/auth/login', [
           'email' => $email,
           'password' => $newPassword,
       ]);

   $resp->assertOk();
   expect($resp->json('success'))->toBeTrue();
   ```

   The status-code-only assertion is intentional: `AuthResponseData::fromSession()` calls `getAllPermissions()` and pulls in any other relations, which can pull in more tables than your test setup has. Status 200 (vs 422 vs 500) is enough to prove "credentials matched AND tenant resolved AND session issued" — which is what the regression test needs.

## Reading the test output

- **422** = tenant resolved, password wrong → admin update and API read disagree
- **500 "Tenant could not be identified"** = X-Tenant missing or unknown tenant
- **500 with `no such table: X`** = test setup missing a table the auth flow needs
- **200** = full happy path

## Don't do

- Don't bypass the X-Tenant middleware for the login step. The whole point of this pattern is to exercise the real tenant-DB lookup.
- Don't hand-roll tenant creation without `Tenant::withoutEvents(...)` — the model observer triggers `TenantProvisionStep` jobs that need `tenant_provision_logs` in the test schema.
- Don't rely on the response shape (`data.user.email` etc.) when the assertion is about the password/tenant behavior. Status code is the load-bearing check.

## Reference file

`app/Modules/Central/Tests/Feature/ApiLoginAdminScenarioTest.php` is the working example.
