This page shows how to quickly set up the Steel Toe Configuration extension in an ASP.NET Core application for accessing configuration values served by a [Spring Cloud Services](https://network.pivotal.io/products/p-spring-cloud-services) Config Server service instance on [Pivotal Cloud Foundry&reg;](https://network.pivotal.io/products/pivotal-cf) (PCF). We'll use [PCF Dev](http://pivotal.io/pcf-dev) for this purpose; if you don't yet have PCF Dev installed, follow the [tutorial at pivotal.io](http://pivotal.io/platform/pcf-tutorials/getting-started-with-pivotal-cloud-foundry-dev/introduction) before proceeding with these instructions.

### Step 0: Create a Config Server Service Instance

We will need a Config Server service instance from which our application can request configuration. To make this Spring Cloud Services service available, you must start PCF Dev with the flag `-s scs`:

```
> cf dev start -s scs
...
Starting VM...
Provisioning VM...
Waiting for services to start...
46 out of 46 running
...
```

Run `cf marketplace` to view the available services:

```
> cf marketplace
Getting services from marketplace in org pcfdev-org / space pcfdev-space as user...
OK

service                       plans        description
p-circuit-breaker-dashboard   standard     Circuit Breaker Dashboard for Spring Cloud Applications
p-config-server               standard     Config Server for Spring Cloud Applications
...
```

Using the `cf create-service` command, create a new Config Server service instance. Use the `-c` flag to provide configuration parameters for the instance--the only parameter we'll need in this case is the URI of a Git repository from which the Config Server can retrieve configuration, so use the following command:

```
> cf create-service p-config-server standard config-server -c '{"git": {"uri": "https://github.com/spring-cloud-samples/cook-config" } }'
Creating service instance config-server in org pcfdev-org / space pcfdev-space as admin...
OK

Create in progress. Use 'cf services' or 'cf service config-server' to check operation status.
```

The Config Server service instance will take a little while to be provisioned. As the command output says, we can use the command `cf service config-server` to check on the instance's status; when it's been created, the command will give us the output `Status: create succeeded`.

```
> cf service config-server

Service instance: config-server
Service: p-config-server
Bound apps:
Tags:
Plan: standard
Description: Config Server for Spring Cloud Applications
Documentation url: http://docs.pivotal.io/spring-cloud-services/
Dashboard: https://spring-cloud-broker.local2.pcfdev.io/dashboard/p-config-server/37f28420-3074-49f9-bfc8-500a977c3ccb

Last Operation
Status: create succeeded
...
```

### Step 1: Add the Steel Toe Configuration dependency

[Generate](https://docs.asp.net/en/latest/client-side/yeoman.html) a new ASP.NET Core application using Yeoman. When the generator asks what type of application you want to create, select the "Web Application Basic [without Membership and Authorization]" option. Call the application &#8220;Steeltoe PCF Example&#8221;. Then create a `nuget.config` file, and within it, list the Steel Toe feeds:

```
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <packageSources>
    <add key="SteelToeMaster" value="https://www.myget.org/F/steeltoemaster/api/v3/index.json" />
    <add key="SteelToeDev" value="https://www.myget.org/F/steeltoedev/api/v3/index.json" />
    <add key="NuGet" value="https://api.nuget.org/v3/index.json" />
  </packageSources>
</configuration>
```

In the `dependencies` block of our `project.json` file, add the `Pivotal.Extensions.Configuration.ConfigServer` dependency:

```
  "dependencies": {
    ...
    "Microsoft.VisualStudio.Web.BrowserLink.Loader": "14.0.0",
    "Pivotal.Extensions.Configuration.ConfigServer": "1.0.0-dev-*"
  },
```

### Step 1.5:

Create a Cloud Foundry manifest for our application. It should be a file called `manifest.yml` and should look like so:

```
---
applications:
- name: steeltoe-pcf-example
  memory: 512M
  disk_quota: 1G
  random-route: true
  path: .
  buildpack: https://github.com/cloudfoundry-community/dotnet-core-buildpack
  services:
    - config-server
```

This sets up our application to use the Config Server service instance we've created. It also tells PCF Dev to stage this application using the [cloudfoundry-community/dotnet-core-buildpack](https://github.com/cloudfoundry-community/dotnet-core-buildpack).

### Step 2: Configure the Config Server settings

Next, open `appsettings.json`. We need to specify a setting for Steel Toe Configuration to request our application's specific configuration from the Config Server:

```
{
  "spring": {
    "application": {
      "name": "steeltoe-pcf-example"
    },
    "cloud": {
      "config": {
        "uri": "http://localhost:8888"
      }
    }
  },
...
}

```

Spring Cloud commonly uses `spring.application.name` to identify client applications. In the case of the Config Server, the files in the Config Server's Git or Subversion repository will include application names in their filenames, and the Server uses `spring.application.name` to determine which files in its repository contain configuration for our application.

The other property, `spring.cloud.config.uri`, is optional in our case. This tells a Steel Toe Configuration client application where to locate its Config Server. We give this a value of `http://localhost:8888` (the default port on which a Spring Cloud Config Server runs). As we'll soon see, however, this will be overridden when we bring in the Cloud Foundry Config Server, in:

### Step 3: Add the Config Server configuration provider

In the constructor of our `Startup.cs`, where we use the `ConfigurationBuilder`, we need to add the Config Server as a configuration source.

```
using Pivotal.Extensions.Configuration;

namespace Steeltoe_PCF_Example
{
    public class Startup
    {
        public Startup(IHostingEnvironment env)
        {
            var builder = new ConfigurationBuilder()
                .SetBasePath(env.ContentRootPath)
                .AddJsonFile("appsettings.json", optional: false, reloadOnChange: true)
                .AddJsonFile($"appsettings.{env.EnvironmentName}.json", optional: true)
                .AddEnvironmentVariables()

                .AddConfigServer(env);
            Configuration = builder.Build();
        }
```

Don't forget the `using` statement at the top. In fact, take special note of that `using` statement. Steel Toe's Cloud Foundry configuration provider parses the special `VCAP_APPLICATION` and `VCAP_SERVICES` environment variables provided to Cloud Foundry applications. When we push this application to PCF Dev, `VCAP_APPLICATION` will be set to contain application information (such as our application's space name, space ID, URIs, host, and port), and `VCAP_SERVICES` will be set to contain information for the services that are bound to the application (including a service instance's name, plan, tags, and connection information).

The connection information for our Config Server service instance, once it's made available to our application via the environment variables, will override what we specified in our `appsettings.json`. That setting is still useful to have for running an application locally against a local Config Server, but since we've placed the environment variables configuration provider higher in priority than `appsettings.json` (it's added _after_ `appsettings.json`), the information from the environment will override our hard-coded setting.

With the provider in place, we'll next add the Config Server to the set of services that we set up in the `ConfigureServices()` method.

```
        // This method gets called by the runtime. Use this method to add services to the container.
        public void ConfigureServices(IServiceCollection services)
        {
            services.AddConfigServer(Configuration);

            // Add framework services.
            services.AddMvc();
        }
```

The `AddConfigServer()` method also takes care of adding the `IOptions` service and adds `IConfigurationRoot` as a service. This will become important in the next step, which is...

### Step 4: Use configuration in the application

Open our `HomeController.cs` file. We need to give this controller an `IConfigurationRoot` property and a constructor to proceed further:

```
using Pivotal.Extensions.Configuration.ConfigServer;
using Microsoft.Extensions.Configuration;

namespace Steeltoe_PCF_Example.Controllers
{
    public class HomeController : Controller
    {

        private IConfigurationRoot Config { get; set; }

        public HomeController(IConfigurationRoot config)
        {
            Config = config;
        }
```

(Again, don't forget the `using` statements.)

We now have access to our configuration within the controller (the `Config` property). Next, let's add a `ConfigServer()` action. This action's view will display the value of a configuration property that we obtain from the Config Server, so let's set that value here:

```
        public IActionResult ConfigServer()
        {
            ViewData["Foo"] = Config["Foo"];
            return View();
        }
```

Create the `ConfigServer.cshtml` view in `Views/Home/`. It should look like this:

```
<h2>Configuration from the Spring Cloud Config Server</h2>

<p>Here is the value.</p>

<table width="50%">
  <tr>
    <th>Property</th>
    <th>Value</th>
  </tr>
  <tr>
    <th><em>Foo</em></td>
    <th><em>@ViewData["Foo"]</em></td>
  </tr>
</table>
```

### Step 5: Voila!

That's it! Run `dotnet restore` to install all of our dependencies:

```
Steeltoe-PCF-Example> dotnet restore
...
Feeds used:
    https://www.myget.org/F/steeltoemaster/api/v3/index.json
    https://www.myget.org/F/steeltoedev/api/v3/index.json
    https://api.nuget.org/v3/index.json
```

Then push the application to PCF Dev:

```
Steeltoe-PCF-Example> cf push
...

0 of 1 instances running, 1 starting
1 of 1 instances running

App started

...

     state     since                    cpu    memory      disk      details
#0   running   2016-07-14 03:24:25 PM   0.0%   0 of 512M   0 of 1G
```

In a browser, visit the path `/Home/ConfigServer` on the application. You should see something like this:

![Steeltoe-PCF-Example Config Server page]](images/getting-started/configuration-pcf.png)

