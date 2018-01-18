---
layout: post
title:  "AspNetCore WebApi, Swashbuckle Swagger, OAuth2 AzureActiveDirectory example"
date:   2018-01-17
categories: dotnetcore
---
# Create the AspNetCore WebApi

```
mkdir Com.Bekijkhet.MySwashbuckleAADWebApi
cd Com.Bekijkhet.MySwashbuckleAADWebApi
dotnet new webapi
```

Test the application by starting it and point the browser to http://localhost:5000/api/values

```
C:\dotnet\Com.Bekijkhet.MySwashbuckleAADWebApi>dotnet run
Hosting environment: Production
Content root path: C:\dotnet\Com.Bekijkhet.MySwashbuckleAADWebApi
Now listening on: http://localhost:5000
Application started. Press Ctrl+C to shut down.
```

![Add AAD User]({{ "/assets/20180117noauth.png" | absolute_url }})

It works without authentication.

# Add ActiveDirectory Authentication

We have to create some Active Directory objects that we use in this example.

First we create a user. We go to the active directory portal and add a user:

![Add AAD User]({{ "/assets/20180117adduser.jpg" | absolute_url }})

![Add AAD User]({{ "/assets/20180117adduser2.jpg" | absolute_url }})

Remember the username and the temporary password. I will refer to them as < aadusername > and < aaduserpassword > in this blog.

mydemo@broersa.onmicrosoft.com
Vasa2076

Click Create.

We also need two Active Directory App Registrations

![Add AAD App Registration]({{ "/assets/20180117addappreg.jpg" | absolute_url }})

First add the client application:

![Add AAD App Registration]({{ "/assets/20180117addappreg2.jpg" | absolute_url }})

Click Create.

Open the application.

Note down the Application ID as < clientappid >

Change the Reply URLs to `http://localhost:5000/swagger/o2c.html`

Create a Key, name it mykey set the duration to Never Expires and click on Save. Now note down the Value as < clientappkey >

Now click on the Manifest button and edit the selected line as is in the image:

![Add AAD App Registration]({{ "/assets/20180117addappreg4.jpg" | absolute_url }})

Now add the webapi application:

![Add AAD App Registration]({{ "/assets/20180117addappreg3.jpg" | absolute_url }})

Open the application.

Note down the Application ID as < webapiappid >

Create a Key, name it mykey set the duration to Never Expires and click on Save. Now note down the Value as < webapiappkey >

Click on Properties and note down the App ID URI as < webapiresourceid >

Now go back to mydemoapp and grant permissions of this application to the mywebapi application:

![Add AAD App Registration]({{ "/assets/20180117addappreg5.jpg" | absolute_url }})

Follow the steps and click Create.

![Add AAD App Registration]({{ "/assets/20180117addappreg6.jpg" | absolute_url }})

Set the options as the image and click Select. Than click Done.

Now go to the Active Directory main page and select Properties and note down the Directory ID as < tenantid >

![Add AAD User]({{ "/assets/20180117tenant.png" | absolute_url }})

Ok so far so good.

# Add Authentication to the AspNetCore WebApi application

First add the line [Authorize] to the ValuesController.cs. Also add a for loop to print the claimset. Also add the folowing usings:
``` csharp
using Microsoft.AspNetCore.Authorization;
```

``` csharp
namespace Com.Bekijkhet.MySwashbuckleAADWebApi.Controllers
{
    [Route("api/[controller]")]
    [Authorize]
    public class ValuesController : Controller
    {
        // GET api/values
        [HttpGet]
        public IEnumerable<string> Get()
        {
            foreach (var claim in User.Claims)
            {
                Console.WriteLine($"{claim.Type}\t{claim.Value}");
            }

            return new string[] { "value1", "value2" };
        }

```

Edit Startup.cs so that it looks like the following:

``` csharp
        public void ConfigureServices(IServiceCollection services)
        {
            services.AddMvc();

            services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
                .AddJwtBearer(options =>
                {
                    options.Audience = "< webapiresourceid >";
                    options.Authority = "https://login.windows.net/< tenantid >");
                });

        }

        // This method gets called by the runtime. Use this method to configure the HTTP request pipeline.
        public void Configure(IApplicationBuilder app, IHostingEnvironment env)
        {
            if (env.IsDevelopment())
            {
                app.UseDeveloperExceptionPage();
            }

            app.UseAuthentication();

            app.UseMvc();
        }
```
Also add the following usings:
``` csharp
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.AspNetCore.Authentication.OpenIdConnect;
```

Now try to run the application again:

```
C:\dotnet\Com.Bekijkhet.MySwashbuckleAADWebApi>dotnet run
Hosting environment: Production
Content root path: C:\dotnet\Com.Bekijkhet.MySwashbuckleAADWebApi
Now listening on: http://localhost:5000
Application started. Press Ctrl+C to shut down.
```

![Add AAD User]({{ "/assets/20180117auth.png" | absolute_url }})

We get an error...

# Add Swashbuckle (Swagger) to the story.

```
dotnet add package Swashbuckle.AspNetCore -v 1.1.0
```

Add the following lines to Startup.cs ConfigureServices method:

``` csharp
services.AddSwaggerGen(c =>
{
    c.SwaggerDoc("v1", new Info { Title = "My API", Version = "v1" });
});
```

Add the following lines to Startupcs Configure method:

``` csharp
app.UseSwagger();
app.UseSwaggerUI(c =>
{
    c.SwaggerEndpoint("/swagger/v1/swagger.json", "My API V1");
});
```

Also add the using:
``` csharp
using Swashbuckle.AspNetCore.Swagger;
```

When we restart the application we have Swashbuckle in place:

```
C:\dotnet\Com.Bekijkhet.MySwashbuckleAADWebApi>dotnet run
Hosting environment: Production
Content root path: C:\dotnet\Com.Bekijkhet.MySwashbuckleAADWebApi
Now listening on: http://localhost:5000
Application started. Press Ctrl+C to shut down.
```

Check the url: http://localhost:5000/swagger

![Add AAD User]({{ "/assets/20180117swagger.png" | absolute_url }})

When we try the /api/Values we get the same 401 error.

# Add the AAD Oauth2 to the mix

Change the Startup.cs and replace in the ConfigureServices method the previous added with this:

``` csharp
services.AddSwaggerGen(c =>
{
    c.SwaggerDoc("v1", new Info { Title = "My API", Version = "v1" });
    c.AddSecurityDefinition("oauth2", new OAuth2Scheme
    {
        Type = "oauth2",
        Flow = "implicit",
        AuthorizationUrl = "https://login.microsoftonline.com/< tenantid >/oauth2/authorize",
        Scopes = new Dictionary<string, string>
        {
            { "read", "Access read operations" },
            { "write", "Access write operations" }
        },
        TokenUrl = "https://login.microsoftonline.com/< tenantid >/oauth2/token"
    });
});
```
And replace the following in the Configure method:

``` csharp
app.UseSwaggerUI(c =>
{
    c.ConfigureOAuth2("< clientappid >", "< clientappkey >", "< webapiresourceid >", "mywebapi", " ", additionalQueryStringParameters: new Dictionary<string, string>
    {
        { "resource", "< webapiresourceid >" }
    });
    c.SwaggerEndpoint("/swagger/v1/swagger.json", "My API V1");
});
```

When we now start the application:

```
C:\dotnet\Com.Bekijkhet.MySwashbuckleAADWebApi>dotnet run
Hosting environment: Production
Content root path: C:\dotnet\Com.Bekijkhet.MySwashbuckleAADWebApi
Now listening on: http://localhost:5000
Application started. Press Ctrl+C to shut down.
```

And go to the swagger endpoint: http://localhost:5000/swagger we see an Authorize button. When we click on this butten we get a popup:

![Add AAD User]({{ "/assets/20180117swaggerauth.png" | absolute_url }})

Select read and write and click on Authorize. Now we get a login screen from Azure Active Directory. Here we can enter our created user < aadusername > and authorize with the temporary password < aaduserpassword >. Sorry for the Dutch prompts.

![Add AAD User]({{ "/assets/20180117swaggerauth2.png" | absolute_url }})
![Add AAD User]({{ "/assets/20180117swaggerauth3.png" | absolute_url }})

Next we get the question to set our definitive password. After that the application asks our permission press Accept.

![Add AAD User]({{ "/assets/20180117swaggerauth4.png" | absolute_url }})

Now we are signed in and we can make the secure call via the swagger UI.

![Add AAD User]({{ "/assets/20180117swaggerauth5.png" | absolute_url }})

Also the claims are printed by the application:

```
aud     https://x.onmicrosoft.com/x
iss     https://sts.windows.net/x/
iat     x
nbf     x
exp     x
http://schemas.microsoft.com/claims/authnclassreference 1
aio     x
http://schemas.microsoft.com/claims/authnmethodsreferences      pwd
appid   x
appidacr        0
ipaddr  a.b.c.d
name    My Demo User
http://schemas.microsoft.com/identity/claims/objectidentifier   x
http://schemas.microsoft.com/identity/claims/scope      user_impersonation
http://schemas.xmlsoap.org/ws/2005/05/identity/claims/nameidentifier    x
http://schemas.microsoft.com/identity/claims/tenantid   x
http://schemas.xmlsoap.org/ws/2005/05/identity/claims/name      mydemo@x.onmicrosoft.com
http://schemas.xmlsoap.org/ws/2005/05/identity/claims/upn       mydemo@x.onmicrosoft.com
uti     x
ver     1.0
```

Comments are welcome via my twitter account.

The code can be found on my github:
[https://github.com/broersa/Com.Bekijkhet.MySwashbuckleAADWebApi](https://github.com/broersa/Com.Bekijkhet.MySwashbuckleAADWebApi)
