---
name: velents-testing
description: Testing patterns - PHPUnit with MultiTenancyTestCase, HTTP faking, service mocking, tenant isolation
user-invocable: false
---

# VelentsAI Testing Patterns

All feature tests extend `MultiTenancyTestCase`, which handles tenant creation, database migration, role seeding, external service mocking, and cleanup. Tests use PHPUnit attributes for grouping and the `staff` guard for authentication.

## MultiTenancyTestCase Base Class (Full)

```php
<?php

namespace Tests;

use App\Core\Models\Tenant;
use App\Integration\Services\Elevenlabs;
use App\Integration\Services\TextAgent;
use App\Integration\Services\VoiceAgent;
use App\Integration\Services\VelentsIntegrationsVelentsAi;
use Illuminate\Foundation\Testing\DatabaseMigrations;
use Illuminate\Support\Str;
use Illuminate\Support\Facades\Http;
use Stancl\Tenancy\Database\Models\Domain;
use Mockery;

abstract class MultiTenancyTestCase extends TestCase
{
    use DatabaseMigrations;

    protected array $createdTenants = [];
    protected string $testRunId;
    protected ?string $currentTenantDomain = null;

    protected function setUp(): void
    {
        parent::setUp();

        $this->testRunId = Str::random(8);
        Http::preventStrayRequests();
        Http::fake([
            // Elevenlabs API
            'api.eu.residency.elevenlabs.io/*' => Http::response([
                'agent_id' => 'test-agent-id',
                'success' => true
            ], 200),
            'api.elevenlabs.io/*' => Http::response([
                'voice_id' => 'test-voice-id',
                'success' => true
            ], 200),

            // Text/Voice Agent APIs (internal k8s services)
            'text-agent*.svc.cluster.local/*' => Http::response([
                'success' => true,
                'conversations' => []
            ], 200),
            'voice-agent*.svc.cluster.local/*' => Http::response([
                'success' => true,
                'conversations' => []
            ], 200),

            // Velents Integrations
            'velents-integrations.velents.ai/*' => Http::response([
                'account_id' => 'test-account',
                'success' => true
            ], 200),

            // Catch-all
            '*' => Http::response(['success' => true], 200)
        ]);

        $this->mockExternalServices();
    }

    protected function assertNoRealApiCalls(): void
    {
        $this->assertTrue(true, 'No real API calls were made');
    }

    /**
     * Mock all external service integrations using Mockery.
     * Each service is replaced in the container so no real HTTP calls are made.
     */
    protected function mockExternalServices(): void
    {
        // Mock Elevenlabs
        $elevenlabsMock = Mockery::mock(Elevenlabs::class);
        $elevenlabsMock->shouldReceive('knowledge')->andReturn(true);
        $elevenlabsMock->shouldReceive('knowledge_delete')->andReturn(true);
        $elevenlabsMock->shouldReceive('SyncAgent')->andReturn(true);
        $elevenlabsMock->shouldReceive('CreateVoice')->andReturn(['voice_id' => 'test-voice-id']);
        $elevenlabsMock->shouldIgnoreMissing();
        $this->app->instance(Elevenlabs::class, $elevenlabsMock);

        // Mock TextAgent
        $textAgentMock = Mockery::mock(TextAgent::class);
        $textAgentMock->shouldReceive('knowledge')->andReturn(true);
        $textAgentMock->shouldReceive('knowledge_delete')->andReturn(true);
        $textAgentMock->shouldIgnoreMissing();
        $this->app->instance(TextAgent::class, $textAgentMock);

        // Mock VoiceAgent
        $voiceAgentMock = Mockery::mock(VoiceAgent::class);
        $voiceAgentMock->shouldReceive('knowledge')->andReturn(true);
        $voiceAgentMock->shouldReceive('knowledge_delete')->andReturn(true);
        $voiceAgentMock->shouldIgnoreMissing();
        $this->app->instance(VoiceAgent::class, $voiceAgentMock);

        // Mock VelentsIntegrationsVelentsAi
        $velentsIntegrationsMock = Mockery::mock(VelentsIntegrationsVelentsAi::class);
        $velentsIntegrationsMock->shouldReceive('CreateAccounts')->andReturn(true);
        $velentsIntegrationsMock->shouldReceive('CreatePaymentAccount')->andReturn(
            new \Illuminate\Http\Client\Response(
                new \GuzzleHttp\Psr7\Response(200, [], json_encode(['account_id' => 'test-account']))
            )
        );
        $velentsIntegrationsMock->shouldIgnoreMissing();
        $this->app->instance(VelentsIntegrationsVelentsAi::class, $velentsIntegrationsMock);
    }

    /**
     * Override JSON request to prefix tenant domain for proper routing.
     */
    public function json($method, $uri, array $data = [], array $headers = [], $options = 0)
    {
        if ($this->currentTenantDomain) {
            $uri = 'http://' . $this->currentTenantDomain . $uri;
        }

        return parent::json($method, $uri, $data, $headers, $options);
    }

    protected function tearDown(): void
    {
        Mockery::close();

        if (tenancy()->initialized) {
            tenancy()->end();
        }

        $this->currentTenantDomain = null;

        // Clean up created tenants (databases + domains + records)
        foreach ($this->createdTenants as $tenant) {
            try {
                $tenant->database()->manager()->deleteDatabase($tenant);
            } catch (\Throwable $e) {}

            try {
                $tenant->domains()->delete();
            } catch (\Throwable $e) {}

            try {
                $tenant->delete();
            } catch (\Throwable $e) {}
        }

        $this->createdTenants = [];

        parent::tearDown();
    }

    /**
     * Create a tenant with unique ID and domain.
     */
    protected function createTenant(string $baseName = 'tenant'): Tenant
    {
        $tenantId = "{$baseName}-{$this->testRunId}-" . count($this->createdTenants);

        $tenant = Tenant::create([
            'id' => $tenantId
        ]);

        $domain = "{$tenantId}.test";
        $tenant->domains()->create(['domain' => $domain]);

        $this->createdTenants[] = $tenant;

        return $tenant;
    }

    /**
     * Create multiple tenants.
     */
    protected function createTenants(int $count): array
    {
        $tenants = [];

        for ($i = 0; $i < $count; $i++) {
            $tenants[] = $this->createTenant("tenant-{$i}");
        }

        return $tenants;
    }

    /**
     * Run tenant migrations for the current tenant context.
     */
    protected function runTenantMigrations(): void
    {
        \Illuminate\Support\Facades\Artisan::call('migrate', [
            '--path' => 'database/migrations/tenant',
            '--realpath' => true,
            '--force' => true,
        ]);
    }

    /**
     * Initialize tenant: set context, run migrations, seed roles, set domain for HTTP.
     */
    protected function initializeTenant(Tenant $tenant): void
    {
        tenancy()->initialize($tenant);
        $this->runTenantMigrations();
        $this->seedRolesAndPermissions();

        $domain = $tenant->domains()->first();
        if ($domain) {
            $this->currentTenantDomain = $domain->domain;
        }
    }

    /**
     * Seed roles and permissions for the current tenant.
     */
    protected function seedRolesAndPermissions(): void
    {
        $seeder = new \Database\Seeders\Tenant\RolesAndPermissionsSeeder();
        $seeder->run();
    }

    /**
     * Create staff in a specific tenant context, preserving the current context.
     */
    protected function createTenantStaff(Tenant $tenant, array $attributes = []): \App\Management\Models\Staff
    {
        $wasInitialized = tenancy()->initialized;
        $previousTenant = tenancy()->tenant;

        if (!tenancy()->initialized || tenancy()->tenant->id !== $tenant->id) {
            $this->initializeTenant($tenant);
        }

        $staff = \App\Management\Models\Staff::factory()->create($attributes);

        if (!$wasInitialized) {
            tenancy()->end();
        } elseif ($previousTenant && $previousTenant->id !== $tenant->id) {
            tenancy()->end();
            tenancy()->initialize($previousTenant);
        }

        return $staff;
    }

    /**
     * Create agent in a specific tenant context, preserving the current context.
     */
    protected function createTenantAgent(Tenant $tenant, array $attributes = []): \App\Agents\Models\Agent
    {
        $wasInitialized = tenancy()->initialized;
        $previousTenant = tenancy()->tenant;

        if (!tenancy()->initialized || tenancy()->tenant->id !== $tenant->id) {
            $this->initializeTenant($tenant);
        }

        if (!isset($attributes['created_by_id'])) {
            $staff = \App\Management\Models\Staff::factory()->create();
            $attributes['created_by_id'] = $staff->id;
        }

        $agent = \App\Agents\Models\Agent::factory()->create($attributes);

        if (!$wasInitialized) {
            tenancy()->end();
        } elseif ($previousTenant && $previousTenant->id !== $tenant->id) {
            tenancy()->end();
            tenancy()->initialize($previousTenant);
        }

        return $agent;
    }
}
```

## Feature Test Example: TeamManagementApiTest

Shows the complete pattern for writing feature tests -- setup with tenant/staff/roles, PHPUnit attributes, auth via `actingAs`, JSON assertions, validation tests, Arabic support tests.

```php
<?php

namespace Tests\Feature\Settings;

use Tests\MultiTenancyTestCase;
use App\Management\Models\Staff;
use Illuminate\Support\Facades\Mail;
use App\Management\Mail\Staff\Create as StaffCreateMail;
use PHPUnit\Framework\Attributes\Test;
use PHPUnit\Framework\Attributes\Group;

class TeamManagementApiTest extends MultiTenancyTestCase
{
    protected Staff $staff;
    protected string $baseUrl = '/Staff';
    protected string $invitationUrl = '/Invitation';

    protected function setUp(): void
    {
        parent::setUp();
        $tenant = $this->createTenant('team-management');
        $this->initializeTenant($tenant);
        $this->staff = Staff::factory()->create();
        $this->staff->assignRole('Owner');
        Mail::fake();
    }

    // =========================================================================
    // INDEX - List Team Members
    // =========================================================================

    #[Test]
    #[Group('settings')]
    #[Group('team')]
    public function test_can_list_team_members(): void
    {
        Staff::factory()->count(5)->create();

        $response = $this->actingAs($this->staff, 'staff')
            ->getJson($this->baseUrl);

        $response->assertOk()
            ->assertJsonStructure([
                'data' => [
                    '*' => ['id', 'name', 'email'],
                ],
            ]);

        $this->assertGreaterThanOrEqual(5, count($response->json('data')));
    }

    #[Test]
    #[Group('settings')]
    #[Group('team')]
    public function test_list_team_members_with_pagination(): void
    {
        Staff::factory()->count(20)->create();

        $response = $this->actingAs($this->staff, 'staff')
            ->getJson($this->baseUrl . '?first=10&page=1');

        $response->assertOk();
        $this->assertCount(10, $response->json('data'));
    }

    #[Test]
    #[Group('settings')]
    #[Group('team')]
    public function test_list_team_members_requires_authentication(): void
    {
        $response = $this->getJson($this->baseUrl);

        $response->assertUnauthorized();
    }

    // =========================================================================
    // STORE - Invite Team Member (via POST /Invitation)
    // =========================================================================

    #[Test]
    #[Group('settings')]
    #[Group('team')]
    public function test_can_invite_team_member(): void
    {
        $response = $this->actingAs($this->staff, 'staff')
            ->postJson($this->invitationUrl, [
                'first_name' => 'New',
                'last_name' => 'Team Member',
                'email' => 'newmember@example.com',
                'phone' => '+1234567890',
                'Landing_url' => 'https://app.example.com/invite',
                'role' => 'Admin',
            ]);

        $response->assertSuccessful()
            ->assertJsonPath('data.name', 'New Team Member')
            ->assertJsonPath('data.email', 'newmember@example.com');

        $this->assertDatabaseHas('invitations', [
            'email' => 'newmember@example.com',
            'first_name' => 'New',
            'last_name' => 'Team Member',
        ]);
    }

    #[Test]
    #[Group('settings')]
    #[Group('team')]
    public function test_inviting_team_member_sends_email(): void
    {
        $this->actingAs($this->staff, 'staff')
            ->postJson($this->invitationUrl, [
                'first_name' => 'Email',
                'last_name' => 'Test Member',
                'email' => 'emailtest@example.com',
                'phone' => '+1234567890',
                'Landing_url' => 'https://app.example.com/invite',
                'role' => 'Admin',
            ]);

        Mail::assertSent(StaffCreateMail::class);
    }

    // =========================================================================
    // VALIDATION
    // =========================================================================

    #[Test]
    #[Group('settings')]
    #[Group('team')]
    public function test_invite_requires_first_name(): void
    {
        $response = $this->actingAs($this->staff, 'staff')
            ->postJson($this->invitationUrl, [
                'last_name' => 'Member',
                'email' => 'test@example.com',
                'Landing_url' => 'https://app.example.com/invite',
                'role' => 'Admin',
            ]);

        $response->assertStatus(422)
            ->assertJsonValidationErrors(['first_name']);
    }

    #[Test]
    #[Group('settings')]
    #[Group('team')]
    public function test_invite_validates_unique_email(): void
    {
        Staff::factory()->create(['email' => 'existing@example.com']);

        $response = $this->actingAs($this->staff, 'staff')
            ->postJson($this->invitationUrl, [
                'first_name' => 'New',
                'last_name' => 'Member',
                'email' => 'existing@example.com',
                'Landing_url' => 'https://app.example.com/invite',
                'role' => 'Admin',
            ]);

        $response->assertStatus(422)
            ->assertJsonValidationErrors(['email']);
    }

    // =========================================================================
    // SHOW
    // =========================================================================

    #[Test]
    #[Group('settings')]
    #[Group('team')]
    public function test_can_get_team_member_by_id(): void
    {
        $member = Staff::factory()->create([
            'name' => 'John Doe',
            'email' => 'john@example.com',
        ]);

        $response = $this->actingAs($this->staff, 'staff')
            ->getJson($this->baseUrl . '/' . $member->id);

        $response->assertOk()
            ->assertJsonPath('data.name', 'John Doe')
            ->assertJsonPath('data.email', 'john@example.com');
    }

    // =========================================================================
    // UPDATE
    // =========================================================================

    #[Test]
    #[Group('settings')]
    #[Group('team')]
    public function test_can_update_team_member_phone(): void
    {
        $member = Staff::factory()->create([
            'phone' => '+1111111111',
        ]);

        $response = $this->actingAs($this->staff, 'staff')
            ->putJson($this->baseUrl . '/' . $member->id, [
                'phone' => '+9999999999',
            ]);

        $response->assertOk()
            ->assertJsonPath('data.phone', '+9999999999');

        $this->assertDatabaseHas('staff', [
            'id' => $member->id,
            'phone' => '+9999999999',
        ]);
    }

    // =========================================================================
    // ARABIC SUPPORT
    // =========================================================================

    #[Test]
    #[Group('settings')]
    #[Group('team')]
    #[Group('arabic')]
    public function test_can_invite_team_member_with_arabic_name(): void
    {
        $response = $this->actingAs($this->staff, 'staff')
            ->postJson($this->invitationUrl, [
                'first_name' => "\u{0623}\u{062D}\u{0645}\u{062F}",
                'last_name' => "\u{0645}\u{062D}\u{0645}\u{062F}",
                'email' => 'ahmed@example.com',
                'phone' => '+966501234567',
                'Landing_url' => 'https://app.example.com/invite',
                'role' => 'Admin',
            ]);

        $response->assertSuccessful();

        $this->assertDatabaseHas('invitations', [
            'email' => 'ahmed@example.com',
        ]);
    }

    // =========================================================================
    // RESPONSE STRUCTURE
    // =========================================================================

    #[Test]
    #[Group('settings')]
    #[Group('team')]
    public function test_team_member_response_structure(): void
    {
        $response = $this->actingAs($this->staff, 'staff')
            ->postJson($this->invitationUrl, [
                'first_name' => 'Structure',
                'last_name' => 'Test',
                'email' => 'structure@example.com',
                'phone' => '+1234567890',
                'Landing_url' => 'https://app.example.com/invite',
                'role' => 'Admin',
            ]);

        $response->assertSuccessful()
            ->assertJsonStructure([
                'data' => [
                    'id',
                    'name',
                    'email',
                    'first_name',
                    'last_name',
                    'phone',
                    'role',
                    'status',
                    'expires_at',
                    'created_at',
                    'updated_at',
                ],
            ]);
    }

    #[Test]
    #[Group('settings')]
    #[Group('team')]
    public function test_team_member_does_not_expose_password(): void
    {
        $response = $this->actingAs($this->staff, 'staff')
            ->postJson($this->invitationUrl, [
                'first_name' => 'Password',
                'last_name' => 'Test',
                'email' => 'password@example.com',
                'phone' => '+1234567890',
                'Landing_url' => 'https://app.example.com/invite',
                'role' => 'Admin',
            ]);

        $response->assertSuccessful();
        $this->assertArrayNotHasKey('password', $response->json('data'));
    }
}
```

## Test Patterns Summary

### Test Setup Pattern

```php
protected function setUp(): void
{
    parent::setUp();                                    // Calls MultiTenancyTestCase::setUp (HTTP faking, service mocking)
    $tenant = $this->createTenant('my-feature');        // Creates tenant with unique ID + domain
    $this->initializeTenant($tenant);                   // Migrates tenant DB, seeds roles
    $this->staff = Staff::factory()->create();          // Creates staff in tenant
    $this->staff->assignRole('Owner');                  // Assigns role for auth
    Mail::fake();                                       // Optional: fake mail
}
```

### Authentication Pattern

Always use `actingAs` with the `staff` guard:

```php
$this->actingAs($this->staff, 'staff')
    ->getJson('/Staff')
    ->assertOk();
```

### Permission Testing Pattern

```php
// Test that a Viewer cannot access admin routes
$viewer = Staff::factory()->create();
$viewer->assignRole('Viewer');

$this->actingAs($viewer, 'staff')
    ->postJson('/Agent', [...])
    ->assertForbidden();

// Test that an Owner can
$this->actingAs($this->staff, 'staff')  // staff has Owner role from setUp
    ->postJson('/Agent', [...])
    ->assertSuccessful();
```

### Response Assertion Pattern

```php
$response->assertOk()                                          // Status 200
    ->assertJsonStructure(['data' => ['*' => ['id', 'name']]]) // Structure check
    ->assertJsonPath('data.name', 'Expected Name')              // Exact value
    ->assertJsonPath('status.check', true);                     // Envelope check

$response->assertStatus(422)
    ->assertJsonValidationErrors(['email', 'name']);             // Validation errors
```

### Database Assertion Pattern

```php
$this->assertDatabaseHas('invitations', [
    'email' => 'test@example.com',
    'first_name' => 'Test',
]);

$this->assertDatabaseMissing('staff', [
    'email' => 'deleted@example.com',
]);
```

### Mail Assertion Pattern

```php
Mail::fake();

// ... perform action that sends mail ...

Mail::assertSent(StaffCreateMail::class);
Mail::assertSentCount(3);
```

### Cross-Tenant Isolation Testing

```php
// Create two tenants
$tenant1 = $this->createTenant('tenant-1');
$tenant2 = $this->createTenant('tenant-2');

// Create agent in tenant 1
$agent = $this->createTenantAgent($tenant1, ['name' => 'Agent A']);

// Verify not accessible from tenant 2
$this->initializeTenant($tenant2);
$this->assertDatabaseMissing('agents', ['name' => 'Agent A']);
```

### PHPUnit Attributes

Tests use PHP 8.1 attributes for metadata:

```php
use PHPUnit\Framework\Attributes\Test;
use PHPUnit\Framework\Attributes\Group;

#[Test]
#[Group('settings')]
#[Group('team')]
public function test_can_list_team_members(): void { ... }
```

Run specific groups: `php artisan test --group=team`

## Key Testing Rules

1. **Always extend `MultiTenancyTestCase`** for any test that touches tenant data
2. **Never make real HTTP requests** -- `Http::preventStrayRequests()` is enforced in setUp
3. **Use the `staff` guard** -- `actingAs($staff, 'staff')`, never default `web`
4. **Clean up tenants** -- handled automatically by tearDown, but use unique test run IDs
5. **Mock external services** -- Elevenlabs, TextAgent, VoiceAgent, VelentsIntegrations are all mocked
6. **Seed roles before testing** -- `initializeTenant()` runs `RolesAndPermissionsSeeder` automatically
7. **Use factories** -- `Staff::factory()`, `Agent::factory()` for test data
8. **Test the API response envelope** -- always check `status.check` and `status.message`
9. **Use PHPUnit attributes** -- `#[Test]`, `#[Group('...')]` for organization
10. **Test both success and validation** -- every endpoint should have positive and negative test cases
