# Docker files to define Kubernetes ready container images of your Laravel application with Bunnyshell

### So, you want to deploy your Laravel application in a Cloud Native way? Bunnyshell can do that for you!

<br><br>

## First of all, what do we mean by "cloud native way"?


If you're familiar with Laravel, you most likely are familiar with Laravel Sail and the "all-in-one" approach for building a dev container. Sail builds a Docker image that runs everything in one container (nginx/apache, php, node, crons, and the cli for artisan), similar to how a VM would work. 
While great for development purposes (because it watches for code changes and rebuilds artifacts), this "all-in-one" approach is not well suited for staging or production environments in cloud native architectures. 

<br><br>


### So what are we looking to achieve with our "cloud native" way?
- deploy to Kubernetes
- each component has a single concern
- allow independent scaling of aech component (web server, php-fpm, cron server, cli)
- make the application available through an ingress with HTTPS and valid certificate

<br><br>

### Considerations when deploying an application to containers:
When deploying your app to containers, you should have in mind at least these three very important aspects:
- the filesystem is not persitent. Consider it read-only.
- for each container image there might be more than one running container (eg for scaling)
- there is a load balancer / proxy / ingress in front of the containers

<br><br>

### And these are the implications of the three poins above:

- you need to store sessions in a store ( database or redis). You can't store sessions locally, in local filesystem
- send your logs to stdout/stderr. Do not logs on local disk.
- you need to force the app to generate HTTPS links/routes. The application is exposed through HTTPS, with a valid certificate, but TLS will be terminated at the ingress, and the request reaching the application will be plain HTTP. The application cannot rely on the protocol of the incomming request for generating URLs
- you need to trust proxies. Since the application will run  in pods and pe accesible through a proxy (ingress), it needs to accept traffic from proxies.

<br><br>

## Our approach


Bunyshell makes use of your Dockerfile and docker-compose.yaml to define envinments, to build container images for your apps, and then deploy those images to a Kubernetes cluster.

But instead of relying on the Sail container images, we will build a new set of slim container images, each with it's own narrow purpose, as inspired by Chris Vermeulen's great Laravel on Kubernetes series. https://chris-vermeulen.com/laravel-in-kubernetes/

<br>

### Here are the steps we will go through:
1. We add a new Dockerfile and docker-compose-bunnyshelll.yml to the project in order to define a set of "cloud native" containers.
2. We do a set of (optional but recomended) adjustments to the application, to make sure it runs correctly in a container environment
3. We import these containers in Bunnyshell and define an environment based on them
4. We define secrets and environment variables in Bunnyshell.
5. Deploy and enjoy

<br><br>

### Before we begin, you need to have:
- your Laravel application in a git repo
- your Bunnyshell account (free tier will do)


<br><br>

## STEP 0: The application and the repo

<br>

You can create a basic Laravel application and push it to a git repo with these commands:


```bash
curl -s "https://laravel.build/laravel-on-bunnyshell" | bash
cd laravel-on-bunyshell 
./vendor/bin/sail up

# so far we have a laravel app that does not use mysql
# let's add jetstream to use the database for user authentication
./vendor/bin/sail composer require laravel/jetstream
./vendor/bin/sail artisan jetstream:install inertia


git init
git add .
git commit -m "initial import"
git branch -M main
git remote add origin git@___your_remote_here______
git push -u origin main

```

<br><br>

## STEP 1: New Dockerfiles

<br>

As expained, we will not be using the Dockerfile and docker-compose.yaml files created by Sail, but we will create our own "cloud native" files. 
To make it easier, you can copy the files from this repo:

https://github.com/alexo-bunnyshell/laravel-kubernetes-dockerfiles 


Copy the following files and folders to your repo:
- Dockerfile
- docker-compose-bunnyshell.yml
- docker/*


You can download the files from GitHub with these commands:

```bash
curl -o docker-compose-bunnyshell.yml   https://raw.githubusercontent.com/alexo-bunnyshell/laravel-kubernetes-dockerfiles/main/docker-compose-bunnyshell.yml
curl -o Dockerfile   https://raw.githubusercontent.com/alexo-bunnyshell/laravel-kubernetes-dockerfiles/main/Dockerfile
curl -o docker/bunnyshell/nginx.conf.template   https://raw.githubusercontent.com/alexo-bunnyshell/laravel-kubernetes-dockerfiles/main/docker/bunnyshell/nginx.conf.template
```

These files define a number of containers:
- web-server (runs the webserver - nginx)
- fpm-php - runs the fpm server that listens for connections on port 9000 an runs PHP code
- cron - runs all the crons
- cli - usefull for running artisan commands
- frontend - just builds the frontend assests (npm run build) and terminates - will not be deployed, it's just an intermediate container in our multi stage Docker setup

This approach is inspired by Chris Vermeulen's great series https://chris-vermeulen.com/laravel-in-kubernetes/. I highly recommend you have an in-depth look at it.


Feel free to make the necessary adjustments to these files. For example, you might need to change the "npm run build" command.

Make sure you add the new files to the repo, commit and push.

```bash
git add .
git commit -m "add Bunyshell docker files"
git push -u origin main
```

<br><br>

## STEP 2: Adjust application (Optional)

<br>

As mentioned earlier, some adjustments to the application are necessary to make it work properly in containerized environments. Let's see how to do these adjsutments.


### Logging to `stdout`

In Kubernetes applications should just log to stdout and logs are handled by the cluster.
To logg to stdout, add a new channel in config/logging.php, like so:

```php
'stdout' => [
            'driver' => 'monolog',
            'level' => env('LOG_LEVEL', 'debug'),
            'handler' => StreamHandler::class,
            'formatter' => env('LOG_STDOUT_FORMATTER'),
            'with' => [
                'stream' => 'php://stdout',
            ],
        ],
```

Later, when defining envinronment variables we will need to set 
LOG_CHANNEL=stdout.

<br><br>

### Using a session store:


<br>

The application must be ready to run in more than one pod, so storing sessions info locally will not work. We need a database or redis to handle the sessions.

If you choose redis, you need

```bash 
sail composer require predis/predis
```

Later, when defining envinronment variables we will need to set 
SESSION_DRIVER=redis|database


Force HTTPS:
in app/Providers/AppServiceProvider.php
```php
<?php

namespace App\Providers;

# Add the Facade
use Illuminate\Support\Facades\URL;
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    /** All the rest */

    public function boot()
    {
        if( !$this->app->environment('local')) {
            URL::forceScheme('https');
        }
    }
}
```

### Trust proxies:

in /app/Http/Middleware/TrustProxies.php set <proxies>="\*"

```php
class TrustProxies extends Middleware
{
    /**
     * The trusted proxies for this application.
     *
     * @var array<int, string>|string|null
     */
    protected $proxies="*";

``` 

<br><br>  

## STEP 3: Create a new environment in Bunnyshell and import the containers

<br>

For this step, we will be working in the Bunnyshell UI. You need to have your git account connected to Bunnyshell (in "integrations" menu), so that Bunnyshell can access your application repository.

Basic steps are:
- create a new project (optional)
- create a new environment
- add an application to that environment. When creating the new application, specify your application repository, the branch ("main"), and "docker-compose-bunnyshell.yml" for 'Root application path'.
- review and finalize. You might want to remove the MySQL and Redis containers if you do not need them.

Here is a short video of how to do that, in case you are not familiar with Bunnyshell.

https://www.loom.com/share/6e6180003a6d4692b17e465f0fb2e7f1

<br><br>

## STEP 4: Add environment variables and secrets in the Bunnshell environment

<br>

Our containerized app will not be using .env files, but will instead use actual environment varialbes. We need to specify these variables and secrets in our environment config. Bunnyshell will make sure they are injected in the containers.

The easy way to set the environment variables in Bunnyshell is to edit you environment's configuration and add these lines in "environmentVariables" section:

```env
environmentVariables:
    LOG_CHANNEL=stdout
    SESSION_DRIVER=redis
    # or 
    # SESSION_DRIVER=database
    APP_DEBUG: '1'
    APP_ENV: local
    APP_KEY: bns_secret(____________YOUR_VALUE____________)
    APP_NAME: Laravel on Bunnyshell
    APP_URL: 'http://laravel-eaas.test'
    AWS_ACCESS_KEY_ID: null
    AWS_BUCKET: null
    AWS_DEFAULT_REGION: us-east-1
    AWS_SECRET_ACCESS_KEY: null
    AWS_USE_PATH_STYLE_ENDPOINT: ''
    BROADCAST_DRIVER: log
    CACHE_DRIVER: file
    DB_CONNECTION: mysql
    DB_DATABASE: bns_secret(____________YOUR_VALUE____________)
    DB_HOST: ____________YOUR_VALUE____________
    DB_PASSWORD: bns_secret(____________YOUR_VALUE____________)
    DB_PORT: '3306'
    DB_USERNAME: bns_secret(____________YOUR_VALUE____________)
    FILESYSTEM_DISK: local
    LOG_CHANNEL: stdout
    LOG_DEPRECATIONS_CHANNEL: null
    LOG_LEVEL: debug
    MAIL_ENCRYPTION: null
    MAIL_FROM_ADDRESS: hello@example.com
    MAIL_FROM_NAME: '${APP_NAME}'
    MAIL_HOST: mailhog
    MAIL_MAILER: smtp
    MAIL_PASSWORD: null
    MAIL_PORT: '1025'
    MAIL_USERNAME: null
    MEILISEARCH_HOST: 'http://meilisearch:7700'
    MEMCACHED_HOST: memcached
    PUSHER_APP_CLUSTER: mt1
    PUSHER_APP_ID: null
    PUSHER_APP_KEY: null
    PUSHER_APP_SECRET: null
    PUSHER_HOST: null
    PUSHER_PORT: '443'
    PUSHER_SCHEME: https
    QUEUE_CONNECTION: sync
    REDIS_HOST: ____________YOUR_____________
    REDIS_PASSWORD: null
    REDIS_PORT: '6379'
    SCOUT_DRIVER: meilisearch
    SESSION_DRIVER: database
    SESSION_LIFETIME: '120'
    VITE_PUSHER_APP_CLUSTER: '${PUSHER_APP_CLUSTER}'
    VITE_PUSHER_APP_KEY: '${PUSHER_APP_KEY}'
    VITE_PUSHER_HOST: '${PUSHER_HOST}'
    VITE_PUSHER_PORT: '${PUSHER_PORT}'
    VITE_PUSHER_SCHEME: '${PUSHER_SCHEME}'
``` 

Here is a short video of how to do that, in case you are not familiar with Bunnyshell.


https://www.loom.com/embed/ea8dcfd2870744debfd7ca079c0b3290

<br><br>

## STEP 5: Deploy and enjoy

<br>

Go ahead and deploy your new app/environment. 
It will take a few minutes for Bunnyshell to build the containers in your application and to deploy them to Kubernetes. Once the deployment is ready you can acceess your application.

## Benefits

Now that your application is containerized an you have an environment deployed with Bunnyshell, there are BENEFITS:

- you can clone the environment with just a few clicks
- you can set up ephemeral/preview environments so that whenever there is a pull request against your app branch, a new environment is automatically created to showcase the new code.\
- you can enable auto-updating of the environment: whenever there is a commit to the app branch, the containers are rebuilt and redeployed with the updated version.


