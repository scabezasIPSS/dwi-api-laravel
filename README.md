# Configuración con API Simple

# Instalación de la funcionalidad

```bash
php artisan install:api
```

## Versionamiento

Agregar el versionamiento del API. Modificar el archivo **/bootstrap/app.php** agregando:

```php
api: __DIR__.'/../routes/api.php',
apiPrefix: '/api/v1',
```

### Endpoints

Código del archivo **routes/api.php**:

```php
<?php

use App\Http\Controllers\DemoController;
use Illuminate\Support\Facades\Route;

Route::get('/demo', [DemoController::class, 'index']);
Route::get('/demo/{_id}', [DemoController::class, 'show']);
Route::post('/demo', [DemoController::class, 'store']);
Route::put('/demo/{_id}', [DemoController::class, 'update']);
Route::delete('/demo/{_id}', [DemoController::class, 'destroy']);
// Definición de rutas personalizadas para LOCK y UNLOCK
Route::match(['lock'], '/demo/{_id}', [DemoController::class, 'disable']);
Route::match(['unlock'], '/demo/{_id}', [DemoController::class, 'enable']);
```

## Provider

```bash
php artisan make:provider RouteServiceProvider
```

### Código

```php
<?php

namespace App\Providers;

use Illuminate\Foundation\Support\Providers\RouteServiceProvider as ServiceProvider;
use Illuminate\Support\Facades\Route;

class RouteServiceProvider extends ServiceProvider
{
    /**
     * This namespace is applied to your controller routes.
     *
     * In addition, it is set as the URL generator's root namespace.
     *
     * @var string
     */
    protected $namespace = 'App\Http\Controllers';

    /**
     * Define your route model bindings, pattern filters, etc.
     *
     * @return void
     */
    public function boot()
    {
        //

        parent::boot();
    }

    /**
     * Define the routes for the application.
     *
     * @return void
     */
    public function map()
    {
        $this->mapApiRoutes();
        $this->mapWebRoutes();

        //
    }

    /**
     * Define the "web" routes for the application.
     *
     * These routes all receive session state, CSRF protection, etc.
     *
     * @return void
     */
    protected function mapWebRoutes()
    {
        Route::middleware('web')
            ->namespace($this->namespace)
            ->group(base_path('routes/web.php'));
    }

    /**
     * Define the "api" routes for the application.
     *
     * These routes are typically stateless.
     *
     * @return void
     */
    protected function mapApiRoutes()
    {
        Route::prefix('api')
            ->middleware('api')
            ->namespace($this->namespace)
            ->group(base_path('routes/api.php'));
    }
}
```

# Crear Modelo, Archivo de Migración y Controlador

```bash
php artisan make:model Demo -mc
```

## Archivo de Migración

```php
Schema::create('demo', function (Blueprint $table) {
    $table->id();
    $table->string('titulo');
    $table->text('contenido');
    $table->boolean('activo')->default(false);
    $table->timestamps();
});
```

## Archivo Modelo

```php
class Demo extends Model
{
    use HasFactory;

    protected $table = 'demo';

    protected $fillable = [
        'titulo',
        'contenido',
        'activo'
    ];
}
```

## Archivos Controladores

### Archivo Controller.php

```php
<?php

namespace App\Http\Controllers;

abstract class Controller
{
    private $_header;
    private $_api_get;
    private $_api_post;
    private $_api_put;
    private $_api_lock;
    private $_api_unlock;
    private $_api_delete;

    public function __construct()
    {
        $this->_header = isset(getallheaders()['Authorization']) ? getallheaders()['Authorization'] : null;
        $this->_api_get = 'Bearer abc0';
        $this->_api_post = 'Bearer abc1';
        $this->_api_put = 'Bearer abc2';
        $this->_api_lock = 'Bearer abc123';
        $this->_api_unlock = 'Bearer abc321';
        $this->_api_delete = 'Bearer abc4';
    }

    public function getHeader()
    {
        return $this->_header;
    }

    public function getAuthBearer($_metodo)
    {
        switch ($_metodo) {
            case 'GET':
                return $this->_api_get == $this->_header;
                break;
            case 'POST':
                return $this->_api_post == $this->_header;
                break;
            case 'PUT':
                return $this->_api_put == $this->_header;
                break;
            case 'DELETE':
                return $this->_api_delete == $this->_header;
                break;
            case 'LOCK':
                return $this->_api_lock == $this->_header;
            case 'UNLOCK':
                return $this->_api_unlock == $this->_header;
                break;
            default:
                return false;
                break;
        }
        return false;
    }
}
```

### Archivo DemoController.php

```php
<?php

namespace App\Http\Controllers;

use App\Models\Demo;
use Illuminate\Http\Request;

class DemoController extends Controller
{
    public function index()
    {
        if ($this->getHeader() == NULL) {
            return response()->json(['message' => 'Sin Autorización'], 401);
        }

        if (!$this->getAuthBearer('GET')) {
            return response()->json(['message' => 'Sin Autorización, incorrecta'], 401);
        }

        $data = Demo::all();
        foreach ($data as $d) {
            $d->activo = $d->activo == 1 ? true : false;
        }
        return response()->json($data, 200);
    }

    public function store(Request $_request)
    {
        if ($this->getHeader() == NULL) {
            return response()->json(['message' => 'Sin Autorización'], 401);
        }

        if (!$this->getAuthBearer('POST')) {
            return response()->json(['message' => 'Sin Autorización, incorrecta'], 401);
        }

        $validatedData = $_request->validate([
            'titulo' => 'required',
            'contenido' => 'required'
        ]);
        $data = Demo::create($validatedData);
        return response()->json($data, 201);
    }

    public function show(string $_id)
    {
        if ($this->getHeader() == NULL) {
            return response()->json(['message' => 'Sin Autorización'], 401);
        }

        if (!$this->getAuthBearer('GET')) {
            return response()->json(['message' => 'Sin Autorización, incorrecta'], 401);
        }

        $data = Demo::find($_id);
        if (!$data) {
            return response()->json(['message' => 'Not Found'], 404);
        }
        $data->activo = $data->activo == 1 ? true : false;
        return response()->json($data, 200);
    }

    public function update(Request $_request, string $_id)
    {
        if ($this->getHeader() == NULL) {
            return response()->json(['message' => 'Sin Autorización'], 401);
        }

        if (!$this->getAuthBearer('PUT')) {
            return response()->json(['message' => 'Sin Autorización, incorrecta'], 401);
        }

        $data = Demo::find($_id);
        if (!$data) {
            return response()->json(['message' => 'Not Found'], 404);
        }
        $validatedData = $_request->validate(
            [
                'titulo' => 'required',
                'contenido' => 'required'
            ]
        );
        $data->update($validatedData);
        return response()->json($data, 300);
    }

    public function destroy(string $_id)
    {
        if ($this->getHeader() == NULL) {
            return response()->json(['message' => 'Sin Autorización'], 401);
        }

        if (!$this->getAuthBearer('DELETE')) {
            return response()->json(['message' => 'Sin Autorización, incorrecta'], 401);
        }

        $data = Demo::find($_id);
        if (!$data) {
            return response()->json(['message' => 'Not Found'], 404);
        }
        $data->delete();
        return response()->json($data, 202);
    }

    public function enable(Request $_request, string $_id)
    {
        if ($this->getHeader() == NULL) {
            return response()->json(['message' => 'Sin Autorización'], 401);
        }

        if (!$this->getAuthBearer('UNLOCK')) {
            return response()->json(['message' => 'Sin Autorización, incorrecta'], 401);
        }

        $data = Demo::find($_id);
        if (!$data) {
            return response()->json(['message' => 'Not Found'], 404);
        }

        $data->activo = true;
        $data->save();
        return response()->json($data, 300);
    }

    public function disable(Request $_request, string $_id)
    {
        if ($this->getHeader() == NULL) {
            return response()->json(['message' => 'Sin Autorización'], 401);
        }

        if (!$this->getAuthBearer('LOCK')) {
            return response()->json(['message' => 'Sin Autorización, incorrecta'], 401);
        }

        $data = Demo::find($_id);
        if (!$data) {
            return response()->json(['message' => 'Not Found'], 404);
        }

        $data->activo = false;
        $data->save();
        return response()->json($data, 300);
    }
}
```

# Endpoints en POSTMAN

GET  | http://localhost:8000/api/demo/     | Authorization Bearer: abc0

GET  | http://localhost:8000/api/demo/2    | Authorization Bearer: abc0

POST | http://localhost:8000/api/demo/     | Authorization Bearer: abc1

JSON Body
```json
{
    "titulo": "T1",
    "contenido" : "C1"
}
```
PUT  | http://localhost:8000/api/demo/1     | Authorization Bearer: abc2
JSON Body
```json
{
    "titulo": "Título 1",
    "contenido" : "Contenido 1"
}
```
UNLOCK | http://localhost:8000/api/demo/1   | Authorization Bearer: abc321

UNLOCK | http://localhost:8000/api/demo/1   | Authorization Bearer: abc123

DELETE | http://localhost:8000/api/demo/1   | Authorization Bearer: abc4
