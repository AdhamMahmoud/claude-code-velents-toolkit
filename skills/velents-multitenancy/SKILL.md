---
name: velents-multitenancy
description: stancl/tenancy v3 multi-tenant patterns - database-per-tenant, resolution, jobs
user-invocable: false
---

# VelentsAI Multi-Tenancy Architecture

VelentsAI uses `stancl/tenancy` v3 with database-per-tenant isolation. Each tenant gets a dedicated MySQL database named `tenant_{id}_{APP_NAME}`. Central and tenant contexts have separate route files, migrations, seeders, and guards.

## Tenant Model

```php
<?php

namespace App\Core\Models;

class Tenant extends \Stancl\Tenancy\Database\Models\Tenant implements \Stancl\Tenancy\Contracts\TenantWithDatabase {

    use
        \Stancl\Tenancy\Database\Concerns\HasDatabase ,
        \Stancl\Tenancy\Database\Concerns\HasDomains
    ;

    public function users( ) {
        return $this -> hasMany( \App\Core\Models\User::class );
    }

    public function emails( ) {
        return $this -> hasMany( \App\Core\Models\EmailTenant::class );
    }

}
```

## Tenancy Configuration (config/tenancy.php)

```php
<?php return [
    'tenant_model' => \App\Core\Models\Tenant::class,
    'id_generator' => Stancl\Tenancy\UUIDGenerator::class,
    'domain_model' => \Stancl\Tenancy\Database\Models\Domain::class,

    'central_domains' => [
        '127.0.0.1',
        'localhost',
    ],

    'bootstrappers' => [
        Stancl\Tenancy\Bootstrappers\DatabaseTenancyBootstrapper::class,
        Stancl\Tenancy\Bootstrappers\CacheTenancyBootstrapper::class,
        Stancl\Tenancy\Bootstrappers\FilesystemTenancyBootstrapper::class,
        Stancl\Tenancy\Bootstrappers\QueueTenancyBootstrapper::class,
    ],

    'database' => [
        'central_connection' => env('DB_CONNECTION', 'central'),
        'template_tenant_connection' => null,
        'prefix' => 'tenant_',
        'suffix' => '_' . env( 'APP_NAME' ) ,
        'managers' => [
            'sqlite' => Stancl\Tenancy\TenantDatabaseManagers\SQLiteDatabaseManager::class,
            'mysql' => Stancl\Tenancy\TenantDatabaseManagers\MySQLDatabaseManager::class,
            'pgsql' => Stancl\Tenancy\TenantDatabaseManagers\PostgreSQLDatabaseManager::class,
        ],
    ],

    'cache' => [
        'tag_base' => 'tenant',
    ],

    'filesystem' => [
        'suffix_base' => '',
        'disks' => [],
        'root_override' => [
            'local' => '%storage_path%/app/',
            'public' => '%storage_path%/app/public/',
        ],
        'suffix_storage_path' => false,
        'asset_helper_tenancy' => true,
    ],

    'redis' => [
        'prefix_base' => 'tenant',
        'prefixed_connections' => [],
    ],

    'features' => [],
    'routes' => true,

    'migration_parameters' => [
        '--force' => true,
        '--path' => [database_path('migrations/tenant')],
        '--realpath' => true,
    ],

    'seeder_parameters' => [
        '--class' => \Database\Seeders\Tenant\DatabaseSeeder::class ,
    ],
];
```

## Migration Directory Structure

```
database/
  migrations/
    central/          # Runs on the central database (tenants, domains, users tables)
    tenant/           # Runs per-tenant database (agents, conversations, staff, etc.)
  seeders/
    Tenant/
      DatabaseSeeder.php               # Root tenant seeder
      RolesAndPermissionsSeeder.php    # Seeds 50 permissions + 4 roles per tenant
```

## InitTenant Trait (Cross-Tenant Public ID System)

This trait is used across controllers, repositories, jobs, and requests. It generates composite public IDs for cross-tenant entity references and resolves them back. Format: `{app}_{tenant}_{model}_{column}`.

```php
<?php

namespace App\Core\Helpers;

use Illuminate\Support\Str;

trait InitTenant {

    public function public_id( string $column = 'id' ) : string {
        return strtr( 'app_tenant_model_column' , [
            'app'    => config( 'app.name' ) ,
            'tenant' => tenant( ) -> id ,
            'model'  => class_basename( $this ) ,
            'column' => $this -> $column ,
        ] ) ;
    }

    public function resolve_public_id( string $public_id ) : array {
        [ $app , $tenant , $model , $column ] = explode( '_' , $public_id ) ;
        return [
            'app'    => $app    ,
            'tenant' => $tenant ,
            'model'  => $model  ,
            'column' => $column ,
        ];
    }

    public function data_public_id( string $public_id ) {
        $array = $this -> resolve_public_id( $public_id ) ;
        tenancy( ) -> initialize( $array[ 'tenant' ] ) ;
        return match( $array[ 'model' ] ){
            'Staff'               => \App\Management\Models\Staff           :: ByPublicId( $array[ 'column' ] ) ,
            'Agent'               => \App\Agents\Models\Agent               :: ByPublicId( $array[ 'column' ] ) ,
            'knowledgeBase'       => \App\Agents\Models\knowledgeBase       :: ByPublicId( $array[ 'column' ] ) ,
            'AgentBatch'          => \App\Agents\Models\AgentBatch          :: ByPublicId( $array[ 'column' ] ) ,
            'AgentProvider'       => \App\Agents\Models\AgentProvider       :: ByPublicId( $array[ 'column' ] ) ,
            'Call'                => \App\Calls\Models\Call                 :: ByPublicId( $array[ 'column' ] ) ,
            'Conversation'        => \App\Conversations\Models\Conversation :: ByPublicId( $array[ 'column' ] ) ,
            'AgentCalendarSlot'   => \App\Agents\Models\AgentCalendarSlot   :: ByPublicId( $array[ 'column' ] ) ,
            'PaymentPlanTemplate' => \App\Payment\Models\PaymentPlanTemplate:: ByPublicId( $array[ 'column' ] ) ,
            'ConversationAnalysis'=> \App\Analytics\Models\ConversationAnalysis:: ByPublicId( $array[ 'column' ] ) ,
            default=> null
        } ;
    }

    public function tenant_central( callable $callback ) {
        return \Stancl\Tenancy\Facades\Tenancy::central( $callback ) ;
    }

}
```

## Central Routes (routes/web.php)

Central routes are NOT tenant-scoped. The `WithoutTenant` middleware enforces that these routes are never called within a tenant context.

```php
<?php

Route::any( '/' , function ( \Illuminate\Http\Request $Request ) {
    $route = \App\Core\Support\PendingRequest::instance( ) ;
    return response( ) -> json( [
        $route -> Url_url( ) ,
        $route -> Url_route( ) ,
    ] ) ;
} ) -> name( 'home' ) ;

Route::any( 'storage/{Url}' , \App\Core\Controllers\Storage::class ) -> where( 'Url' , '.*' ) -> name( 'storage' ) ;

Route::group( [ 'as' => 'Auth.' , 'prefix' => 'Auth' , 'controller' => \App\Core\Controllers\Auth::class , 'middleware' => [ \App\Core\Middleware\WithoutTenant::class ] ] , fn( ) => [
    Route::post( 'registration'     , 'registration'     ) -> name( 'registration'     ) ,
    Route::post( 'login'            , 'login'            ) -> name( 'login'            ) ,
    Route::post( 'FindMyTenant'     , 'FindMyTenant'     ) -> name( 'FindMyTenant'     ) ,
    Route::post( 'acceptInvitation' , 'acceptInvitation' ) -> name( 'acceptInvitation' ) ,
    Route::post( 'declineInvitation', 'declineInvitation') -> name( 'declineInvitation') ,
    Route::post( 'requestInvitation', 'requestInvitation') -> name( 'requestInvitation') ,
    Route::get ( 'invitationInfo'   , 'invitationInfo'   ) -> name( 'invitationInfo'   ) ,
    Route::get ( 'loginBySecret'    , 'loginBySecret'    ) -> name( 'loginBySecret'    ) -> middleware( [ 'signed:relative' ] ) ,
] ) ;
```

## WithoutTenant Middleware

```php
<?php

namespace App\Core\Middleware;

class WithoutTenant {

    public function handle( \Illuminate\Http\Request $request , \Closure $next) {
        if( ! is_null( tenant( ) ) ) throw new \Exception( 'you cant make this request under tenant route' ) ;
        return $next( $request ) ;
    }
}
```

## Tenant Route Structure (routes/tenant.php)

All tenant routes are prefixed with `Tenant.` and require `auth:staff` middleware for authenticated endpoints. The tenant context is auto-initialized by Stancl's domain identification middleware.

```php
<?php

Route::group( [ 'as' => 'Tenant.' ] , fn( ) => [
    Route::group( [ 'as' => 'Auth.' , 'prefix' => 'Auth' , 'controller' => \App\Management\Controllers\Auth::class ] , fn( ) => [
        Route::post( 'loginByTenant' , 'login'  ) -> name( 'login'  ) ,
        Route::middleware( 'auth:staff' ) -> group( fn ( ) => [
            Route::post( 'update' , 'update' ) -> name( 'update' ) ,
            Route::get ( 'me'     , 'me'     )
        ] )
    ] ) ,
    Route::middleware( 'auth:staff' ) -> group( fn ( ) => [
        // ... module routes with permission middleware
    ] ) ,
] ) ;
```

## Tenant Lifecycle

### Creating a Tenant

```php
$tenant = Tenant::create(['id' => $tenantId]);
$tenant->domains()->create(['domain' => "{$tenantId}.velents.ai"]);
// DatabaseTenancyBootstrapper creates tenant_{id}_{app_name} database
// Then: artisan tenants:migrate && artisan tenants:seed
```

### Initializing Tenant Context

```php
// In jobs that receive a public_id:
tenancy()->initialize($tenantId);
// ... do work in tenant database ...
tenancy()->end();

// Or use the central callback:
$this->tenant_central(function() {
    // runs in central database context
});
```

### Queue Jobs with Tenant Context

Jobs store the `public_id` (which embeds the tenant ID). On execution, they call `data_public_id()` which initializes the correct tenant before resolving the model.

```php
class AgentInit extends \App\Core\Support\Job {

    public function __construct( public string $public_id ) { }

    public function handle( ) : void {
        $Agent = $this -> data_public_id( $this -> public_id ) ;
        // Now in correct tenant context, Agent model resolved
    }
}
```

## Storage with Tenant Isolation

File paths are always tenant-scoped: `{tenant_id}/{directory}/{uuid}-{name}.{extension}`.

```php
$path = $this->storage_convertFileToUrl('knowledge', $uploadedFile);
// Result: "tenant-abc123/knowledge/65f3a1b2-my-document.pdf"
```

## Key Multi-Tenancy Rules

1. **Database isolation**: Each tenant has its own MySQL database (`tenant_{id}_{app_name}`)
2. **Central vs tenant migrations**: `database/migrations/central/` vs `database/migrations/tenant/`
3. **Tenant seeder**: Each new tenant gets roles/permissions seeded via `RolesAndPermissionsSeeder`
4. **Domain routing**: Tenants are identified by subdomain (e.g., `company.velents.ai`)
5. **Guard**: Tenant-scoped auth uses the `staff` guard, not the default `web` guard
6. **Public IDs**: Cross-tenant references use composite IDs: `{app}_{tenant}_{model}_{column}`
7. **Central fallback**: Use `$this->tenant_central(fn)` or `Tenancy::central(fn)` to escape tenant context
8. **File storage**: S3 paths are prefixed with `tenant()->id` for isolation
9. **Cache isolation**: Cache keys are auto-tagged with `tenant_{id}` via `CacheTenancyBootstrapper`
10. **Queue isolation**: Jobs use `QueueTenancyBootstrapper` to restore tenant context on worker
