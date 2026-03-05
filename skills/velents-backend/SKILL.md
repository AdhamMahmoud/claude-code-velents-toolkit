---
name: velents-backend
description: Laravel backend patterns - controllers, repositories, services, jobs, resources
user-invocable: false
---

# VelentsAI Backend Architecture

The VelentsAI backend follows a strict layered architecture: Controller -> Repository -> Model, with shared behavior extracted into traits. Every layer uses dependency injection and the `SelfMaker` trait for static `::instance()` resolution.

## Directory Structure

```
app/
  Core/
    Controllers/Controller.php    # Abstract base controller
    Repositories/Repository.php   # Abstract base repository
    Resources/JsonResource.php    # Abstract base API resource
    Requests/FormRequest.php      # Base form request with custom validation
    Support/Job.php               # Base queue job
    Support/http.php              # Base HTTP client for external services
    Support/PendingRequest.php    # Extended HTTP pending request
    Helpers/
      Result.php                  # JSON response trait (Successful/Errors)
      InitTenant.php              # Tenant resolution trait (public_id)
      SelfMaker.php               # Static ::instance() factory trait
      logger.php                  # Logging trait (Log/LogError/LogNews)
      Storage.php                 # File storage trait (S3, base64, CSV)
      Url.php                     # URL generation trait (routes, webhooks)
    Models/Tenant.php             # Central tenant model
    Middleware/WithoutTenant.php   # Blocks tenant-scoped requests on central routes
  {Module}/
    Controllers/                  # Module controllers extend Core\Controllers\Controller
    Repositories/                 # Module repos extend Core\Repositories\Repository
    Models/                       # Eloquent models
    Resources/                    # API resources (Resource + Collection per entity)
    Requests/                     # Form requests per action (create, update, etc.)
    Jobs/                         # Queue jobs extend Core\Support\Job
    Enums/                        # PHP 8.1 backed enums
    Services/                     # Business logic services
```

## Base Controller

All controllers extend this abstract class. It provides rate limiting, response helpers, logging, tenant access, URL generation, and storage utilities via traits.

```php
<?php

namespace App\Core\Controllers;

use Illuminate\Support\Facades\{RateLimiter,Auth};

abstract class Controller {

    use
        \App\Core\Helpers\InitTenant ,
        \App\Core\Helpers\Result ,
        \App\Core\Helpers\logger ,
        \App\Core\Helpers\Url ,
        \App\Core\Helpers\Storage
    ;

    public function RateLimiter( \Closure $Fn , string $key , string $message = '' , int $decayMinutes = 60 , int $attempts = 1 ) {

        $key = strtr( 'classfunctionip' , $data = [
            'class'    => class_basename( static::class ) ,
            'function' => $key ,
            'ip'       => ( $_SERVER['HTTP_X_REAL_IP' ] ?? '' ) . ( $_SERVER['HTTP_X_FORWARDED_FOR' ] ?? '' )
        ] ) ;

        if ( RateLimiter::tooManyAttempts( $key , $attempts ) ) {
            $availableIn = \Carbon\CarbonInterval::seconds( RateLimiter::availableIn( $key ) ) -> cascade( ) -> forHumans( ) ;
            return $this -> Errors( 'Too Many Attempts. ' . $message . ' in ' . $availableIn , false , \Illuminate\Http\JsonResponse::HTTP_TOO_MANY_REQUESTS ) ;
        }

        $payload = $Fn( ) ;

        RateLimiter::increment( $key , $decayMinutes * 60 ) ;

        return $payload ;

    }

}
```

## Base Repository

All repositories extend this abstract class. It provides query building with filtering, sorting, word search, pagination, relations, counters, transactions, and CRUD operations.

```php
<?php

namespace App\Core\Repositories;

use Illuminate\Support\Arr;
use Illuminate\Support\Facades\DB;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Builder;
use Illuminate\Database\Query\Builder as Query;

abstract class Repository {

    use
        \App\Core\Helpers\InitTenant ,
        \App\Core\Helpers\SelfMaker  ,
        \App\Core\Helpers\logger     ,
        \App\Core\Helpers\Storage    ,
        \App\Core\Helpers\Url
    ;

    Abstract public function BaseQuery ( ) : Builder ;

    public array $relations          = [ ] ;
    public array $counters           = [ ] ;
    public array $can_word_search_in = [ ] ;
    public array $can_sorted_by      = [ 'id' , 'created_at' ] ;
    public array $can_filterd_By     = [
        [ 'key' => 'id'              , 'type' => 'number[]'             ] ,
        [ 'key' => 'agent_batch_id'  , 'type' => 'number[]'             ] ,
        [ 'key' => 'created_at_from' , 'type' => 'dateTime:Y-m-d H:i:s' ] ,
        [ 'key' => 'created_at_to'   , 'type' => 'dateTime:Y-m-d H:i:s' ]
    ] ;

    public function relations          ( ) : array { return $this -> relations          ; }
    public function counters           ( ) : array { return $this -> counters           ; }
    public function can_sorted_by      ( ) : array { return $this -> can_sorted_by      ; }
    public function can_filterd_By     ( ) : array { return $this -> can_filterd_By     ; }
    public function can_word_search_in ( ) : array { return $this -> can_word_search_in ; }

    public function filters( Builder $Query , Array $Array = [ ] ) : Builder {
        $this -> WhenArrayExists ( 'id'             , $Array , fn ( array $items ) => $Query -> whereIn      ( 'id'             , $items ) ) ;
        $this -> WhenArrayExists ( 'agent_batch_id' , $Array , fn ( array $items ) => $Query -> whereIn      ( 'agent_batch_id' , $items ) ) ;
        $this -> WhenExistsFT    ( 'created_at'     , $Array , fn ( array $dates ) => $Query -> whereBetween ( 'created_at'     , $dates ) ) ;
        $this -> WhenArrayExists ( 'sorting'        , $Array , fn ( array $items ) => collect( $items ) -> filter( fn( $item ) => in_array( $item[ 'by' ] ?? null , $this -> can_sorted_by( ) ) ) -> map( fn( $sort ) => $Query -> orderBy( $sort[ 'by' ] , $sort[ 'as' ] ?? 'asc' ) )) ;
        $this -> WhenArrayExists ( 'word'           , $Array , fn ( array $items ) => $Query -> where( fn( $Query ) => collect( $this -> can_word_search_in( ) ) -> crossJoin( $items ) -> map( fn( $item ) => $Query -> orWhere( $item[ 0 ] , 'like' , '%' . $item[ 1 ] . '%' ) ) ) ) ;
        return $Query ;
    }

    public function Query( Array $Array = [ ] , bool $with = false ) : Builder {
        $Query = $this -> filters ( $this -> BaseQuery ( ) , $Array ) -> withCount ( $this -> counters( ) ) ;
        if ( $with ) $Query = $Query -> with ( $this -> relations ( ) ) ;
        return $Query ;
    }

    public function loudData( Model $Model ) : Model {
        return $Model
            -> loadMissing ( $this -> relations( ) )
            -> loadCount   ( $this -> Counters ( ) )
        ;
    }

    public function statment( ) : Query {
        return DB::query( ) ;
    }

    public function afterCommit( callable $callback ) {
        return DB::afterCommit( $callback ) ;
    }

    public function transaction( \Closure $callback, int $attempts = 1 ) : mixed {
        return DB::transaction( $callback , $attempts ) ;
    }

    public function create( array $payload ) : Model {
        return $this -> transaction( fn( ) => $this -> loudData( $this -> baseQuery( ) -> create( $payload ) -> refresh( ) ) ) ;
    }

    public function update( Model $Model , array $newDetails ) : Model {
        return $this -> transaction( function( ) use( $newDetails , $Model ) {
            $Model -> update( $newDetails ) ;
            return $this -> loudData( $Model -> refresh( ) ) ;
        } ) ;
    }

    public function WhenArrayExists( string | int $key , Array $Array = [ ] , \Closure | null $Closure = null , $default = null ) {
        if ( Arr::has( $Array , $key ) ) return value( $Closure , Arr::wrap( Arr::get( $Array , $key ) ) ) ;
        return func_num_args( ) === 4 ? value( $Closure , ( array ) $default ) : null ;
    }

    public function WhenBoolExists( string | int $key , Array $Array = [ ] , \Closure | null $Closure = null , $default = null ) {
        if ( Arr::has( $Array , $key ) and is_bool( $bool = filter_var( Arr::get( $Array , $key ) , FILTER_VALIDATE_BOOLEAN ) ) ) return value( $Closure , $bool ) ;
        return func_num_args( ) === 4 ? value( $Closure , ( array ) $default ) : null ;
    }

    public function WhenExists( string | int $key , Array $Array = [ ] , \Closure | null $Closure = null , $default = null ) {
        if ( Arr::has( $Array , $key ) ) return value( $Closure , Arr::get( $Array , $key ) ) ;
        return func_num_args( ) === 4 ? value( $Closure , $default ) : null ;
    }

    public function WhenExistsFT( string $key , Array $Array = [ ] , \Closure | null $Closure = null , $default = null ) {
        if ( Arr::has( $Array , [ $key . '_from' , $key . '_to' ] ) ) return value( $Closure , ( array ) [ Arr::get( $Array , $key . '_from' , fn( ) => now( ) -> subYear( ) ) , Arr::get( $Array , $key . '_to' , fn( ) => now( ) -> addYear( ) ) ] ) ;
        return func_num_args( ) === 4 ? value( $Closure , ( array ) $default ) : null ;
    }

    public static function Relation( string $Relation , Builder $Query , Array $Array = [ ] ) : Builder {
        return $Query -> whereRelation ( $Relation , fn ( Builder $Query ) => self::instance( ) -> filters( $Query , $Array ) ) ;
    }

}
```

## Result Trait (JSON Response Envelope)

Every controller and form request uses this trait. All API responses follow the `{ status: { message, check }, ...data }` envelope format.

```php
<?php

namespace App\Core\Helpers;

use Illuminate\Http\JsonResponse ;
use Illuminate\Http\Resources\Json\JsonResource;

trait Result {

    public function ok( ... $args ) : JsonResponse {
        return $this -> Successful( [ ] , ... $args ) ;
    }

    public function Successful( JsonResource|Array|String|Int|Bool|null $array = [ ] , string $message = 'Successful.' , bool $check = true , int $code = JsonResponse::HTTP_OK ) : JsonResponse {
        return new JsonResponse( [ 'status' => [
            'message' => $message ,
            'check'   => $check   ,
        ] ] + collect( ) -> wrap( $array ) -> toArray( ) , $code ) ;
    }

    public function Errors ( string $message = 'error.' , bool $check = false , int $code = JsonResponse::HTTP_UNPROCESSABLE_ENTITY , array $array = [ ] ) : JsonResponse {
        return new JsonResponse( [ 'status' => [
            'message' => $message ,
            'check'   => $check   ,
        ] , 'errors' => $array ] , $code ) ;
    }

}
```

## SelfMaker Trait (Static Factory)

```php
<?php

namespace App\Core\Helpers;

trait SelfMaker {

    static function instance( ) : static {
        return \Illuminate\Support\Facades\App::make( static::class ) ;
    }

}
```

## Base JSON Resource

Resources use composable methods: `id()`, `timestamps()`, `dates()`, `base()`, `Type()`. The `Resource` trait adds the standard status envelope.

```php
<?php

namespace App\Core\Resources;

use BackedEnum;
use Illuminate\Support\{Str,Carbon};

Abstract class JsonResource extends \Illuminate\Http\Resources\Json\JsonResource{

    use Resource ;

    public function id( $request ) { return[
        'id' => $this -> id ,
    ] ; }

    public function timestamps( $request ) { return[
        'created_at' => $this -> date( $this -> created_at ) ,
        'updated_at' => $this -> date( $this -> updated_at ) ,
    ] ; }

    public function dates( $request ) { return[
        ... $this -> timestamps ( $request ) ,
        'deleted_at' => $this -> date( $this -> deleted_at  ) ,
    ] ; }

    public function date( Carbon|null $attr ) :? string {
        return $attr ?-> __toString( );
    }

    public function base( $request ) { return[
        ... $this -> id( $request ) ,
        ... $this -> timestamps( $request ) ,
    ] ; }

    public function Type( $type ) {
        return $this -> when( $type instanceof \BackedEnum , fn( ) => \App\Core\Resources\Type::make( $type ) ) ;
    }

}
```

### Resource Trait (Response Envelope)

```php
<?php

namespace App\Core\Resources;

use Illuminate\Http\JsonResponse ;

trait Resource{

    public function __construct( $resource ,
        public string $message = 'Successful.' ,
        public bool   $check   = true ,
        public int    $code    = JsonResponse::HTTP_OK
    ) {
        parent::__construct( $resource );
    }

    public function with( $request ) : array { return [
        'status' => [
            'message' => $this -> message ,
            'check'   => $this -> check   ,
        ]
    ] ; }

}
```

## Base Form Request

Custom validation error formatting that returns the standard `{ status, errors }` envelope.

```php
<?php

namespace App\Core\Requests;

use Illuminate\Support\Carbon;
use Illuminate\Http\Exceptions\HttpResponseException;

class FormRequest extends \Illuminate\Foundation\Http\FormRequest{

    use \App\Core\Helpers\Result ;
    use \App\Core\Helpers\InitTenant ;

    public function failedValidation( \Illuminate\Contracts\Validation\Validator $validator ) : HttpResponseException {
        throw new HttpResponseException( $this -> Errors( static::summarize ( $validator -> errors( ) -> all( ) ) , array : $validator -> errors( ) -> getMessages( ) ) );
    }

    public static function summarize( array $messages ) : string {
        if ( ! count( $messages ) ) return 'The given data was invalid.' ;
        $message = array_shift( $messages );
        if ( $additional = count( $messages ) ) {
            $pluralized = 1 === $additional ? 'error' : 'errors';
            $message .= " (and {$additional} more {$pluralized})";
        }
        return $message;
    }

    public function _file( string $required = 'required' , string $type = 'csv' , int $sizeByMiga = 1024 ) : array {
        return [ $required , 'file' , 'extensions:' . $type , 'max:' . 2 * $sizeByMiga ] ;
    }

    public function webhookData( string | null $eventEnum = null , array $add = [ ] ) : array {
        return collect( [ ] )
            ->  merge( [
                'webhook'             => [ 'sometimes' , 'array' ] ,
                'webhook.link'        => [ 'required_with:webhook' , 'nullable' , 'url' ] ,
                'webhook.body'        => [ 'sometimes' , 'nullable' , 'array' ] ,
                'webhook.headers'     => [ 'sometimes' , 'nullable' , 'array' ] ,
                'webhook.events'      => [ 'sometimes' , 'nullable' , 'array' ] ,
                'webhook.events.*'    => [ 'required_with:webhook.events' , ! is_null( $eventEnum ) ? \Illuminate\Validation\Rule::enum( $eventEnum ) : 'string' ]
            ] )
            -> merge( $add )
            -> toArray( )
        ;
    }

    public function inFuture( string $required = 'required' , string $date = 'now' , array $add = [ ] ) : array {
        return [ $required , 'date_format:"Y-m-d H:i:s"' , fn( $attribute , $value , $fail ) => Carbon::parse( $value ) -> isBefore( $date ) ? $fail( $attribute . ' must be on or after ' . Carbon::parse( $date ) -> toDateTimeString( ) ) : null , ...$add ] ;
    }

    public function addattributes( array $attributes = [ ] ) : void {
        $this -> merge( $attributes ) ;
        $this -> request -> add( $attributes ) ;
    }

}
```

## Base Job

All queue jobs extend this. Jobs are marked `#[WithoutRelations]` for serialization safety.

```php
<?php

namespace App\Core\Support;

#[\Illuminate\Queue\Attributes\WithoutRelations]
class Job implements \Illuminate\Contracts\Queue\ShouldQueue {

    use
        \Illuminate\Foundation\Queue\Queueable ,
        \App\Core\Helpers\InitTenant ,
        \App\Core\Helpers\SelfMaker ,
        \App\Core\Helpers\Url ,
        \App\Core\Helpers\Storage ,
        \App\Core\Helpers\logger
    ;

    public $maxExceptions = 1 ;

    public function uniqueId( ) : string {
        return static::class ;
    }

    public function failed( ? \Throwable $throwable ): void {
        if( $throwable instanceof \Throwable ) $this -> LogError( $throwable , static::class . $this -> uniqueId( ) ) ;
    }
}
```

## Base HTTP Client (External Service Integration)

Services that call external APIs extend `http`, which extends `PendingRequest`. Config is auto-resolved from `config/services.{ClassName}`.

```php
<?php

namespace App\Core\Support;

use Illuminate\Http\Client\Response;
use Illuminate\Support\Facades\{Log,Config} ;

class http extends PendingRequest{

    protected $_timeout   = 30   ;

    public function __construct( ... $args ){
        parent::__construct( ... $args ) ;
        $this -> setbaseUrl( ) ;
    }

    public function setbaseUrl( string | null $url = null ) : static{
        return $this ;
    }

    public function ResponseData( Response $response , string $channel = 'Responses' ) : Response {
        return parent::ResponseData( $response , channel : $channel ) ;
    }

    public function ErrorData( \Throwable $Throwable , string|null $message = null , string $channel = 'Errors' ) : void {
        parent::ErrorData( $Throwable , channel : $channel ) ;
    }

    public function config( $key = null , $default = null ) {
        return Config::get( 'services.' . class_basename( static::class ) . '.' . $key , $default ) ;
    }

}
```

### PendingRequest (HTTP Client Base)

```php
<?php

namespace App\Core\Support;

use Illuminate\Http\Client\Response;

class PendingRequest extends \Illuminate\Http\Client\PendingRequest{

    use \App\Core\Helpers\SelfMaker ;
    use \App\Core\Helpers\Url ;
    use \App\Core\Helpers\logger ;

    protected $tries      = 3    ;
    protected $retryThrow = false ;
    protected $_timeout   = 30   ;

    public function send( ... $args ) : Response {
        try{
            return $this -> ResponseData( parent::send( ... $args ) ) ;
        } catch ( \Illuminate\Http\Client\RequestException $Exception ){
            $this -> ResponseData( $Exception -> response ) ;
            $this -> ErrorData( $Exception );
            throw $Exception ;
        } catch ( \Throwable $Throwable ) {
            $this -> ErrorData( $Throwable );
            throw $Throwable ;
        }
    }

}
```

## Logger Trait

```php
<?php

namespace App\Core\Helpers;

use Illuminate\Support\Str;
use Illuminate\Support\Facades\Log;

trait logger {

    public function Log( string|\Stringable $message , array $context = [ ] , string $channel = 'stderr' , string $type = 'info' ) : void {
        Log::channel( $channel ) -> $type( Str::limit( $message , 300 ) , $context ) ;
    }

    public function LogError( \Throwable $throwable , string|null $message = null , array $context = [ ] , string $channel = 'Errors' ) : void {
        $exception = \App\Core\Support\Adapters::instance( ) -> Throwable( $throwable ) ;
        $this -> log( $message ?? static::class , [ 'message' => $exception[ 'message' ] , 'context' => $exception[ 'context' ] ] + $context , $channel , 'error' ) ;
        Log::channel( 'op-log' ) -> info( $channel , $exception ) ;
    }

    public function LogResponse( \Illuminate\Http\Client\Response $response , string|null $message = null , array $context = [ ] , string $channel = 'Responses' ) : void {
        $data = \App\Core\Support\Adapters::instance( ) -> Response( $response ) + $context ;
        $this -> log( $message ?? static::class , $data , $channel ) ;
        Log::channel( 'op-log' ) -> info( $channel , $data ) ;
    }

    public function LogNews( string $message , array $vars = [ ] , string $channel = 'Teams-news' ) : void {
        $this -> log( strtr( $message , $vars ) , [ ] , channel : $channel ) ;
    }

}
```

## Storage Trait

Handles file uploads to S3, base64 conversion, CSV generation. All paths are tenant-scoped: `{tenant_id}/{directory}/{uuid}-{name}.{extension}`.

```php
<?php

namespace App\Core\Helpers;

use Illuminate\Support\Str;
use Illuminate\Http\UploadedFile;
use Symfony\Component\Mime\MimeTypes;

trait Storage{

    public function Storage( string | null $disk = null ) : \Illuminate\Filesystem\FilesystemAdapter {
        return \Illuminate\Support\Facades\Storage::disk( $disk ?? config( 'filesystems.default' , 's3' ) );
    }

    public function storage_file_name_conve( string $directory , string $name , string $extension ) {
        return strtr( 'tenant/directory/uuid-name.extension' , [
            'tenant'    => tenant( ) -> id ,
            'directory' => $directory ,
            'uuid'      => uniqid( ) ,
            'name'      => $name ,
            'extension' => $extension
        ] ) ;
    }

    public function storage_convertFileToUrl( string $directory , UploadedFile $file ) {
        $path = $this -> storage_file_name_conve( $directory , Str::slug( pathinfo( $file -> getClientOriginalName( ) , PATHINFO_FILENAME ) , '-' ) , $file -> clientExtension( ) ) ;
        $this -> Storage( ) -> put( $path , $file -> get( ) , 'public' ) ;
        return $path ;
    }

    public function storage_convertLinkToUrl( string $directory , string $file ) {
        $pathinfo = pathinfo( $file ) ;
        $path = $this -> storage_file_name_conve( $directory , Str::slug( $pathinfo[ 'filename' ] , '-' ) , $pathinfo[ 'extension' ] ?? $this -> storage_guessExtensionOnLink( $file ) ) ;
        $this -> Storage( ) -> put( $path , file_get_contents( $file ) , 'public' ) ;
        return $path ;
    }

    public function storage_convertBase64ToUrl( string $content , string $directory , string $file , string $type ) {
        $path = $this -> storage_file_name_conve( $directory , Str::slug( $file , '-' ) , ( new MimeTypes ) -> getExtensions( explode( ';' , $type ) [ 0 ] )[ 0 ] ) ;
        $this -> Storage( ) -> put( $path , base64_decode( $content ) , 'public' ) ;
        return $path ;
    }

    public function storage_ConvertArrayToCsvUploadedFile( array $data ) : UploadedFile {
        $Directory = \Spatie\TemporaryDirectory\TemporaryDirectory::make( ) -> name( 'csv' . uniqid( ) ) -> create( ) ;
        $path      = $Directory -> path( $name = uniqid( ) . '.csv' ) ;
        $file      = fopen( $path , 'w' );
        fputcsv( $file , array_merge( [ 'phone' ] , array_keys( $data[ 0 ] [ 'values' ] ) ) ) ;
        foreach ( $data as $row ) fputcsv( $file , [ 'phone' => $row[ 'phone' ] ] + $row[ 'values' ] );
        return new UploadedFile( $path , $name , 'text/csv' , UPLOAD_ERR_OK , true );
    }

}
```

## Concrete Controller Example: Agent Controller

Shows constructor injection of repository, rate-limited store, permission checks, audit logging, resource responses.

```php
<?php

namespace App\Agents\Controllers;

use Illuminate\Http\Request;
use Illuminate\Http\JsonResponse;
use App\Agents\Models\Agent as model;
use App\Agents\Resources\Agent\{Resource,Collection};
use App\AuditLog\Services\AuditLogService;
use App\AuditLog\Enums\Action;

class Agent extends \App\Core\Controllers\Controller {

    public function __construct( public \App\Agents\Repositories\Agent $repo , public \App\Agents\Repositories\AgentBatch $AgentBatch ) { }

    public function index( Request $Request ) : Collection {
        return Collection::make( $this -> repo -> query( $Request -> all( ) ) -> with( [ 'Channels' ] ) -> paginate( $Request -> get( 'first' , 15 ) ) ) ;
    }

    public function store( \App\Agents\Requests\Agent\create $Request ) : Resource| \Illuminate\Http\JsonResponse {
        return $this -> RateLimiter( function( ) use ( $Request ) {
            $agent = $this -> repo -> create( $Request -> validated( ) + [ 'created_by_id' => auth( ) -> id( ) ] ) ;
            $this -> LogNews( '*"tenant"* has created new Agent : *"agent"* , Type : *"type"*' , [
                'tenant' => tenant( ) -> id ,
                'agent'  => $agent -> name ,
                'type'   => $agent -> type -> value ,
                'email'  => auth( ) -> user( ) -> email ,
                'phone'  => auth( ) -> user( ) -> phone ,
            ] ) ;
            AuditLogService::agentAction(
                action:     Action::AGENT_CREATED,
                description: "Agent '{$agent->name}' created (type: {$agent->type->value})",
                resourceId: $agent->id,
                after:      ['name' => $agent->name, 'type' => $agent->type->value, 'status' => $agent->status?->value],
            );
            return Resource::make( $agent ) ;
        } , 'store' , 'You can create a new agent' , 5 ) ;
    }

    public function show( model $Agent ) : Resource {
        return Resource::make( $this -> repo -> loudData( $Agent ) ) ;
    }

    public function update( model $Agent , \App\Agents\Requests\Agent\create $Request ) {
        $user = $Request -> user( 'staff' ) ;
        if ( ! $user -> hasRole( 'Owner' ) && ! $user -> hasPermissionTo( 'edit_any_agent' ) ) {
            abort_unless(
                $user -> hasPermissionTo( 'edit_own_agent' ) && $Agent -> created_by_id === $user -> id ,
                403 , 'You do not have permission to edit this agent.'
            ) ;
        }
        $beforeState = ['name' => $Agent->name, 'type' => $Agent->type?->value, 'status' => $Agent->status?->value];
        $result = Resource::make( $this -> repo -> updateAgent( $Agent , $Request -> validated( ) ) ) ;
        $Agent->refresh();
        AuditLogService::agentAction(
            action:     Action::AGENT_UPDATED,
            description: "Agent '{$Agent->name}' updated",
            resourceId: $Agent->id,
            before:     $beforeState,
            after:      ['name' => $Agent->name, 'type' => $Agent->type?->value, 'status' => $Agent->status?->value],
        );
        return $result;
    }

    public function destroy( model $Agent ) {
        $user = request( ) -> user( 'staff' ) ;
        if ( ! $user -> hasRole( 'Owner' ) && ! $user -> hasPermissionTo( 'delete_any_agent' ) ) {
            abort_unless(
                $user -> hasPermissionTo( 'delete_own_agent' ) && $Agent -> created_by_id === $user -> id ,
                403 , 'You do not have permission to delete this agent.'
            ) ;
        }
        return Resource::make( $this -> repo -> update( $Agent , [ 'status' => \App\Agents\Enums\Agent\status::unactive ] ) ) ;
    }

}
```

## Concrete Repository Example: Agent Repository

Shows extending base repository with custom `BaseQuery`, `filters`, constructor-injected external services, transaction-wrapped logic, job dispatching.

```php
<?php

namespace App\Agents\Repositories;

use App\Agents\Models\Agent as Model;
use Illuminate\Database\Eloquent\Builder;
use App\Integration\Services\{VelentsIntegrationsVelentsAi, VoiceAgent, Elevenlabs};

class Agent extends \App\Core\Repositories\Repository{

    public function __construct(
        public VelentsIntegrationsVelentsAi $VelentsIntegrationsVelentsAi ,
        public VoiceAgent $VoiceAgent ,
        public Elevenlabs $Elevenlabs ,
    ) { }

    public array $counters           = [ 'Conversations' , 'Calls' , 'Batchs' ] ;
    public array $relations          = [ 'knowledgeBase' , 'Schema' , 'Channels' , 'Tools'  ] ;
    public array $can_word_search_in = [ 'name'          , 'prompt' , 'text->first_message' ] ;

    public function can_filterd_By( ) : array { return parent::can_filterd_By( ) + [
        [ 'key' => 'status'    , 'type' => 'string[]' , 'enum' => \App\Agents\Enums\Agent\status :: class ] ,
        [ 'key' => 'type'      , 'type' => 'string[]' , 'enum' => \App\Agents\Enums\Agent\type   :: class ] ,
        [ 'key' => 'can_start' , 'type' => 'boolean' ] ,
    ] ; }

    public function filters( Builder $Query , Array $Array = [ ] ) : Builder {
        $this -> WhenArrayExists( 'status'    , $Array , fn ( array $items ) => $Query -> whereIn ( 'status'          , $items ) ) ;
        $this -> WhenArrayExists( 'type'      , $Array , fn ( array $items ) => $Query -> whereIn ( 'type'            , $items ) ) ;
        $this -> WhenBoolExists ( 'can_start' , $Array , fn ( bool  $item  ) => $Query -> where   ( 'text->can_start' , $item  ) ) ;
        return parent::filters( $Query , $Array ) ;
    }

    public function BaseQuery( ) : Builder {
        return Model::query( ) ;
    }

    public function create( array $payload ) : Model {
        $Model = $this -> transaction( function( ) use( $payload ) {
            $Model = $this -> baseQuery( ) -> create( $payload ) ;
            $Model -> CalendarData       ( ) -> firstOrCreate( ) ;
            $Model -> ListExtractionData ( ) -> firstOrCreate( ) ;
            return $this -> loudData( $Model -> refresh( ) ) ;
        } ) ;
        dispatch( new \App\Agents\Jobs\Agent\init( $Model -> public_id ) ) ;
        return $Model ;
    }

    public function updateAgent( Model $Model , array $newDetails ) : Model {
        $Model -> update( $newDetails ) ;
        dispatch( new \App\Agents\Jobs\Agent\Update( $Model -> public_id ) ) ;
        return $this -> loudData( $Model -> refresh( ) ) ;
    }

    public function dashboard( Model $Model ) : array {
        $Query = $this -> statment( ) ;
        $Query -> SelectRaw( '( SELECT count( * ) from calls where calls.agent_id = ? ) as calls' , [ $Model -> id ] ) ;
        $Query -> SelectRaw( '( SELECT count( * ) from conversations where conversations.agent_id = ? ) as conversations' , [ $Model -> id ] ) ;
        return ( array ) $Query -> first( ) ;
    }

}
```

## Routing Patterns (Tenant Routes)

Tenant routes are wrapped in `Route::group( [ 'as' => 'Tenant.' ] )` and protected by `auth:staff`. Permission middleware: `'permission:{name}'`. Role middleware: `'role:Owner'`.

```php
<?php
// routes/tenant.php

Route::group( [ 'as' => 'Tenant.' ] , fn( ) => [
    Route::group( [ 'as' => 'Auth.' , 'prefix' => 'Auth' , 'controller' => \App\Management\Controllers\Auth::class ] , fn( ) => [
        Route::post( 'loginByTenant' , 'login' ) -> name( 'login' ) ,
        Route::middleware( 'auth:staff' ) -> group( fn ( ) => [
            Route::post( 'update' , 'update' ) ,
            Route::get ( 'me'     , 'me'     )
        ] )
    ] ) ,
    Route::middleware( 'auth:staff' ) -> group( fn ( ) => [
        Route::group([ 'as' => 'Agent.' , 'prefix' => 'Agent' ] , fn ( ) => [
            Route::put   ( '{Agent}/knowledge' , [ Agent::class , 'knowledge' ] ) -> middleware( 'permission:edit_knowledge_base' ) ,
            Route::patch ( '{Agent}/Schema'    , [ Agent::class , 'Schema'    ] ) -> middleware( 'permission:configure_advanced_settings' ) ,
            Route::get   ( '{Agent}/dashboard' , [ Agent::class , 'dashboard' ] ) -> middleware( 'permission:view_agents' ) ,
            Route::post  ( '{Agent}/archive'   , [ Agent::class , 'archive'   ] ) -> middleware( 'permission:archive_agent' ) ,
        ] ) ,
        Route::apiResource( 'Agent' , Agent::class ) ,
        Route::middleware( [ 'permission:view_team' , 'team-access' ] ) -> group( fn ( ) => [
            Route::apiResource( 'Staff' , Staff::class ) -> except( [ 'store' ] ) ,
            Route::patch( 'Staff/{Staff}/role' , [ Staff::class , 'changeRole' ] ) -> middleware( 'permission:edit_member_roles' ) ,
            Route::post ( 'Staff/{Staff}/transfer-ownership', [ Staff::class , 'transferOwnership' ] ) -> middleware( 'role:Owner' ) ,
        ] ) ,
    ] ) ,
] ) ;
```

## Key Patterns Summary

1. **Constructor injection**: Controllers inject repositories; repositories inject external services
2. **Repository pattern**: All queries go through `BaseQuery()` -> `filters()` -> `Query()` pipeline
3. **Rate limiting**: Wrapped in controller via `$this->RateLimiter(fn, key, message, decay, attempts)`
4. **Transactions**: Repository wraps mutations in `$this->transaction(fn)`
5. **Audit logging**: `AuditLogService::agentAction()` or `AuditLogService::log()` after mutations
6. **Resource responses**: Always return `Resource::make()` or `Collection::make()` with status envelope
7. **Permission checks**: Inline `abort_unless()` with own/any permission pattern
8. **Job dispatching**: After transaction, `dispatch(new Job($model->public_id))`
9. **Public IDs**: Cross-tenant references use `{app}_{tenant}_{model}_{column}` format
10. **Soft deletes via status**: Entities are "deleted" by setting status enum to `unactive`
