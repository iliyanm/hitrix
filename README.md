![Check & test](https://github.com/coretrix/hitrix/workflows/Check%20&%20test/badge.svg)
[![codecov](https://codecov.io/gh/coretrix/hitrix/branch/main/graph/badge.svg)](https://codecov.io/gh/coretrix/hitrix)
[![Go Report Card](https://goreportcard.com/badge/github.com/coretrix/hitrix)](https://goreportcard.com/report/github.com/coretrix/hitrix)
[![MIT license](https://img.shields.io/badge/license-MIT-brightgreen.svg)](https://opensource.org/licenses/MIT)



# Hitrix

Hitrix is a web framework written in Go (Golang) and support Graphql and REST api.
Hitrix is based on top of [Gqlgen](https://gqlgen.com/]) and [Gin Framework](https://github.com/gin-gonic/gin) and it's high performance and easy to use

### Built-in features:

 * It supports all features of [Gqlgen](https://gqlgen.com/]) and [Gin Framework](https://github.com/gin-gonic/gin)
 * Integrated with [ORM](https://github.com/summer-solutions/orm)
 * Follows [Dependency injection](https://en.wikipedia.org/wiki/Dependency_injection) pattern
 * Provides many DI services that makes your live easier. You can read more about them [here](https://github.com/coretrix/hitrix#built-in-services)
 * Provides [Dev panel](https://github.com/coretrix/dev-frontend) where you can monitor and manage your application(monitoring, error log, db alters and so on)

## Installation

```
go get -u github.com/coretrix/hitrix
``` 
 
 
## Quick start
1. Run next command into your project's main folder and the graph structure will be created
```
go run github.com/99designs/gqlgen init
```


2. Create `cmd` folder into your project and file called `main.go`

Put the next code into the file:
```go
package main

import (
	"github.com/coretrix/hitrix"
	"github.com/gin-gonic/gin"
	
	"your-project/graph" //path you your graph
	"your-project/graph/generated" //path you your graph generated folder
)

func main() {
	s, deferFunc := hitrix.New(
		"app-name", "your secret",
	).Build()
    defer deferFunc()
	s.RunServer(9999, generated.NewExecutableSchema(generated.Config{Resolvers: &graph.Resolver{}}),  func(ginEngine *gin.Engine) {
		//here you can register all your middlewares
	})
}

```

You can register different DI services and dev-panel before you build your server.
For example your main.go file can looks like that:

```go
package main

import (
	"github.com/coretrix/hitrix"
	"github.com/coretrix/hitrix/service/registry"
	"your-project/entity"
	"your-project/graph"
	"your-project/graph/generated"
	"github.com/coretrix/hitrix/pkg/middleware"
	"github.com/gin-gonic/gin"
)

func main() {
	s, deferFunc := hitrix.New(
		"app-name", "your secret",
	).RegisterDIService(
		registry.ServiceProviderErrorLogger(), //register redis error logger
		registry.ServiceProviderConfigDirectory("../config"), //register config service. As param you should point to the folder of your config file
		registry.ServiceDefinitionOrmRegistry(entity.Init), //register our ORM and pass function where we set some configurations 
		registry.ServiceDefinitionOrmEngine(), //register our ORM engine for background processes
		registry.ServiceDefinitionOrmEngineForContext(), //register our ORM engine per context used in foreground processes
		registry.ServiceProviderJWT(), //register JWT DI service
		registry.ServiceProviderPassword(), //register pasword DI service
	).
    RegisterDevPanel(&entity.AdminUserEntity{}, middleware.Router, nil). //register our dev-panel and pass the entity where we save admin users, the router and the third param is used for the redis stream pool if its used
    Build()
    defer deferFunc()

	s.RunServer(9999, generated.NewExecutableSchema(generated.Config{Resolvers: &graph.Resolver{}}),  func(ginEngine *gin.Engine) {
		middleware.Cors(ginEngine)
	})
}

```
Now I will explain the main.go file line by line
 1. We create **New** instance of Hitrix and pass app name and a secret that is used from our security services 
 2. We register some DI services
    
    2.1. Global DI service for error logger. It will be used for error handler as well in case of panic
        If you register SlackApi error logger also it will send messages to slack channel
    
    2.2. Global DI service that loads config file
    
    2.3. Global DI service that initialize our ORM registry
    
    2.4. Global DI ORM engine used in background processes
    
    2.5. Request DI ORM engine used in foreground processes
    
    2.6. Global DI JWT service used by dev panel
    
    2.7. Global DI Password service used by dev-panel
 3. Register dev panel   
    
    3.1. First param is the entity where we gonna keep admin users
    
    3.2. The router used by dev panel
    
    3.3. If you use redis stream pool you should pass it's name as third param
 4. We run the server on port `9999`, pass graphql resolver and as third param we pass all middlewares we need.   
As you can see in our example we register only Cors middleware
    
If you want to register dev panel I will show you example AdminUserEntity
```go
package entity

import (
	"github.com/summer-solutions/orm"
)

type AdminUserEntity struct {
	orm.ORM   `orm:"table=admin_users;redisCache"`
	ID        uint64
	Email     string `orm:"unique=Email"`
	Password  string

	UserEmailIndex *orm.CachedQuery `queryOne:":Email = ?"`
}

func (e *AdminUserEntity) GetUsername() string {
	return e.Email
}

func (e *AdminUserEntity) GetPassword() string {
	return e.Password
}

```

After that you should register it to the `entity.Init` function
```go
package entity

import "github.com/summer-solutions/orm"

func Init(registry *orm.Registry) {
	registry.RegisterEntity(
		&AdminUserEntity{},
	)
}

```

Please execute this alter into your database
```sql

create table admin_users
(
    ID       bigint unsigned auto_increment primary key,
    Email    varchar(255) null,
    Password varchar(255) null,
    constraint Email unique (Email)
) charset = utf8mb4;


```

After that you can make GET request to http://localhost:9999/dev/create-admin/?Username=contact@coretrix.com&Password=coretrix
This will generate sql query that should be executed into your database to create new user for dev panel

### Defining DI services
#### We have two types of DI services - Global and Request services
Global services are singletons created once for the whole application
Request services are singletons created once per request

If you want to access the registered DI services you can do in in that way:
```
service.DI().App() //access the app
service.DI().Config() //access config
service.DI().OrmEngine() //access global orm engine
service.DI().OrmEngineForContext() //access reqeust orm engine
service.DI().JWT() //access JWT
service.DI().Password() //access JWT
//...and so on
```

#### You are able to create and register your own services in next way:
```go

func ServiceDefinitionMyService() *ServiceDefinition {
	return &ServiceDefinition{
		Name:   "my_service",
		Global: true,
		Build: func(ctn di.Container) (interface{}, error) {
			return &yourService{}, nil
		},
	}
}

```

And you have to register `ServiceDefinitionMyService()` in your `main.go` file



Now you can access this service in your code using:

```go
import (
    "github.com/coretrix/hitrix"
)

func SomeResolver(ctx context.Context) {

    service.HasService("my_service") // return true
    
    // return error if Build function returned error
    myService, has, err := service.GetServiceSafe("my_service") 
    // will panic if Build function returns error
    myService, has := service.GetServiceOptional("my_service") 
    // will panic if service is not registered or Build function returned errors
    myService := service.GetServiceRequired("my_service") 

    // if you registered service with field "Global" set to false (request service)

    myContextService, has, err := hitrix.GetServiceForRequestSafe(ctx).Get("my_service_request")
    myContextService, has := hitrix.GetServiceForRequestOptional(ctx).Get("my_service_request") 
    myContextService := hitrix.GetServiceForRequestRequired(ctx).Get("my_service_request") 
}

```

It's a good practice to define one object to return all available services:

```go
package my_package
import (
    "github.com/coretrix/hitrix"
)



func MyService() MyService {
    return service.GetServiceRequired("service_key").(*MyService)
}


```

### Setting mode

#### APP_MODE environment variable
You can define hitrix mode using special environment variable "**APP_MODE**".

Hitrix provides by default four modes:

 * **hitrix.ModeLocal - local**
   * should be used on local development machine (developer laptop)
   * errors and stack trace is printed directly to system console
   * log level is set to Debug level
   * log is formatted using human friendly console text formatter
   * Gin Framework is running in GinDebug mode
 * **hitrix.ModeTest - test**
   * should be used when you run your application tests
 * **hitrix.ModeDemo - demo**
   * should be used on your demo server
 * **hitrix.ModeProd - prod**
   * errors and stack trace is printed only using Log
   * log level is set to Warn level
   * log is formatted using json formatter   
    
Mode is just a string. You can define any name you want. Remember that every mode that you create
follows **hitrix.ModeProd** rules explained above.
    
    
In code you can easly check current mode using one of these methods:    

```go
service.DI().App().Mode()
service.DI().App().IsInLocalMode()
service.DI().App().IsInProdMode()
service.DI().App().IsInMode("my_mode")
```

#### APP_CONFIG_FOLDER environment variable
There are another important environment variable called `APP_CONFIG_FOLDER`
You can set path to your config folder for your demo, prod or any other environment

#### Environment variables in config file
Its good practice to keep your secrets like database credentials and so on out of the repository.
Our advice is to keep them like environment variables and call them into config.yaml file
For example your config can looks like this:
```yaml
orm:
  default:
    mysql: ENV[DEFAULT_MYSQL]
    redis: ENV[DEFAULT_REDIS]
    locker: default
    local_cache: 1000
```
where `DEFAULT_MYSQL` and `DEFAULT_REDIS` are env variables and our framework will automatically replace `ENV[DEFAULT_MYSQL]` and `ENV[DEFAULT_REDIS]` with the right values

Also we check if there is .env.XXX file in main config folder where XXX is the value of the APP_MODE.
If there is for example .env.local we are reading those env variables and merge them with config.yaml how we presented above

### Running scripts

First You need to define script definition that implements hitrix.Script interface:

```go

type TestScript struct {}

func (script *TestScript) Code() string {
    return "test-script"
}

func (script *TestScript) Unique() bool {
    // if true you can't run more than one script at the same time
    return false
}

func (script *TestScript) Description() string {
    return "script description"
}

func (script *TestScript) Run(ctx context.Context, exit hitrix.Exit) {
    // put logic here
	if shouldExitWithCode2 {
        exit.Error()	// you can exit script and specify exit code
    }
}

```

Methods above are required. Optionally you can also implement these interfaces:

```go

// hitrix.ScriptInterval interface
func (script *TestScript) Interval() time.Duration {                                                    
    // run script every minute
    return time.Minute 
}

// hitrix.ScriptIntervalOptional interface
func (script *TestScript) IntervalActive() bool {                                                    
    // only run first day of month
    return time.Now().Day() == 1
}

// hitrix.ScriptIntermediate interface
func (script *TestScript) IsIntermediate() bool {                                                    
    // script is intermediate, for example is listening for data in chain
    return true
}

// hitrix.ScriptOptional interface
func (script *TestScript) Active() bool {                                                    
    // this script is visible only in local mode
    return DIC().App().IsInLocalMode()
}

```

Once you defined script you can run it using RunScript method:

```go
package main
import "github.com/coretrix/hitrix"

func main() {
	hitrix.New("app_name", "your secret").Build().RunScript(&TestScript{})
}
``` 


You can also register script as dynamic script and run it using program flag:

```go
package main
import "github.com/coretrix/hitrix"

func main() {
	
    hitrix.New("app_name", "your secret").RegisterDIService(
        &registry.ServiceDefinition{
            Name:   "my-script",
            Global: true,
            Script: true, // you need to set true here
            Build: func(ctn di.Container) (interface{}, error) {
                return &TestScript{}, nil
            },
        },
    ).Build()
}
``` 

You can see all available script by using special flag **-list-scripts**:

```shell script
./app -list-scripts
```

To run script:

```shell script
./app -run-script my-script
```


### Built-in services

#### App 
This service contains information about the application like MODE and so on

#### Config
This service provides you access to your config file. We support only YAML file
When you register the service `registry.ServiceProviderConfigDirectory("../config")`
you should provide the folder where are your config files
The folder structure should looks like that
```
config
 - app-name
    - config.yaml
 - hitrix.yaml #optional config where you can define some settings related to built-in services like slack service
```

#### ORM Engine  
Used to access ORM in background scripts. It is one instance for the whole script

You can register it in that way:
`registry.ServiceDefinitionOrmEngine()`

#### ORM Engine Context
Used to access ORM in foreground scripts like API. It is one instance per every request

You can register it in that way:
`registry.ServiceDefinitionOrmEngineForContext()`

#### Error Logger
Used to save unhandled errors in error log. It can be used to save custom errors as well.
If you have setup Slack service you also gonna receive notifications in your slack

You can register it in that way:
`registry.ServiceProviderErrorLogger()`

#### SlackAPI
Gives you ability to send slack messages using slack bot. Also it's used to send messages if you use our ErrorLogger service.
The config that needs to be set in hitrix.yaml is:

```yaml
slack:
    token: "your token"
    error_channel: "test" #optional, used by ErrorLogger
    dev_panel_url: "test" #optional, used by ErrorLogger

```

You can register it in that way:
`registry.ServiceDefinitionSlackAPI()`

#### JWT
You can use that service to encode and decode JWT tokens

You can register it in that way:
`registry.ServiceDefinitionJWT()`

#### Password
This service it can be used to hash and verify hashed passwords. It's use the secret provided when you make new Hitrix instance

You can register it in that way:
`registry.ServiceDefinitionPassword()`


### Validator
We support 2 types of validators. One of them is related to graphql and the other one is related to rest

#### Graphql validator
There are 2 steps that needs to be executed if you want to use this kind of validator

1. Add `directive @validate(rules: String!) on INPUT_FIELD_DEFINITION` into your `schema.graphqls` file

2. Call `ValidateDirective` into your main.go file
```go
config := generated.Config{Resolvers: &graph.Resolver{}, Directives: generated.DirectiveRoot{Validate: hitrix.ValidateDirective()} }

s.RunServer(4001, generated.NewExecutableSchema(config), func(ginEngine *gin.Engine) {
    commonMiddleware.Cors(ginEngine)
    middleware.Router(ginEngine)
})
```