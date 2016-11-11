This page shows how to quickly set up the Steeltoe Configuration extension in an ASP.NET Core application for accessing configuration values served by a [Spring Cloud Config](http://cloud.spring.io/spring-cloud-config/) Config Server.

### Step 0: Run a Config Server

We will need a running Config Server from which our application can request configuration. To run the Config Server, we'll also need the Java Development Kit (JDK). [Download](http://www.oracle.com/technetwork/java/javase/downloads/index.html) the JDK from the Oracle website and follow the instructions to [install](http://docs.oracle.com/javase/8/docs/technotes/guides/install/install_overview.html) it. Then open Control Panel, search for and select "Edit the system environment variables", and add a system environment variable called `JAVA_HOME`, with the JDK installation's path (something like `C:\Program Files\Java\jdk1.8.0_91`) as the value.

We will need [Git](https://git-scm.com) in order to obtain the code for the Config Server that we'll be using. If you haven't already installed Git, follow the [instructions](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git) to do so. You may wish to look at [Git for Windows](https://git-for-windows.github.io).

Now for the Config Server! Visit the [SteeltoeOSS/configserver](https://github.com/SteeltoeOSS/configserver) repository on GitHub. Copy the URL given in the "Clone or download" dropdown and use `git clone` to get a local copy of the repository:

```
> git clone git@github.com:SteeltoeOSS/configserver.git
```

When the clone operation has finished, `cd` into the Config Server's directory. The Config Server is built using [Apache Maven](https://maven.apache.org), and the project that we've just downloaded includes a Maven "wrapper" file (`mvnw.cmd`) that can be used in place of an actual Maven installation to run build commands. Start up the Config Server with the `mvnw spring-boot:run` command:

```
configserver> mvnw spring-boot:run

(...Startup output...)

2016-05-25 15:25:44.295  INFO 98987 --- [           main] s.b.c.e.t.TomcatEmbeddedServletContainer : Tomcat started on port(s): 8888 (http)
2016-05-25 15:25:44.298  INFO 98987 --- [           main] o.s.c.c.server.ConfigServerApplication   : Started ConfigServerApplication in 2.634 seconds (JVM running for 58.236)
```

### Step 1: Add the Steeltoe Configuration dependency

[Generate](https://docs.asp.net/en/latest/client-side/yeoman.html) a new ASP.NET Core application using Yeoman. When the generator asks what type of application you want to create, select the "Web Application Basic [without Membership and Authorization]" option. For our example purposes, call the application &#8220;Foo&#8221;. Then create a `nuget.config` file, and within it, list the Steeltoe feeds:

```
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <packageSources>
    <add key="SteeltoeMaster" value="https://www.myget.org/F/steeltoemaster/api/v3/index.json" />
    <add key="SteeltoeDev" value="https://www.myget.org/F/steeltoedev/api/v3/index.json" />
    <add key="NuGet" value="https://api.nuget.org/v3/index.json" />
  </packageSources>
</configuration>
```

In the `dependencies` block of our `project.json` file, add the `Steeltoe.Extensions.Configuration.ConfigServer` dependency:

```
  "dependencies": {
    ...
    "Microsoft.VisualStudio.Web.BrowserLink.Loader": "14.0.0-rc2-final",
    "Steeltoe.Extensions.Configuration.ConfigServer": "1.0.0-dev-*"
  },
```

### Step 2: Configure the Config Server settings

Next, open `appsettings.json`. We need to specify a couple of settings for Steeltoe Configuration to locate the Config Server and then to request our application's specific configuration from it:

```
{
  "spring": {
    "application": {
      "name": "foo"
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

The other property, `spring.cloud.config.uri`, tells a Steel Toe Configuration client application where to locate its Config Server. We give this the URL of the Config Server that we have running on port 8888.

### Step 3: Add the Config Server configuration provider

In the constructor of our `Startup.cs`, where we use the `ConfigurationBuilder`, we need to add the Config Server as a configuration source.

```
using Steeltoe.Extensions.Configuration;

namespace Foo
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

(Don't forget the `using` statement at the top.)

Now drop down to `ConfigureServices()` and add the Config Server to our `IServiceCollection`.

```
        // This method gets called by the runtime. Use this method to add services to the container.
        public void ConfigureServices(IServiceCollection services)
        {
            services.AddConfigServer(Configuration);

            // Add framework services.
            services.AddMvc();
        }
```

This method also takes care of adding the `IOptions` service and adds `IConfigurationRoot` as a service. This will become important in the next step, which is...

### Step 4: Use configuration in the application

Open our `HomeController.cs` file. We need to give this controller an `IConfigurationRoot` property and a constructor to proceed further:

```
using Microsoft.Extensions.Configuration;
using Steeltoe.Extensions.Configuration.ConfigServer;

namespace Foo.Controllers
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

That's it! Run `dotnet restore` to install all of our dependencies, then run the application:

```
Foo> dotnet restore
...
Feeds used:
    https://www.myget.org/F/steeltoemaster/api/v3/index.json
    https://www.myget.org/F/steeltoedev/api/v3/index.json
    https://api.nuget.org/v3/index.json
Foo> dotnet run
...
Now listening on: http://localhost:5000
Application started. Press Ctrl+C to shut down.
```

And in a browser, visit http://localhost:5000/Home/ConfigServer. You should see something like this:

![config-server UI](https://github.com/SteeltoeOSS/Tutorials/blob/master/images/getting-started/configuration.png)

