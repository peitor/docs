---
lodash: true
---

## ASP.NET Web API OWIN Tutorial

<div class="package">
  <blockquote>
    <a href="https://docs.auth0.com/auth0-aspnet-owin/master/create-package?path=examples/WebApi&type=owin@@account.clientParam@@" class="btn btn-lg btn-success btn-package" style="text-transform: uppercase; color: white">
      <span style="display: block">Download a Seed project</span>
      <% if (account.userName) { %> 
        <span class="smaller" style="display:block; font-size: 11px">with your Auth0 API Keys already set and configured</span>
      <% } %>
    </a> 
  </blockquote>
</div>

**Otherwise, please follow the steps below to configure your existing ASP.NET Web API OWIN app to use it with Auth0.**

### 1. Setup NuGet dependencies

Update this package:
````Powershell
Update-Package System.IdentityModel.Tokens.Jwt
```

Install the following NuGet package:
````Powershell
Install-Package Microsoft.Owin.Security.Jwt
```

### 2. Configure Json Web Token authentication

Open the **Startup.cs** class located inside the **App_Start** folder.

Add the following using statements
````CSharp
using Microsoft.Owin.Security.Jwt;
using System.Web.Http;
using Microsoft.Owin.Security.DataHandler.Encoder;
using Microsoft.Owin.Security;
using WebConfigurationManager = System.Web.Configuration.WebConfigurationManager;
```

Update the `Configuration` method with the following code:
````CSharp
var issuer = WebConfigurationManager.AppSettings["Auth0Domain"];
var audience = WebConfigurationManager.AppSettings["Auth0ClientID"];
var secret = TextEncodings.Base64Url.Decode(
    WebConfigurationManager.AppSettings["Auth0ClientSecret"]);

// Api controllers with an [Authorize] attribute will be validated with JWT
app.UseJwtBearerAuthentication(
    new JwtBearerAuthenticationOptions
    {
        AuthenticationMode = AuthenticationMode.Active,
        AllowedAudiences = new[] { audience },
        IssuerSecurityTokenProviders = new IIssuerSecurityTokenProvider[]
        {
            new SymmetricKeyIssuerSecurityTokenProvider(issuer, secret)
        },
    });
```

### 3. Update the web.config file with your app's credentials
Open the **web.config** file located at the solution's root.

Add the following entries as children of the `<appSettings>` element. 
````xml
<add key="Auth0Domain" value="https://<%= account.namespace %>/"/>
<add key="Auth0ClientID" value="<%= account.clientId %>"/>
<add key="Auth0ClientSecret" value="<%= account.clientSecret %>"/>
```

### 4. Securing your API
All you need to do now is add the `[System.Web.Http.Authorize]` attribute to the controllers/actions for which you want to verify that users are authenticated.

### 5. You've nailed it.

Now you have both your FrontEnd and Backend configured to use Auth0. Congrats, you're awesome!

### Optional Steps
#### Configuring CORS

To configure CORS you need to first install the `Microsoft.Owin.Cors` NuGet package. Once that is done you can simply update the configuration method in your **Startup** class as shown in the following snippet:
````CSharp
app.UseCors(CorsOptions.AllowAll);
```

> Important: The CORS policy used in the previous code snippet is not recommended for a production. You will need to implement the `ICorsPolicyProvider` interface.

#### Working with claims
If you want to read/modify the claims that are populated based on the JWT you can use the following extensibility points:
````CSharp
string token = string.Empty;

// Api controllers with an [Authorize] attribute will be validated with JWT
app.UseJwtBearerAuthentication(
    new JwtBearerAuthenticationOptions
    {
        AuthenticationMode = AuthenticationMode.Active,
        AllowedAudiences = new[] { audience },
        IssuerSecurityTokenProviders = new IIssuerSecurityTokenProvider[]
        {
            new SymmetricKeyIssuerSecurityTokenProvider(issuer, secret)
        },
        Provider = new OAuthBearerAuthenticationProvider
        {
            OnRequestToken = context =>
            {
                token = context.Token;
                return Task.FromResult<object>(null);
            },
            OnValidateIdentity = context =>
            {
                if (!string.IsNullOrEmpty(token))
                {
                    var notPadded = token.Split('.')[1];
                    var claimsPart = Convert.FromBase64String(
                        notPadded.PadRight(notPadded.Length + (4 - notPadded.Length % 4) % 4, '='));
                
                    var obj = JObject.Parse(Encoding.UTF8.GetString(claimsPart, 0, claimsPart.Length));

                    // simple, not handling specific types, arrays, etc.
                    foreach (var prop in obj.Properties().AsJEnumerable())
                    {
                        if (!context.Ticket.Identity.HasClaim(prop.Name, prop.Value.Value<string>()))
                        {
                            context.Ticket.Identity.AddClaim(new Claim(prop.Name, prop.Value.Value<string>()));
                        }
                    }
                }

                return Task.FromResult<object>(null);
            }
        }
    });
```