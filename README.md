# API REST - DEFINIÇÕES E FORMATOS
---
## Responsabilidades do Client
- Altamente recomendado utilizar header Content-Type, onde você especifica para o servidor o tipo de conteúdo que está sendo enviado, por exemplo: ```Content-Type:application/json```;
- Recomendado utilizar header Accept, assim você especifica para o servidor o tipo de conteúdo que espera receber como resposta, ex: Accept:application/json. Em alguns frameworks web (ex: Laravel), este header é utilizado para serializar as respostas, no caso de uma Exception, por exemplo e conforme exemplificado no Handler de erros mais abaixo;
## Responsabilidades do Servidor
- O servidor deve implementar suporte a **CORS** através dos **HTTP HEADER's** abaixo. Mais sobre o assunto pode ser encontrado em [Enable CORS](https://enable-cors.org/server_nginx.html):
```
Access-Control-Allow-Credentials: true
Access-Control-Allow-Origin: *
```
- As respostas para chamadas do tipo **OPTIONS**, devem ser do tipo **204**, com body vazio e de preferência esta regra deve ser implementada direto no servidor web (apache/nginx) para otimizar a performance de sua aplicação. A opção *Access-Control-Max-Age* habilita o cache das chamadas do tipo *OPTIONS* que são geradas automaticamente pelo browser;
```
Access-Control-Allow-Origin: *
Access-Control-Allow-Credentials: true
Access-Control-Allow-Methods: POST, GET, OPTIONS, PUT, DELETE
Access-Control-Allow-Headers: X-Requested-With, Content-Type, Origin, Authorization
Access-Control-Max-Age: 864000
Content-Type: text/plain
Content-Length: 0
```
- Todas as respostas do servidor para os métodos **DIFERENTES** de *OPTIONS* **DEVEM** incluir o header **Content-type: application/json**, **EXCETO** quando você for servir algum outro tipo de mídia, como um PDF (application/pdf), por exemplo;
- Todas as respostas do servidor para os métodos diferentes de *OPTIONS* **DEVEM** incluir o nome da API em um HEADER do tipo ```X-API-NAME: NOME-DA-API```;
- O status das chamadas **DEVE** ser controlado utilizando os [HTTP STATUS CODE](https://httpstatuses.com/), basicamente significa que 20X para sucesso, 40X para erros originados pelo cliente e 50X para erros no servidor (bugzinho). A maioria dos frameworks frontend de mercado já trata o retorno das requisições de acordo com o status code de resposta, sua aplicação deve garantir sempre um retorno adequado; *__"status code 200 com erro não existe"__*.
- Garanta que seu webserver esteja configurado para responder arquivos de erro estáticos para qualquer tipo de requisição, inclusive quando sua aplicação inteira estiver com problemas (status code > 50X);
- Configure handlers de erro no formato json em seu framework de backend para garantir que ele nunca envie uma resposta no formato HTML, por exemplo;
```php
<?php
// Exemplo Laravel 5.4
// app/Exceptions/Handler.php
<?php

namespace App\Exceptions;

use Exception;
use Illuminate\Auth\AuthenticationException;
use Illuminate\Foundation\Exceptions\Handler as ExceptionHandler;
use Illuminate\Database\Eloquent\ModelNotFoundException;
use Illuminate\Validation\ValidationException;

use Illuminate\Contracts\Container\Container;
use Psr\Log\LoggerInterface;

class Handler extends ExceptionHandler
{

    /**
     * The app config repository.
     *
     * @var \Illuminate\Config\Repository
    */
    protected $config;
    
    /**
     * The container instance.
     *
     * @var \Illuminate\Contracts\Container\Container
    */
    protected $container;

    public function __construct(Container $container)
    {
        $this->config    = $container->config;
        $this->container = $container;
    }

    /**
     * A list of the exception types that should not be reported.
     *
     * @var array
     */
    protected $dontReport = [
        \Illuminate\Auth\AuthenticationException::class,
        \Illuminate\Auth\Access\AuthorizationException::class,
        \Symfony\Component\HttpKernel\Exception\HttpException::class,
        \Illuminate\Database\Eloquent\ModelNotFoundException::class,
        \Illuminate\Session\TokenMismatchException::class,
        \Illuminate\Validation\ValidationException::class,
    ];

    /**
     * Report or log an exception.
     *
     * This is a great spot to send exceptions to Sentry, Bugsnag, etc.
     *
     * @param \Exception $exception
     */
    public function report(Exception $exception)
    {
        if ($this->shouldReport($exception)) {
            // A Config to represent if remote error log is switched on to this env 
            if ($this->config->get('app.monitoring_errors')) {
                $sentry = $this->container->sentry;
                // Build you own $session variable
                $session = $_REQUEST;

                // Add user context
                $sentry->user_context($session);
                
                // Add tags context
                $sentry->tags_context(
                    [
                        'referer'       => isset($_SERVER['HTTP_REFERER']) ? $_SERVER['HTTP_REFERER'] : null,
                        'referer-ip'    => $this->container->request->getClientIp(true)
                    ]
                );
                $sentry->captureException($exception);
            }
        }
        parent::report($exception);
    }

    /**
     * Render an exception into an HTTP response.
     *
     * @param \Illuminate\Http\Request $request
     * @param \Exception               $exception
     *
     * @return \Illuminate\Http\Response
     */
    public function render($request, Exception $exception)
    {
        $code = $exception->getStatusCode();

        if ($exception instanceof ModelNotFoundException) {
            return response()->json(['errors' =>[['code' => 404,'title' => Response::$statusTexts[$code]]]], 404);
        }

        if ($exception instanceof ValidationException) {
            return $this->convertValidationExceptionToResponse($exception, $request);
        }

        if (method_exists($exception, 'getStatusCode')) {
            return response()->json(['errors' =>[['code' => $code,'title' => Response::$statusTexts[$code]]]], $code);
        }

        if ($exception instanceof AuthenticationException) {
            return $this->unauthenticated($request, $exception);
        }

        if ($request->expectsJson()) {
            return response()->json(['errors' =>[['code' => 422,'title' => Response::$statusTexts[422]]]], 422);
        }

        return redirect()->guest(route('login.login'));
    }

    /**
     * Convert an authentication exception into an unauthenticated response.
     *
     * @param \Illuminate\Http\Request                 $request
     * @param \Illuminate\Auth\AuthenticationException $exception
     *
     * @return \Illuminate\Http\Response
     */
    protected function unauthenticated($request, AuthenticationException $exception)
    {
        $code = $exception->getStatusCode();

        if ($request->expectsJson()) {
            return response()->json(['errors' =>[['code' => 401,'title' => Response::$statusTexts[$code]]]], 401);
        }
        return redirect()->guest(route('login.create'));
    }
}

```
- Seus endpoints/rotas **DEVEM** serguir o padrão de recursos REST, valendo-se dos verbos HTTP (GET, POST, PUT e DELETE) para implementar as funcionalidades/actions;

Verb   | Path               | Action  | Route Name
-------|--------------------|---------|------------
GET    | /photos            | index   | photos.index
POST   | /photos            | store   | photos.store
GET    | /photos/{photo_id} | show    | photos.show
PUT    | /photos/{photo_id} | update  | photos.update
DELETE | /photos/{photo_id} | destroy | photos.destroy
- Rest trabalha com recursos, use sua criatividade e evite uma url do tipo /getContents, /getList, ../download ou algo expressando ações;

## Estrutura do Documento de Resposta:
### Nível mais alto:
No nível mais alto de resposta a aplicação poderá retornar objetos JSON dos tipos abaixo:

- ```data```: os dados primários da resposta;
- ```errors```: um array de objetos de erro; uma chamada pode gerar mais de um erro;
- ```meta```: um objeto do tipo meta contém dados adicionais sobre a requisição (ex: paginação, sessão, etc).

#### Exemplo de resposta de um Objeto Individual:
Abaixo um exemplo do conteúdo da resposta para uma chamada do tipo GET (mapeada para a action show no backend) recuperando um registro individual (como *plus* incluindo um relacionamento 1 -> n com o objeto authors). Repare que o retorno dentro de ```data``` é um objeto json e não um array.
**Request:**
```js
GET /articles/13?includes[]=authors HTTP/1.1
```
**Response:**
```js
HTTP/1.1 200 OK
Content-Type: application/json
{
  "data": {
    "id": 13,
    "title": "Manual do Dev FullStackOverflow Junior",
    "body": "The shortest article. Ever.",
    "created_at": "2015-05-22 14:56:29",
    "updated_at": "2015-05-22 14:56:28",
    "authors": [
        {
            "id": 9,
            "name": "Joao Silva",
            "created_at": "2015-05-22 14:56:29",
            "updated_at": "2015-05-22 14:56:28"
        },
        {
            "id": 3,
            "name": "Maria Silva",
            "created_at": "2015-05-22 14:56:29",
            "updated_at": "2015-05-22 14:56:28"
        }
      ]
    }
}
```
#### Exemplo de resposta de uma Lista de Objetos:
Exemplo de uma chamada GET (mapeada para a action index no backend) para uma lista de registros usando um limit de 2 registros (*plus* retornando com um objeto do tipo meta para representar a paginação da solicitação). Repare que o objeto ```data``` retorna um array de objetos.
**Request:**
```js
GET /articles?limit=2 HTTP/1.1
```
**Response:**
```js
HTTP/1.1 200 OK
Content-Type: application/json
{
    "meta": {
        "pagination": [
            "limit": 2,
            "offset": 0,
            "rows_current": 2,
            "rows_total": 4,
            "page_current": 1,
            "page_total": 2
        ]
    },
    "data": [
        {
            "id": 13,
            "title": "Manual do Dev FullStackOverflow Junior",
            "body": "The shortest article. Ever.",
            "created_at": "2015-05-22 14:56:29",
            "updated_at": "2015-05-22 14:56:28",
        },
        {
            "id": 24,
            "title": "Manual do Dev FullStackOverflow Junior - Edição 10",
            "body": "The shortest article. Ever.",
            "created_at": "2015-05-22 14:56:29",
            "updated_at": "2015-05-22 14:56:28",
        }
    ]
}
```
#### Erros
Exemplo de uma resposta para um erro específico da aplicação, ou seja, algum requisito de validação do recurso chamado não foi atendido. Você pode implementar códigos de erros internos da sua aplicação que são implementados na propiedade ```code``` de um objeto error, utilize valores acima de 600 para evitar conflito com os códigos HTTP.
```js
HTTP/1.1 422 Unprocessable Entity
Content-Type: application/json
{
  "errors": [
    {
      "code":   "1623",
      "title":  "Value is too short",
      "detail": "First name must contain at least three characters."
    },
    {
      "code":   "1225",
      "title": "Passwords must contain a letter, number, and punctuation character.",
      "detail": "The password provided is missing a punctuation character."
    },
    {
      "code":   "1226",
      "title": "Password and password confirmation do not match."
    }
  ]
}
```

```js
    // Exemplo de captura do Erro através do componente $resource do AngularJs
    
    var client_resource = $resource('/restapi/resource');
    
    client_resource.query(function(data) {
        // if http code >= 20X
        // Success handler - dados de retorno disponíveis no objeto data
    }, function(error) {
        // Error handler - Dados do erro disponíveis no objeto error
        // Ex: 404, 400, 500, etc
        console.log(error.status);
    });
    
    // Exemplo de captura de Erro através do componente $http do AngularJs:
    
    $http.get('/restapi/resource')
    .success(function(response){
        // if http code >= 20X
    })
    .error(function(error){
    
    })
    
    // Usando $http com then
    $http.get("someUrl")
    .then(function(response){
        // if http code == 200
    }, function(error){
        // else
    }); 

```

- O exemplo abaixo mostra um retorno de erro que deve ser implementado direto no handler de erros do seu framework, no caso o usuário tentou acessar um recurso (url), que não existe.
```js
HTTP/1.1 404 Not Found
Content-Type: application/json
{
  "errors": [
    {
      "code":   "404",
      "title":  "Not Found",
      "detail": "The resource that you are trying to access can't be found."
    }
  ]
}
```

- Abaixo temos um erro do tipo 500, que representa um erro no lado do servidor, pode ser alguma query que não funcionou por falta de uma validação ou algum cenário que vc não previu no momento de estruturar seu código. Recomendamos que este tipo de erro seja loggado pelo seu framework e que seja implementado o report de erros para o sentry.io, desta forma você pode corrigir o problema.
```js
HTTP/1.1 500 Internal Server Error
Content-Type: application/json
{
  "errors": [
    {
      "code":   "500",
      "title":  "Internal Server Error",
      "detail": "Sorry something went wrong, we are working to fix it!"
    }
  ]
}
```

- Lembra que falamos sobre arquivos estáticos de erro? Supondo que vc alterou algo na configuração do seu webserver e agora ele não consegue mais processar os arquivos .php, como será reportado o erro? Em geral os webservers (apache/nginx) retornam por padrão uma página HTML com uma mensagem de erro, mas o seu frontend está esperando um json. Configurando o webserver para retornar arquivos json estáticos em caso de errors, você garante compatibilidade com seu frontend.
```sh
# Configuração do vhost no nginx
server{
    listen *:80
    server_name xpto.com.br
    ...
    error_page 502 /errors/502.json;
    error_page 503 /errors/503.json;
    ...
    location ^~ /errors/ {
        internal;
        add_header Content-Type 'application/json charset=UTF-8 always';
        root  /var/www/static-files;
    }
}
```
```js
HTTP/1.1 502 Bad Gateway
Content-Type: application/json
{
  "errors": [
    {
      "code":   "502",
      "title":  "Bad Gateway",
      "detail": "Sorry our service is unavaiable, we are working to fix it quickly!"
    }
  ]
}
```
