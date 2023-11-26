```bash
# Create Laravel Project
composer create-project laravel/laravel app
cd app

# Initialize docker-compose.yml
touch docker-compose.yml
```
```yml
version: "3.8"
services:
  # Database Service
  database:
    image: mysql:8.0 # https://hub.docker.com/_/mysql
    ports:
      - 3307:3306
    environment:
      - MYSQL_DATABASE=${DB_DATABASE}
      - MYSQL_USER=${DB_USERNAME}
      - MYSQL_PASSWORD=${DB_PASSWORD}
      - MYSQL_ROOT_PASSWORD=${DB_PASSWORD}
    volumes:
      - db-data:/var/lib/mysql
volumes:
  db-data: ~ # docker volume inspect app_db-data
```
```bash
vi .env
# DB_HOST=database
# DB_DATABASE=app
# DB_USERNAME=user
# DB_PASSWORD=secret

# QUEUE_CONNECTION=database
```
```bash
docker-compose ps
docker-compose down
docker-compose up --build -d
docker-compose ps
docker logs app-database-1
docker volume inspect app_db-data
mysql -h 127.0.0.1 -P 3307 -u user -p
```
```yaml
  # Redis Service
  redis:
    image: redis:alpine
    command: redis-server --appendonly yes --requirepass "${REDIS_PASSWORD}" # Append Only File (AOF) mode to persist data
    ports:
      - 6370:6379
```
```bash
vi .env
# REDIS_HOST=redis
# REDIS_PASSWORD=secret
# CACHE_DRIVER=redis
# SESSION_DRIVER=redis
```
```bash
docker-compose down
docker-compose up -d
docker-compose ps
redis-cli -h 127.0.0.1 -p 6370
```
```bash
# Initialize Dockerfile
touch Dockerfile
```
```Dockerfile
FROM php:8.1 as php

RUN apt-get update -y
RUN apt-get install -y unzip libpq-dev libcurl4-gnutls-dev
RUN docker-php-ext-install pdo pdo_mysql bcmath
RUN pecl install -o -f redis \
    && rm -rf /tmp/pear \
    && docker-php-ext-enable redis

WORKDIR /var/www
COPY . .

COPY --from=composer:2.3.5 /usr/bin/composer /usr/bin/composer

ENV PORT=8000
RUN chmod +x docker/entrypoint.sh
ENTRYPOINT [ "docker/entrypoint.sh" ]
```
```bash
# Initialize docker/entrypoint.sh
mkdir docker && touch docker/entrypoint.sh
```
```bash
#!/bin/bash

if [ ! -f "vendor/autoload.php" ]; then
    composer install --no-progress --no-interaction
fi

if [ ! -f ".env" ]; then
    echo "Creating env file for env $APP_ENV"
    cp .env.example .env
else
    echo "env file exists."
fi

php artisan migrate
php artisan key:generate

php artisan serve --port=$PORT --host=0.0.0.0 --env=.env
exec docker-php-entrypoint "$@" # Instructing the container to execute the docker-php-entrypoint script or command, passing all the command-line arguments ("$@") to it. This is a common pattern in Docker entrypoint scripts to allow flexibility in providing additional configuration or commands when starting the container.
# docker run -it your-php-container arg1 arg2
# The exec docker-php-entrypoint "$@" command would effectively translate to:
# docker-php-entrypoint arg1 arg2
```
```bash
chmod +x docker/entrypoint.sh
```
```yaml
  # PHP Service
  php:
    build:
      context: .
      target: php # Specifies a specific build stage named `php` from the multi-stage Dockerfile. 
      args: # build-time arguments
        - APP_ENV=${APP_ENV}
    environment: # Specifies environment variables for the running container.
      - APP_ENV=${APP_ENV}
      - CONTAINER_ROLE=app
    working_dir: /var/www
    volumes:
      - ./:/var/www # Mounts the current directory (.) on the host to /var/www inside the container. This allows for live code reloading, as changes made on the host are reflected inside the container.
    ports:
      - 8001:8000 # Maps port 8001 on the host to port 8000 inside the container.
    depends_on: # Ensures that the php service will only start after the specified services are up and running.
      - database
      - redis
```
```bash
docker-compose down
docker-compose up -d
curl 127.0.0.1:8001
```
```Dockerfile
# Node
FROM node:14-alpine as node

WORKDIR /var/www
COPY . .

RUN npm install --global cross-env
RUN npm install

VOLUME /var/www/node_modules
```
```bash
# This line creates a volume at /var/www/node_modules. When a container is run from an image that includes this VOLUME instruction, Docker will manage this volume separately from the container's filesystem. The idea is to allow you to persist data in that location even if the container is removed.
```
```yaml
  # Node Service
  node:
    build:
      context: .
      target: node
    volumes:
      - .:/usr/src
      - ./node_modules:/usr/src/node_modules
    tty: true
```
```bash
docker-compose down
docker-compose up -d
docker ps -a
docker logs app-node-1
```
```yaml
  # Queue Service
  queue:
    build:
      context: .
      target: php
      args:
        - APP_ENV=${APP_ENV}
    environment:
      - APP_ENV=${APP_ENV}
      - CONTAINER_ROLE=queue
    working_dir: /var/www
    volumes:
      - ./:/var/www
```
```bash
vi docker/entrypoint.sh
role=${CONTAINER_ROLE:-app}

if [ "$role" = "app" ]; then
    php artisan migrate
    php artisan key:generate
    php artisan config:clear

    php artisan serve --port=$PORT --host=0.0.0.0 --env=.env
    exec docker-php-entrypoint "$@"
elif [ "$role" = "queue" ]; then
    echo "Running the queue..."
    php /var/www/artisan queue:work --verbose --tries=3 --timeout=18
fi
```
```bash
docker-compose down
docker-compose up -d
docker ps -a
docker logs app-queue-1
```
```yaml
  # Websocket Service
  websocket:
    build:
      context: .
      target: php
      args:
        - APP_ENV=${APP_ENV}
    environment:
      - APP_ENV=${APP_ENV}
      - CONTAINER_ROLE=websocket
    working_dir: /var/www
    volumes:
      - ./:/var/www
    ports:
      - 6001:6001
    depends_on:
      - database
      - redis
```
```bash
vi docker/entrypoint.sh
elif [ "$role" = "websocket" ]; then
    echo "Running the websocket server..."
    php artisan websockets:serve
fi
```
```bash
docker-compose down
docker-compose up -d
docker ps -a
docker logs app-websocket-1
```
```bash
# Application Setup
```
```bash
mkdir app/Events
touch app/Events/SendMessage.php
```
```php
<?php

namespace App\Events;

use Illuminate\Broadcasting\Channel;
use Illuminate\Broadcasting\InteractsWithSockets;
use Illuminate\Broadcasting\PresenceChannel;
use Illuminate\Broadcasting\PrivateChannel;
use Illuminate\Contracts\Broadcasting\ShouldBroadcast;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;

class SendMessage implements ShouldBroadcast
{
    use Dispatchable, InteractsWithSockets, SerializesModels;

    public string $name;
    public string $message;
    public string $time;

    /**
     * Create a new event instance.
     *
     * @return void
     */
    public function __construct(string $name, string $message, string $time)
    {
        $this->name = $name;
        $this->message = $message;
        $this->time = $time;
    }

    public function broadcastWith() {
        return [
            "name" => $this->name,
            "message" => $this->message,
            "time" => $this->time
        ];
    }

    /**
     * Get the channels the event should broadcast on.
     *
     * @return \Illuminate\Broadcasting\Channel|array
     */
    public function broadcastOn()
    {
        return new Channel("SendMessageEvent");
    }
}
```
```bash
vi routes/web.php
```
```php
use App\Events\SendMessage;
use BeyondCode\LaravelWebSockets\Apps\AppProvider;
use BeyondCode\LaravelWebSockets\Dashboard\DashboardLogger;
use Illuminate\Http\Request;

Route::get('/', function (AppProvider $appProvider) {
    return view('chat-app-example', [
        "port" => "6001",
        "host" => "127.0.0.1",
        "authEndpoint" => "/api/sockets/connect",
        "logChannel" => DashboardLogger::LOG_CHANNEL_PREFIX,
        "apps" => $appProvider->all()
    ]);
});

Route::post("/chat/send", function(Request $request) {
    $message = $request->input("message", null);
    $name = $request->input("name", "Anonymous");
    $time = (new DateTime(now()))->format(DateTime::ATOM);
    if ($name == null) {
        $name = "Anonymous";
    }
    SendMessage::dispatch($name, $message, $time);
});
```
```bash
touch resources/views/chat-app-example.blade.php
```
```html
<html>

<head>
  <title>Laravel WebSockets Chat Example</title>

  <script src="https://code.jquery.com/jquery-3.6.0.min.js"
          integrity="sha256-/xUj+3OJU5yExlq6GSYGSHk7tPXikynS7ogEvDej/m4=" crossorigin="anonymous"></script>
  <script src="https://js.pusher.com/7.0.3/pusher.min.js"></script>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/vue/2.6.14/vue.min.js"
          integrity="sha512-XdUZ5nrNkVySQBnnM5vzDqHai823Spoq1W3pJoQwomQja+o4Nw0Ew1ppxo5bhF2vMug6sfibhKWcNJsG8Vj9tg=="
          crossorigin="anonymous" referrerpolicy="no-referrer"></script>
  <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.0.2/dist/css/bootstrap.min.css" rel="stylesheet"
        integrity="sha384-EVSTQN3/azprG1Anm3QDgpJLIm9Nao0Yz1ztcQTwFspd3yD65VohhpuuCOmLASjC" crossorigin="anonymous">
  <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.0.2/dist/js/bootstrap.bundle.min.js"
          integrity="sha384-MrcW6ZMFYlzcLA8Nl+NtUVF0sA7MsXsP1UyJoMp4YLEuNSfAP+JcXn/tWtIaxVXM" crossorigin="anonymous">
  </script>
</head>

<body>
  <div class="container" id="app">
    <br>
    <br>
    <p class="text-center p-0 m-0">Laravel WebSockets</p>
    <h1 class="text-center p-0 m-0">Chat App Example</h1>
    <div class="card mt-4">
      <div class="card-header p-2">
        <form>
          <div class="col-lg-2 col-md-3 col-sm-12 mt-2 p-0">
            <label>Name</label>
            <input class="form-control form-control-sm" placeholder="Name" v-model="name">
          </div>
          <div class="col-lg-1 col-md-2 col-sm-12 mt-2 p-0">
            <button v-if="connected === false" v-on:click="connect()" type="button"
                    class="mr-2 btn btn-sm btn-primary w-100">
              Connect
            </button>
            <button v-if="connected === true" v-on:click="disconnect()" type="button"
                    class="mr-2 btn btn-sm btn-danger w-100">
              Disconnect
            </button>
          </div>
        </form>
        <div>
          <p>Channels current state is @{{ state }}</p>
        </div>
      </div>
      <div v-if="connected === true" class="card-body">

        <div class="col-12 bg-light pt-2 pb-2 mt-3">
          <p class="p-0 m-0 ps-2 pe-2" v-for="(message, index) in incomingMessages">
            (@{{ message.time }}) <b>@{{ message.name }}</b>
            @{{ message.message }}
          </p>
        </div>

        <h4 class="mt-4">Message</h4>
        <form>
          <div class="row mt-2">
            <div class="col-12 text-white" v-show="formError === true">
              <div class="bg-danger p-2 mb-2">
                <p class="p-0 m-0"><b>Error:</b> Invalid message.</p>
              </div>
            </div>
            <div class="col-12">
              <div class="form-group">
                <textarea v-model="message" placeholder="Your message ..." class="form-control" rows="3"></textarea>
              </div>
            </div>
          </div>
          <div class="row text-right mt-2">
            <div class="col-lg-10">

            </div>
            <div class="col-lg-2">
              <button type="button" v-on:click="sendMessage()" class="btn btn-small btn-primary w-100">Send
                Message</button>
            </div>
          </div>
        </form>
      </div>
    </div>
  </div>
  <script>
    new Vue({
      "el": "#app",
      "data": {
        connected: false,

        pusher: null,
        app: null,
        apps: {!! json_encode($apps) !!},
        logChannel: "{{ $logChannel }}",
        authEndpoint: "{{ $authEndpoint }}",
        host: "{{ $host }}",
        port: "{{ $port }}",

        state: null,

        message: null,
        name: null,

        formError: false,
        incomingMessages: [

        ],
      },
      mounted() {
        this.app = this.apps[0] || null;
      },
      methods: {
        connect() {
          this.pusher = new Pusher("staging", {
            wsHost: this.host,
            wsPort: this.port,
            wssPort: this.port,
            wsPath: this.app.path,
            disableStats: true,
            authEndpoint: this.authEndpoint,
            forceTLS: false,
            auth: {
              headers: {
                "X-CSRF-Token": "{{ csrf_token() }}",
                "X-App-ID": this.app.id
              }
            },
            enabledTransports: ["ws", "flash"]
          });

          this.pusher.connection.bind('state_change', states => {
            this.state = states.current
          });

          this.pusher.connection.bind('connected', () => {
            this.connected = true;
          });

          this.pusher.connection.bind('disconnected', () => {
            this.connected = false;
          });

          this.pusher.connection.bind('error', event => {
            this.formError = true;
          });

          this.subscribeToAllChannels();
        },
        subscribeToAllChannels() {
          [
            "api-message"
          ].forEach(channelName => this.subscribeToChannel(channelName));
        },
        subscribeToChannel(channelName) {
          let inst = this;
          this.pusher.subscribe(this.logChannel + channelName)
            .bind("log-message", (data) => {
              // console.log(data);
              if (data.type === "api-message") {
                if (data.details.includes("SendMessageEvent")) {
                  let messageData = JSON.parse(data.data);
                  let utcDate = new Date(messageData.time);
                  messageData.time = utcDate.toLocaleString();
                  inst.incomingMessages.push(messageData);
                }
              }
            });
        },
        disconnect() {
          this.connected = false;
        },
        sendMessage() {
          this.formError = false;
          if (this.message === "" || this.message === null) {
            this.formError = true;
          } else {
            $.post("/chat/send", {
              _token: '{{ csrf_token() }}',
              message: this.message,
              name: this.name
            }).fail(() => {
              alert("Error sending event.")
            });
          }
        }
      }
    });
  </script>
</body>

</html>
```
```bash
composer require beyondcode/laravel-websockets --with-all-dependencies
php artisan vendor:publish --provider="BeyondCode\LaravelWebSockets\WebSocketsServiceProvider" --tag="migrations"
php artisan vendor:publish --provider="BeyondCode\LaravelWebSockets\WebSocketsServiceProvider" --tag="config"
```
```bash
vi config/websockets.php
```
```php
<?php

return [

    /*
     * Set a custom dashboard configuration
     */
    'dashboard' => [
        'port' => env('LARAVEL_WEBSOCKETS_EXTERNAL_PORT', 6001),
        "host" => env("LARAVEL_WEBSOCKETS_EXTERNAL_HOST", "127.0.0.1")
    ],

    /*
     * This package comes with multi tenancy out of the box. Here you can
     * configure the different apps that can use the webSockets server.
     *
     * Optionally you specify capacity so you can limit the maximum
     * concurrent connections for a specific app.
     *
     * Optionally you can disable client events so clients cannot send
     * messages to each other via the webSockets.
     */
    'apps' => [
        [
            'id' => env('PUSHER_APP_ID', "staging"),
            'name' => env('APP_NAME',  "staging"),
            'host' => env('LARAVEL_WEBSOCKETS_EXTERNAL_HOST',  "127.0.0.1"),
            'port' => env('LARAVEL_WEBSOCKETS_EXTERNAL_PORT', 6001),
            'key' => env('PUSHER_APP_KEY', "staging"),
            'secret' => env('PUSHER_APP_SECRET', "staging"),
            'path' => env('PUSHER_APP_PATH'),
            'enable_client_messages' => true,
            'enable_statistics' => true,
            'encrypted' => (env("APP_ENV") === "local" || env("APP_ENV") === "dev") ? false : true,
        ],
    ],

    /*
     * This class is responsible for finding the apps. The default provider
     * will use the apps defined in this config file.
     *
     * You can create a custom provider by implementing the
     * `AppProvider` interface.
     */
    'app_provider' => BeyondCode\LaravelWebSockets\Apps\ConfigAppProvider::class,

    /*
     * This array contains the hosts of which you want to allow incoming requests.
     * Leave this empty if you want to accept requests from all hosts.
     */
    'allowed_origins' => [],

    /*
     * The maximum request size in kilobytes that is allowed for an incoming WebSocket request.
     */
    'max_request_size_in_kb' => 250,

    /*
     * This path will be used to register the necessary routes for the package.
     */
    'path' => 'websockets-monitor',

    /*
     * Dashboard Routes Middleware
     *
     * These middleware will be assigned to every dashboard route, giving you
     * the chance to add your own middleware to this list or change any of
     * the existing middleware. Or, you can simply stick with this list.
     */
    'middleware' => [
        'web',
        // Authorize::class
        // "master",
        // "auth"
    ],

    'statistics' => [
        /*
         * This model will be used to store the statistics of the WebSocketsServer.
         * The only requirement is that the model should extend
         * `WebSocketsStatisticsEntry` provided by this package.
         */
        'model' => \BeyondCode\LaravelWebSockets\Statistics\Models\WebSocketsStatisticsEntry::class,

        /**
         * The Statistics Logger will, by default, handle the incoming statistics, store them
         * and then release them into the database on each interval defined below.
         */
        'logger' => BeyondCode\LaravelWebSockets\Statistics\Logger\HttpStatisticsLogger::class,

        /*
         * Here you can specify the interval in seconds at which statistics should be logged.
         */
        'interval_in_seconds' => 60,

        /*
         * When the clean-command is executed, all recorded statistics older than
         * the number of days specified here will be deleted.
         */
        'delete_statistics_older_than_days' => 60,

        /*
         * Use an DNS resolver to make the requests to the statistics logger
         * default is to resolve everything to 127.0.0.1.
         */
        'perform_dns_lookup' => false,
    ],

    /*
     * Define the optional SSL context for your WebSocket connections.
     * You can see all available options at: http://php.net/manual/en/context.ssl.php
     */
    'ssl' => [
        /*
         * Path to local certificate file on filesystem. It must be a PEM encoded file which
         * contains your certificate and private key. It can optionally contain the
         * certificate chain of issuers. The private key also may be contained
         * in a separate file specified by local_pk.
         */
        'local_cert' => env('LARAVEL_WEBSOCKETS_SSL_LOCAL_CERT', null),
        'local_pk' => env('LARAVEL_WEBSOCKETS_SSL_LOCAL_PK', null),
        'passphrase' => env('LARAVEL_WEBSOCKETS_SSL_PASSPHRASE', null),
        "verify_peer" => (env("APP_ENV") === "local" || env("APP_ENV") === "dev") ? false : true,
        "allow_self_signed" => (env("APP_ENV") === "local" || env("APP_ENV") === "dev") ? true : false
    ],

    /*
     * Channel Manager
     * This class handles how channel persistence is handled.
     * By default, persistence is stored in an array by the running webserver.
     * The only requirement is that the class should implement
     * `ChannelManager` interface provided by this package.
     */
    'channel_manager' => \BeyondCode\LaravelWebSockets\WebSockets\Channels\ChannelManagers\ArrayChannelManager::class,
];
```
```bash
php artisan make:migration create_jobs_table
```
```php
        Schema::create('jobs', function (Blueprint $table) {
            $table->bigIncrements('id');
            $table->string('queue')->index();
            $table->longText('payload');
            $table->unsignedTinyInteger('attempts');
            $table->unsignedInteger('reserved_at')->nullable();
            $table->unsignedInteger('available_at');
            $table->unsignedInteger('created_at');
        });
```
```bash
vi config/broadcasting.php
```
```php
<?php

return [

    /*
    |--------------------------------------------------------------------------
    | Default Broadcaster
    |--------------------------------------------------------------------------
    |
    | This option controls the default broadcaster that will be used by the
    | framework when an event needs to be broadcast. You may set this to
    | any of the connections defined in the "connections" array below.
    |
    | Supported: "pusher", "ably", "redis", "log", "null"
    |
    */

    'default' => env('BROADCAST_DRIVER', 'pusher'),

    /*
    |--------------------------------------------------------------------------
    | Broadcast Connections
    |--------------------------------------------------------------------------
    |
    | Here you may define all of the broadcast connections that will be used
    | to broadcast events to other systems or over websockets. Samples of
    | each available type of connection are provided inside this array.
    |
    */

    'connections' => [
        'pusher' => [
            'driver' => 'pusher',
            'key' => env('PUSHER_APP_KEY'),
            'secret' => env('PUSHER_APP_SECRET'),
            'app_id' => env('PUSHER_APP_ID'),
            'options' => [
                'cluster' => env('PUSHER_APP_CLUSTER'),
                'host' => env("LARAVEL_WEBSOCKETS_INTERNAL_HOST", '127.0.0.1'),
                'port' => env('LARAVEL_WEBSOCKETS_INTERNAL_PORT', 6001),
                'scheme' => 'http',
                'useTLS' => false,
                'curl_options' => [
                    CURLOPT_SSL_VERIFYHOST => 0,
                    CURLOPT_SSL_VERIFYPEER => 0,
                ],
            ],
        ],

        'ably' => [
            'driver' => 'ably',
            'key' => env('ABLY_KEY'),
        ],

        'redis' => [
            'driver' => 'redis',
            'connection' => 'default',
        ],

        'log' => [
            'driver' => 'log',
        ],

        'null' => [
            'driver' => 'null',
        ],

    ],

];
```
```bash
vi routes/api.php
```
```php
use App\Http\Controllers\SocketsController;

Route::post("/sockets/connect", [SocketsController::class, "connect"]);
```
```bash
touch app/Http/Controllers/SocketsController.php
```
```php
<?php

declare(strict_types=1);

namespace App\Http\Controllers;

use Illuminate\Broadcasting\Broadcasters\PusherBroadcaster;
use Illuminate\Http\Request;
use Pusher\Pusher;

class SocketsController
{
    public function connect(Request $request)
    {
        $broadcaster = new PusherBroadcaster(
            new Pusher(
                env("PUSHER_APP_KEY"),
                env("PUSHER_APP_SECRET"),
                env("PUSHER_APP_ID"),
                []
            )
        );

        return $broadcaster->validAuthenticationResponse($request, []);
    }
}
```
```bash
vi .env

# BROADCAST_DRIVER=pusher

# PUSHER_APP_ID=staging
# PUSHER_APP_KEY=staging
# PUSHER_APP_SECRET=staging
# PUSHER_APP_CLUSTER=mt1

# LARAVEL_WEBSOCKETS_INTERNAL_HOST=websocket
# LARAVEL_WEBSOCKETS_INTERNAL_PORT=6001
# LARAVEL_WEBSOCKETS_EXTERNAL_HOST=127.0.0.1
# LARAVEL_WEBSOCKETS_EXTERNAL_PORT=6001
# LARAVEL_WEBSOCKETS_SSL_LOCAL_PK=
# LARAVEL_WEBSOCKETS_SSL_PASSPHRASE=
# LARAVEL_WEBSOCKETS_SSL_LOCAL_CERT=
```
```bash
docker-compose down
docker-compose up -d
```
```bash
# Push images to Dockerhub
vi docker-compose.yml
```
```yaml
  php:
    image: alansary/php:v1
  node:
    image: alansary/node:v1
  queue:
    image: alansary/queue:v1
  websocket:
    image: alansary/websocket:v1
```
```bash
docker tag app_php:latest alansary/php:v1
docker tag app_node:latest alansary/node:v1
docker tag app_queue:latest alansary/queue:v1
docker tag app_websocket:latest alansary/websocket:v1
```
```bash
docker-compose down
docker compose up -d
```
```bash
docker login
docker-compose push
```
```bash
docker-compose down
docker system prune
docker volume prune
```