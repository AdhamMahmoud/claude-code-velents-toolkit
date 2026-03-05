---
name: velents-auth-rbac
description: Spatie permissions RBAC with 50 permissions across 11 categories, Passport OAuth2, role hierarchy
user-invocable: false
---

# VelentsAI Authentication and RBAC

VelentsAI uses a dual-auth system: Laravel Passport for OAuth2 tenant login and Sanctum for API token-based plugin access. Role-based access control is implemented via Spatie Permission with a 4-tier role hierarchy and 50 granular permissions across 11 domains.

## Staff Model (Tenant User)

```php
<?php

namespace App\Management\Models;

use Spatie\Permission\Traits\HasRoles;
use Illuminate\Database\Eloquent\Builder;

class Staff extends \App\Core\Models\User{

    use HasRoles;

    protected string $guard_name = 'staff';

    protected $fillable = [
        'name',
        'phone',
        'email',
        'password',
        'last_active_at',
        'invited_by',
    ];

    protected function casts( ) : array {
        return array_merge( parent::casts( ) , [
            'last_active_at' => 'datetime' ,
        ] ) ;
    }

    protected static function newFactory( ) : \Illuminate\Database\Eloquent\Factories\Factory{
        return \Database\Factories\StaffFactory::new( ) ;
    }

    public function findForPassport( string $username ) :? static {
        return $this -> where( 'email' , $username ) -> first( ) ;
    }

    public function scopeStatus( Builder $query , string $status ) : Builder {
        return $status === 'active'
            ? $query -> whereNotNull( 'email_verified_at' )
            : $query -> whereNull( 'email_verified_at' ) ;
    }

    public function isOwner( ) : bool {
        return $this -> hasRole( 'Owner' ) ;
    }

    public function isAdmin( ) : bool {
        return $this -> hasRole( 'Admin' ) ;
    }

    public function isOperator( ) : bool {
        return $this -> hasRole( 'Operator' ) ;
    }

    public function isViewer( ) : bool {
        return $this -> hasRole( 'Viewer' ) ;
    }

    public function getRoleLevel( ) : int {
        if ( $this -> isOwner( ) )    return 4 ;
        if ( $this -> isAdmin( ) )    return 3 ;
        if ( $this -> isOperator( ) ) return 2 ;
        if ( $this -> isViewer( ) )   return 1 ;
        return 0 ;
    }

    public function isAbove( Staff $other ) : bool {
        return $this -> getRoleLevel( ) > $other -> getRoleLevel( ) ;
    }
}
```

## Tenant Auth Controller

```php
<?php

namespace App\Management\Controllers;

class Auth extends \App\Core\Controllers\Controller {

    public function __construct( public \App\Core\Repositories\Auth $repo , public \App\Management\Repositories\Staff $Staff ) { }

    public function login( \App\Management\Requests\Auth\login $Request ) {
        try{
            $data = $this -> repo -> getTokenAndRefreshToken( $Request -> get( 'email' ) , $Request -> get( 'password' ) ) ;
            if( \Illuminate\Support\Arr::has( $data , 'error' ) ) throw new \Illuminate\Auth\AuthenticationException ( 'Unauthenticated.' ) ;
            $staff = \App\Management\Models\Staff::with( 'roles.permissions' ) -> where( 'email' , $Request -> get( 'email' ) ) -> first( ) ;
            $updates = [ 'last_active_at' => now( ) ] ;
            if ( ! $staff -> email_verified_at ) $updates[ 'email_verified_at' ] = now( ) ;
            $staff -> update( $updates ) ;
            \App\AuditLog\Services\AuditLogService::log(
                action:       \App\AuditLog\Enums\Action::ORGANIZATION_LOGIN,
                category:     \App\AuditLog\Enums\Category::ACCESS,
                description:  "User {$staff->email} logged into organization",
                resourceType: 'staff',
                resourceId:   $staff->id,
                metadata:     ['tenant' => tenant()->id],
            );
            return \App\Management\Resources\Auth\token::make( [
                'tenant' => tenant( )                             ,
                'staff'  => $staff -> load( 'roles.permissions' ) ,
                'token'  => $data                                 ,
            ] ) ;
        } catch( \Throwable $Throwable ) {
            return $this -> Errors( $Throwable -> getMessage( ) , code : \Illuminate\Http\JsonResponse::HTTP_UNAUTHORIZED ) ;
        }
    }

    public function me( \Illuminate\Http\Request $Request ) {
        try{
            return \App\Management\Resources\Staff\Resource::make( $Request -> user( ) -> load( 'roles.permissions' ) ) ;
        } catch( \Throwable $Throwable ) {
            return $this -> Errors( $Throwable -> getMessage( ) , code : \Illuminate\Http\JsonResponse::HTTP_UNAUTHORIZED ) ;
        }
    }

}
```

## Central Auth Routes

```php
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

## Roles and Permissions Seeder (Full)

This seeder runs per-tenant and defines 50 permissions across 11 domains, plus 4 preset roles.

```php
<?php

namespace Database\Seeders\Tenant;

use Illuminate\Database\Seeder;
use Spatie\Permission\Models\Role;
use Spatie\Permission\Models\Permission;
use Spatie\Permission\PermissionRegistrar;

class RolesAndPermissionsSeeder extends Seeder {

    public function run( ) : void {

        app( )[ PermissionRegistrar::class ] -> forgetCachedPermissions( ) ;

        $guard = 'staff' ;

        $permissions = [

            // AGENTS (9)
            'agents' => [
                [ 'name' => 'view_agents'                  , 'description' => 'View all agents in the organization'              ] ,
                [ 'name' => 'create_agent'                 , 'description' => 'Create new agents'                                ] ,
                [ 'name' => 'edit_own_agent'               , 'description' => 'Edit agents created by yourself'                  ] ,
                [ 'name' => 'edit_any_agent'               , 'description' => 'Edit any agent in the organization'               ] ,
                [ 'name' => 'delete_own_agent'             , 'description' => 'Delete agents created by yourself'                ] ,
                [ 'name' => 'delete_any_agent'             , 'description' => 'Delete any agent in the organization'             ] ,
                [ 'name' => 'publish_agent'                , 'description' => 'Publish agents to production'                     ] ,
                [ 'name' => 'configure_advanced_settings'  , 'description' => 'Configure advanced agent settings and prompts'    ] ,
                [ 'name' => 'archive_agent'                , 'description' => 'Archive and unarchive agents'                     ] ,
            ] ,

            // KNOWLEDGE BASE (4)
            'knowledge_base' => [
                [ 'name' => 'view_knowledge_base'          , 'description' => 'View knowledge base documents'                   ] ,
                [ 'name' => 'upload_documents'             , 'description' => 'Upload documents to knowledge base'               ] ,
                [ 'name' => 'edit_knowledge_base'          , 'description' => 'Edit knowledge base content'                      ] ,
                [ 'name' => 'delete_documents'             , 'description' => 'Delete knowledge base documents'                  ] ,
            ] ,

            // INBOX & CONVERSATIONS (5)
            'inbox' => [
                [ 'name' => 'view_conversations'           , 'description' => 'View conversations and inbox'                    ] ,
                [ 'name' => 'send_message'                 , 'description' => 'Send messages in conversations'                   ] ,
                [ 'name' => 'end_conversation'             , 'description' => 'End active conversations'                         ] ,
                [ 'name' => 'upload_conversation_files'    , 'description' => 'Upload files in conversations'                    ] ,
                [ 'name' => 'manage_conversations'         , 'description' => 'Create and manage conversations'                  ] ,
            ] ,

            // CALLS (5)
            'calls' => [
                [ 'name' => 'view_calls'                   , 'description' => 'View call history and details'                   ] ,
                [ 'name' => 'make_calls'                   , 'description' => 'Initiate and push calls'                          ] ,
                [ 'name' => 'manage_calls'                 , 'description' => 'Create and manage calls'                          ] ,
                [ 'name' => 'view_call_results'            , 'description' => 'View call results and recordings'                 ] ,
                [ 'name' => 'archive_calls'                , 'description' => 'Archive and unarchive calls'                      ] ,
            ] ,

            // CAMPAIGNS (5)
            'campaigns' => [
                [ 'name' => 'view_campaigns'               , 'description' => 'View all campaigns'                              ] ,
                [ 'name' => 'create_campaign'              , 'description' => 'Create new campaigns'                             ] ,
                [ 'name' => 'edit_campaign'                , 'description' => 'Edit existing campaigns'                          ] ,
                [ 'name' => 'delete_campaign'              , 'description' => 'Delete campaigns'                                 ] ,
                [ 'name' => 'launch_campaign'              , 'description' => 'Launch campaigns to production'                   ] ,
            ] ,

            // ANALYTICS & REPORTS (5)
            'analytics' => [
                [ 'name' => 'view_analytics'               , 'description' => 'View analytics dashboards and reports'           ] ,
                [ 'name' => 'view_own_analytics'           , 'description' => 'View analytics for own agents only'              ] ,
                [ 'name' => 'view_all_analytics'           , 'description' => 'View analytics for any agent'                    ] ,
                [ 'name' => 'view_conversation_transcripts', 'description' => 'View conversation transcripts and quality data'  ] ,
                [ 'name' => 'export_analytics'             , 'description' => 'Export analytics data and reports'                ] ,
            ] ,

            // INTEGRATIONS (4)
            'integrations' => [
                [ 'name' => 'view_integrations'            , 'description' => 'View connected integrations'                     ] ,
                [ 'name' => 'create_integration'           , 'description' => 'Connect new integrations'                         ] ,
                [ 'name' => 'edit_integration'             , 'description' => 'Edit integration configurations'                  ] ,
                [ 'name' => 'delete_integration'           , 'description' => 'Disconnect integrations'                          ] ,
            ] ,

            // CONNECTORS (5)
            'connectors' => [
                [ 'name' => 'view_connectors'              , 'description' => 'View available connectors'                       ] ,
                [ 'name' => 'create_connector'             , 'description' => 'Create new connectors'                            ] ,
                [ 'name' => 'edit_connector'               , 'description' => 'Edit connector configurations'                    ] ,
                [ 'name' => 'delete_connector'             , 'description' => 'Delete connectors'                                ] ,
                [ 'name' => 'manage_connector_settings'    , 'description' => 'Manage connector advanced settings'               ] ,
            ] ,

            // TEAM MANAGEMENT (4)
            'team' => [
                [ 'name' => 'view_team'                    , 'description' => 'View team members list'                           ] ,
                [ 'name' => 'invite_members'               , 'description' => 'Invite new members to the organization'           ] ,
                [ 'name' => 'edit_member_roles'            , 'description' => 'Change member roles'                              ] ,
                [ 'name' => 'remove_members'               , 'description' => 'Remove members from the organization'             ] ,
            ] ,

            // BILLING (2)
            'billing' => [
                [ 'name' => 'view_billing'                 , 'description' => 'View billing information and invoices'            ] ,
                [ 'name' => 'manage_billing'               , 'description' => 'Manage billing and subscription'                  ] ,
            ] ,

            // SETTINGS & ADMIN (4)
            'settings' => [
                [ 'name' => 'view_organization_settings'   , 'description' => 'View organization settings and audit logs'       ] ,
                [ 'name' => 'edit_organization_settings'   , 'description' => 'Edit organization settings and manage API keys'   ] ,
                [ 'name' => 'view_audit_logs'              , 'description' => 'View audit logs for the organization'             ] ,
                [ 'name' => 'export_audit_logs'            , 'description' => 'Export audit logs data'                           ] ,
            ] ,

        ] ;

        foreach ( $permissions as $domain => $domainPermissions ) {
            foreach ( $domainPermissions as $perm ) {
                Permission::firstOrCreate(
                    [ 'name' => $perm[ 'name' ] , 'guard_name' => $guard ] ,
                    [ 'description' => $perm[ 'description' ] , 'domain' => $domain ]
                ) ;
            }
        }

        $allPermissions = collect( $permissions ) -> flatten( 1 ) -> pluck( 'name' ) -> all( ) ;

        $roles = [

            'Owner' => [
                'description' => 'Full access to all platform features'   ,
                'category'    => 'Full Access'                            ,
                'is_preset'   => true                                     ,
                'permissions' => $allPermissions                          ,
            ] ,

            'Admin' => [
                'description' => 'Broad platform management access'       ,
                'category'    => 'Platform Management'                    ,
                'is_preset'   => true                                     ,
                'permissions' => array_values( array_diff( $allPermissions , [
                    'manage_billing'               ,
                    'edit_organization_settings'   ,
                    'export_audit_logs'            ,
                ] ) )                                                     ,
            ] ,

            'Operator' => [
                'description' => 'Create and manage agents, campaigns, and content' ,
                'category'    => 'Customer Operations'                              ,
                'is_preset'   => true                                               ,
                'permissions' => [
                    'view_agents' , 'create_agent' , 'edit_own_agent' ,
                    'delete_own_agent' , 'publish_agent' , 'configure_advanced_settings' ,
                    'view_knowledge_base' , 'upload_documents' ,
                    'edit_knowledge_base' , 'delete_documents' ,
                    'view_campaigns' , 'create_campaign' , 'edit_campaign' , 'launch_campaign' ,
                    'view_analytics' , 'view_own_analytics' ,
                    'view_conversations' , 'send_message' ,
                    'view_calls' ,
                ]                                                                   ,
            ] ,

            'Viewer' => [
                'description' => 'Read-only access to monitor and report'  ,
                'category'    => 'Read Only'                               ,
                'is_preset'   => true                                      ,
                'permissions' => [
                    'view_agents'                  ,
                    'view_knowledge_base'          ,
                    'view_conversations'           ,
                    'view_calls'                   ,
                    'view_campaigns'               ,
                    'view_analytics'               ,
                    'view_all_analytics'           ,
                    'view_conversation_transcripts',
                    'view_integrations'            ,
                    'view_connectors'              ,
                    'view_billing'                 ,
                    'view_organization_settings'   ,
                ]                                                          ,
            ] ,

        ] ;

        foreach ( $roles as $roleName => $config ) {
            $role = Role::firstOrCreate(
                [ 'name' => $roleName , 'guard_name' => $guard ] ,
                [
                    'description' => $config[ 'description' ] ,
                    'category'    => $config[ 'category'    ] ,
                    'is_preset'   => $config[ 'is_preset'   ] ,
                ]
            ) ;
            $role -> syncPermissions( $config[ 'permissions' ] ) ;
        }

    }

}
```

## Role Hierarchy Enforcement in Staff Controller

Users can only modify users below their role level. The `RoleHierarchy` service validates all role mutations.

```php
<?php

namespace App\Management\Controllers;

use \App\Management\Models\Staff as Model;
use \App\Management\Resources\Staff\{Collection,Resource};
use \App\Management\Services\RoleHierarchy;

class Staff extends \App\Core\Controllers\Controller {

    public function __construct( public \App\Management\Repositories\Staff $repo ) { }

    public function index( Request $Request ) : Collection {
        return Collection::make( $this -> repo -> query( $Request -> all( ) , with : true ) -> paginate( $Request -> get( 'first' , 15 ) ) ) ;
    }

    public function update( Model $Staff , \App\Management\Requests\Staff\Update $Request ) : Resource {
        $currentUser = $Request -> user( 'staff' ) -> load( 'roles' );
        $Staff -> load( 'roles' );

        if ( $currentUser -> id !== $Staff -> id ) {
            if ( ! RoleHierarchy::canModifyUser( $currentUser, $Staff ) ) {
                RoleHierarchy::logViolationAttempt( $currentUser, 'update_staff', [
                    'target_staff_id' => $Staff -> id,
                    'target_role' => RoleHierarchy::getUserHighestRole( $Staff ),
                ] );
                abort( 403, 'You do not have permission to modify this user.' );
            }
        }

        return Resource::make( $this -> repo -> update( $Staff , $Request -> validated( ) ) ) ;
    }

    public function destroy( Model $Staff , Request $Request ) {
        $currentUser = $Request -> user( 'staff' ) -> load( 'roles' );
        $Staff -> load( 'roles' );

        try {
            RoleHierarchy::validateDeletion( $currentUser, $Staff );
        } catch ( \Throwable $e ) {
            RoleHierarchy::logViolationAttempt( $currentUser, 'delete_staff', [
                'target_staff_id' => $Staff -> id,
                'target_role' => RoleHierarchy::getUserHighestRole( $Staff ),
                'error' => $e -> getMessage(),
            ] );
            throw $e;
        }

        $this -> repo -> removeStaff( $Staff , $currentUser ) ;
        return $this -> ok( ) ;
    }

    public function changeRole( Model $Staff , Request $Request ) : Resource {
        $Request -> validate( [
            'role' => [ 'required' , 'string' , 'exists:roles,name' ] ,
        ] ) ;

        $currentUser = $Request -> user( 'staff' ) -> load( 'roles' );
        $Staff -> load( 'roles' );
        $newRole = $Request -> get( 'role' );

        try {
            RoleHierarchy::validateRoleChange( $currentUser, $Staff, $newRole );
        } catch ( \Throwable $e ) {
            RoleHierarchy::logViolationAttempt( $currentUser, 'change_role', [
                'target_staff_id' => $Staff -> id,
                'target_current_role' => RoleHierarchy::getUserHighestRole( $Staff ),
                'requested_new_role' => $newRole,
            ] );
            throw $e;
        }

        return Resource::make( $this -> repo -> changeRole( $Staff , $newRole , $currentUser ) ) ;
    }

    public function transferOwnership( Model $Staff , Request $Request ) : Resource {
        $currentUser = $Request -> user( 'staff' ) -> load( 'roles' );
        if ( ! $currentUser -> isOwner( ) ) {
            abort( 403 , 'Only Owners can transfer ownership.' ) ;
        }
        return Resource::make( $this -> repo -> transferOwnership( $currentUser , $Staff ) ) ;
    }

}
```

## Role Controller with Audit Logging

```php
<?php

namespace App\Management\Controllers;

use Spatie\Permission\Models\Role as Model;
use App\AuditLog\Services\AuditLogService;
use App\AuditLog\Enums\{Action, Category};

class Role extends \App\Core\Controllers\Controller {

    public function __construct( public \App\Management\Repositories\Role $repo ) { }

    public function store( \App\Management\Requests\Role\Create $Request ) : Resource {
        $role = $this -> repo -> create( $Request -> validated( ) , $Request -> user( 'staff' ) ) ;
        AuditLogService::log(
            action:       Action::ROLE_CREATED,
            category:     Category::TEAM,
            description:  "Role '{$role->name}' created",
            resourceType: 'role',
            resourceId:   $role->id,
            after:        ['name' => $role->name, 'permissions' => $role->permissions->pluck('name')->toArray()],
        );
        return Resource::make( $role ) ;
    }

    public function update( int $Role , \App\Management\Requests\Role\Update $Request ) : Resource {
        $role = Model::findOrFail( $Role ) ;
        $beforeState = ['name' => $role->name, 'description' => $role->description, 'permissions' => $role->permissions->pluck('name')->toArray()];
        $updatedRole = $this -> repo -> update( $role , $Request -> validated( ) ) ;
        AuditLogService::log(
            action:       Action::ROLE_UPDATED,
            category:     Category::TEAM,
            description:  "Role '{$updatedRole->name}' updated",
            resourceType: 'role',
            resourceId:   $updatedRole->id,
            before:       $beforeState,
            after:        ['name' => $updatedRole->name, 'permissions' => $updatedRole->permissions->pluck('name')->toArray()],
        );
        return Resource::make( $updatedRole ) ;
    }

    public function permissions( ) {
        return response( ) -> json( [
            'status' => [ 'message' => 'Successful.' , 'check' => true ] ,
            'data'   => $this -> repo -> getPermissionsGroupedByDomain( ) ,
        ] ) ;
    }

}
```

## Permission Middleware in Routes

```php
// Permission-based
Route::get  ( '{Agent}/dashboard' , [ Agent::class , 'dashboard' ] ) -> middleware( 'permission:view_agents' ) ;
Route::put  ( '{Agent}/knowledge' , [ Agent::class , 'knowledge' ] ) -> middleware( 'permission:edit_knowledge_base' ) ;
Route::patch( '{Agent}/Schema'    , [ Agent::class , 'Schema'    ] ) -> middleware( 'permission:configure_advanced_settings' ) ;

// Role-based (Owner only for mutations)
Route::post  ( '/'      , [ Role::class , 'store'   ] ) -> middleware( 'role:Owner' ) ;
Route::put   ( '/{Role}', [ Role::class , 'update'  ] ) -> middleware( 'role:Owner' ) ;
Route::delete( '/{Role}', [ Role::class , 'destroy' ] ) -> middleware( 'role:Owner' ) ;

// Combined middleware stacks
Route::middleware( [ 'permission:view_team' , 'team-access' ] ) -> group( fn ( ) => [
    Route::apiResource( 'Staff' , Staff::class ) -> except( [ 'store' ] ) ,
    Route::patch( 'Staff/{Staff}/role' , [ Staff::class , 'changeRole' ] ) -> middleware( 'permission:edit_member_roles' ) ,
    Route::post ( 'Staff/{Staff}/transfer-ownership', [ Staff::class , 'transferOwnership' ] ) -> middleware( 'role:Owner' ) ,
] ) ;
```

## Inline Permission Checks (Own vs Any Pattern)

```php
$user = $Request -> user( 'staff' ) ;

// Owner bypasses; edit_any_agent for admins; edit_own_agent + ownership for operators
if ( ! $user -> hasRole( 'Owner' ) && ! $user -> hasPermissionTo( 'edit_any_agent' ) ) {
    abort_unless(
        $user -> hasPermissionTo( 'edit_own_agent' ) && $Agent -> created_by_id === $user -> id ,
        403 , 'You do not have permission to edit this agent.'
    ) ;
}
```

## Auth Flow Summary

1. **Registration** (central): POST `/Auth/registration` -- creates tenant + database + seeds roles
2. **Central Login**: POST `/Auth/login` -- returns Passport token for central operations
3. **Tenant Discovery**: POST `/Auth/FindMyTenant` -- finds tenant by email
4. **Tenant Login**: POST `{tenant-domain}/Auth/loginByTenant` -- returns Passport token + staff + roles + permissions
5. **Invitation Flow**: POST `/Auth/acceptInvitation` -> staff created in tenant with assigned role
6. **Signed URL Login**: GET `/Auth/loginBySecret` -- one-time signed URL login

## Permission Domain Summary

| Domain | Count | Key Permissions |
|--------|-------|-----------------|
| agents | 9 | view, create, edit_own/any, delete_own/any, publish, configure_advanced, archive |
| knowledge_base | 4 | view, upload, edit, delete |
| inbox | 5 | view_conversations, send_message, end_conversation, upload_files, manage |
| calls | 5 | view, make, manage, view_results, archive |
| campaigns | 5 | view, create, edit, delete, launch |
| analytics | 5 | view, view_own, view_all, view_transcripts, export |
| integrations | 4 | view, create, edit, delete |
| connectors | 5 | view, create, edit, delete, manage_settings |
| team | 4 | view, invite, edit_roles, remove |
| billing | 2 | view, manage |
| settings | 4 | view_org_settings, edit_org_settings, view_audit_logs, export_audit_logs |

## Role Permission Matrix

| Role | Count | Exclusive Restrictions |
|------|-------|-----------------------|
| Owner | 50 | None -- full access |
| Admin | 47 | No manage_billing, edit_organization_settings, export_audit_logs |
| Operator | 19 | Own agents only, no delete campaigns, limited analytics |
| Viewer | 12 | Read-only across all domains |
