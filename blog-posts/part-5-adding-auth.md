Finally - we're at the point in this series where I show how to use Azure AD B2C to authenticate a user and then authorize that user to hit a Web API from a Xamarin.Forms application!

In other words - we're at the post with the very official, and very long, title of...

# Authenticating and Authorizing a Mobile App to Use a Web API via Azure AD B2C

In the first couple of posts, we learned what [Azure AD B2C is](https://msou.co/6z), how to [create a Tenant](https://msou.co/60) (which I found a bit tricky, I even created a [video](https://msou.co/7g) to help explain it), then took a quick detour to find out how to [invoke a Web API](https://msou.co/61) from a Xamarin.Forms app - and that's going to be our backing service which will be "protected". Then in the [final post before this one](https://msou.co/7j), we learned what everything is inside of Azure AD B2C... and how it relates together. And the info from that post is going to come in very handy here!

In order to get this all to work, there are 4 parts we have to go through.

1. Configure our Azure AD B2C tenant in the portal
1. Create the Azure AD B2C application within portal.
1. Modify the WebAPI application to only return data if it receives an authorization token with the appropriate scopes.
    1. Although in this quick demo case I'm going to ignore the scope - but don't do that in real life!
1. Modify the Xamarin.Forms app to request the token from Azure AD B2C and then send the authorization token on to the Web API.

Actually, there's a fifth part - and that's to down the beverage of your choice - possibly through a funnel.

I say that because this post is going to be long - and a bit dry. It will be a step-by-step guide on getting everything setup and then finally making a request to a backend service that needs authorization before returning any data.

A lot of these steps are "set 'em and forget 'em" ... so they're important to go over because ... well ... they're easy to forget.

## Step 1 - Configuring the Azure AD B2C Tenant

### Creating a Policy

We need to do some work in the portal. I'm going to go relatively fast through all of this as all of the concepts used here was explained in the [previous post](https://msou.co/7j).

First up - log in to the portal, and select the appropriate directory from the dropdown in the upper left corner under your name:

![](https://res.cloudinary.com/code-mill-technologies-inc/image/upload/bo_2px_solid_rgb:000000/v1512490151/Screen_Shot_2017-12-05_at_10.08.21_AM_raaiwg.png)

Once there, select the Azure AD B2C option from the menu on the far left side:

![](https://res.cloudinary.com/code-mill-technologies-inc/image/upload/bo_2px_solid_rgb:000000/v1512490254/Screen_Shot_2017-12-05_at_10.10.29_AM_x2icov.png)

We need to create a policy for the Azure AD B2C Tenant.

Select __Sign-up or sign-in policies__ from the left-hand menu.

![](https://res.cloudinary.com/code-mill-technologies-inc/image/upload/bo_2px_solid_rgb:000000/v1512491809/Screen_Shot_2017-12-05_at_10.36.28_AM_lv7ga7.png)

Then click __Add__ in the blade that comes up.

Here you're going to be able to configure quite a few options for the new policy.

![](https://res.cloudinary.com/code-mill-technologies-inc/image/upload/bo_2px_solid_rgb:000000/v1512491917/Screen_Shot_2017-12-05_at_10.38.19_AM_vwj7wo.png)

Give it a name you'll remember ... I chose _Generic Sign-up and Sign-In_.

Then under __Identity Providers__, there will only be one option __Email signup__... select that.

Email signup allows a user to sign up and sign in using their email address and a password of their choosing. (Eventually we'll add in some social authentication.)

The __Sign-up attributes__ declare what fields you want to have collected when a user registers for the app. Choose what you will.

![](https://res.cloudinary.com/code-mill-technologies-inc/image/upload/bo_2px_solid_rgb:000000,c_scale,h_600/v1512589498/Screen_Shot_2017-12-06_at_1.44.06_PM_u63lns.png)

The __Application claims__ determines which of the __Sign-up attributes__ values will be returned to the mobile app _after_ the user signs-in.

![](https://res.cloudinary.com/code-mill-technologies-inc/image/upload/bo_2px_solid_rgb:000000,c_scale,h_600/v1512743617/Screen_Shot_2017-12-08_at_8.33.06_AM_uvmomn.png)

_Make sure you select the __User's Object ID__, that will be needed by the MSAL library._

Once you have those two done - leave the rest as is... go ahead and click __Create__.

## Step 2 - Setting Up The Azure AD B2C Application

Go back to the overall Azure AD B2C blade ...  then select the __Applications__ menu. Then click __Add__ from the new blade that appears.

Here you're going to be able to give your new Azure AD B2C application a name - and to specify whether it should contain a Web API and Native client. You want to do both.

Then as you select __Yes__ to those options - a whole bunch of new options are going to appear ... so let's take a look at those one at a time.

![](https://res.cloudinary.com/code-mill-technologies-inc/image/upload/bo_2px_solid_rgb:000000/v1512490746/Screen_Shot_2017-12-05_at_10.18.44_AM_ciygoj.png)

### Web API Configuration

> Note, you only need to select __Yes__ to this part if you're doing something _in addition_ to the Web API which you're securing. If you only have a single Web API that has protected endpoints, and nothing else, you can leave this as __No__.

The __Reply Url__ under the __Web API__ section specifies a web address that you want a "cool / not cool" reply to go to _if a web app were requesting any authorization tokens_. Since we're not doing that - put any old URL in here.

Next up is the __App ID URI__ ... it says it's optional. Fill it out - we'll use it later. It doesn't need to resolve to anything - but make it unique. _(And don't worry if you're not doing this section - just think of something you would have put here, as I mentioned, it doesn't need to resolve.)

### Native Client Configuration

Leave the default values as is. We'll fill out the the __Custom Redirect URI__ a bit later.

Click the __Create__ button.

After the application has been created, its property blade will open. Within there you'll see an __Application ID__ box. Copy that value to the clipboard.

Then go down to the __Custom Redirect URL__ box under the __Native Client__ section and paste it in - in the following format:

`msal{APPLICATION-ID}://auth`

> Weird right? Yeah... but what this is specifying is a custom URL scheme that the web view which performs the sign-up and sign-in will use to communicate back to the app once the sign-up or sign-in is complete.

> So when we get around to configuring the platform projects, we'll need that value tell the OS that anytime something tries to invoke a URL with `msal{APPLICATION-ID}://auth` ... open our app.

### Scopes!

Now we want to add a scope to the Azure AD B2C application.

Hit the __Published scopes (Preview)__ menu option. Then in the new blade enter anything you want - but make it descriptive so you know what it means. Remember, the idea behind scopes is to give _permission_ to the backend resource that's being protected.

![](https://res.cloudinary.com/code-mill-technologies-inc/image/upload/bo_2px_solid_rgb:000000/v1512492882/Screen_Shot_2017-12-05_at_10.54.26_AM_oq2fok.png)

### API Access

![](https://res.cloudinary.com/code-mill-technologies-inc/image/upload/bo_2px_solid_rgb:000000/v1512745345/Screen_Shot_2017-12-08_at_9.01.52_AM_wcr9te.png)

Now to enable API Access. Hit the __API Access (Preview)__ menu option right above the scopes one, click __Add__, and then the available API should be your Azure AD B2C name.

You should also see the scope you just created in it as well.

We are done configuring the portal!! 🎉

## Step 3 - Changes to the Web API

Next we're going to make the Web API utilitize Azure AD B2C. In order to do that, we're going to have to do some refactoring to the Web API that was created [before](https://msou.co/61).

### AppSettings

We need to first update the appsettings.json - only to avoid hardcoding constants in the code itself.

The new portion of the file will look like:

```language-javascript
"Authentication": {
    "AzureAd": {
      "Tenant": "TheReviewer.onmicrosoft.com",
      "ClientId": "{YOUR_APPLICATION_ID_HERE}",
      "Policy": "B2C_1_GenericSignUpAndIn"
    }
  }
```

The __Tenant__ comes from the main blade of the Azure AD B2C tenant, found via the __Overview__ option on the left, and __Domain Name__ on the right.

![](https://res.cloudinary.com/code-mill-technologies-inc/image/upload/bo_2px_solid_rgb:000000/v1512745601/Screen_Shot_2017-12-08_at_9.06.22_AM_cydshi.png)

The __Client ID__ comes from the application's page we just created. And the policy is, of course, the policy we created above.

### Startup.cs

Ok ... now we want to make sure that we configure the WebAPI startup to handle authentication using JWT bearer tokens.

The first part in this is to set the `Configure` function to look like the following:

```language-csharp
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

Notice the use of `app.UseAuthentication()`.

Next up ... we're going do a lot of changes to the `ConfigureServices` function. It's going to end up looking like the following:

```language-csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddAuthentication(options => 
    {
        options.DefaultScheme = JwtBearerDefaults.AuthenticationScheme;
    })
    .AddJwtBearer(options => {
        options.RequireHttpsMetadata = false;
        options.Audience = Configuration["Authentication:AzureAd:ClientId"];
        options.Events = new JwtBearerEvents 
        {
            OnAuthenticationFailed = AuthenticationFailed
        };
        var authorityBase = string.Format("https://login.microsoftonline.com/tfp/{0}/",Configuration["Authentication:AzureAd:Tenant"]);
    
        options.Authority = string.Format("{0}{1}/v2.0/",authorityBase,Configuration["Authentication:AzureAd:Policy"]);

    });
    services.AddMvc();
}
```

The big thing here is that we're taking that `IServiceCollection` which gets passed in and then calling `AddAuthentication` on it - telling it we want to use `JwtBearer` tokens.

Then we go ahead and configure those JWT tokens. Notice the `Audience` is the `ClientId` of the Azure AD B2C application from the app settings file. And there's an `Authority` property that is set to a long URL, that even includes the policy name.

### Updating the Controller

Next we need to update the controller to make sure our app sends along an authorization token whenever it tries to call the `GetAllReviews` endpoint.

First thing that needs to be done is import the `Microsoft.AspNetCore.Authorization` namespace.

Next - use the `[Authorize]` attribute right above the function definition.

This way the definition will look something like the following:

```language-csharp
[HttpGet]
[Authorize]
public IEnumerable<Review> GetAllReviews()
{
```

Now we're all setup. If we had the mobile app try to invoke the Web API right now - it would receive a 401 - Unauthorized status code.

Of course - you're going to want to deploy the app to Azure now.

## Step 4 - Updating the Xamarin.Forms App

We're in the homestretch now!!

We're going to use the [Microsoft.Identity.Client](https://msou.co/7k) NuGet package (or MSAL) to take care of communicating to Azure AD B2C (and caching the tokens in respsonse) for us. This removes a lot of work on our end. (The package is in preview - [but the team is supporting it](https://msou.co/7t), and says you can use it in production environments).

Also, check out this [post](https://msou.co/7s) for more info on the MSAL.

Add that package to the platform projects and to the core Forms project.

### Core Project Basic Setup

For this initial stage, I'm going to brute force my way into getting everything working. That means I'm going to have some hardcoded static strings and static objects. 

Eventually I'll refactor all of this into a proper Login Service - but for now I only want to get the thing working.

So - for some basic setup - in the `App.cs` file - I'm going to add the following strings:

```language-csharp
public static string Tenant = "TheReviewer.onmicrosoft.com";
public static string ClientID = "{YOUR_APPLICATION_ID_HERE}";
public static string SignUpAndInPolicy = "B2C_1_GenericSignUpAndIn";

public static string AuthorityBase = $"https://login.microsoftonline.com/tfp/{Tenant}/";
public static string Authority = $"{AuthorityBase}{SignUpAndInPolicy}";

public static string[] Scopes = new string[] { "https://TheReviewer.onmicrosoft.com/backend/rvw.read.only" };
```

These function as configuration settings. The `Tenant`, `ClientID`, and `SignUpAndInPolicy` values should be evident where they came from.

The `Authority` is the endpoint we call to perform the authorization - it's created by using the 3 values that are defined before it.

Finally - the `Scopes[]` array. This is going to be the __App ID URI__ from the __Web API__ configuration in the portal (it said it was optional - but I said we would eventually need it - and now's the time)! And it's followed by the scope name that we want to invoke.

Then there are 2 object's that we want to create public static variables for:

```language-csharp
public static PublicClientApplication AuthClient = null;

public static UIParent UiParent = null;

public App()
{
    InitializeComponent();

    AuthClient = new PublicClientApplication(ClientID, Authority);
    AuthClient.ValidateAuthority = false;
    AuthClient.RedirectUri = $"msal{ClientID}://auth";

    MainPage = new NavigationPage(new ReviewListPage());
}
```

The `PublicClientApplication` object is what facilitates the communication to Azure AD B2C. And in the constructor, you can see how we're using the configuration variables to intialize various properties. (And look at the `RedirectUri` - looks familiar, eh? It's the __Custom Redirect URI__ from the __Native client__ section in the portal.)

While the `UIParent` is a class which is used by Android ... which I'll get to in a second...

AD B2C shows the login portion of its workflow within a web view. That means when the web view is dismissed, it will need to communicate back to the platform project. And that means ... there's some platform specific work we need to do.

### iOS Specific Steps

Over in iOS land - we're going to have to edit the Info.plist file to add a URL type to define a callback URL that gets invoked when the web view is dismissed. 

_(This is the callback stuff I was talking about above when configuring the _Native Application's_ _Custom Redirect URI_ in the portal.)_

The added section looks like this:

```language-xml
<key>CFBundleURLTypes</key>
<array>
    <dict>
        <key>CFBundleTypeRole</key>
        <string>Editor</string>
        <key>CFBundleURLName</key>
        <string>com.codemilltech.TheReviewer</string>
        <key>CFBundleURLSchemes</key>
        <array>
            <string>msal{YOUR_APPLICATION_ID}</string>
        </array>
    </dict>
</array>
```

The __CFBundleURLName__ is your app's bundle name. The __CFBundleUrlSchemes__ is obtained from the Azure AD B2C application's blade under the native client section. It's the __Custom Redirect URI__ minus the `://auth` at the end.

Then we need to override the `OpenUrl` function in the `AppDelegate`. It's pretty straight forward and will look like this:

```language-csharp
public override bool OpenUrl(UIApplication app, NSUrl url, NSDictionary options)
{
    AuthenticationContinuationHelper.SetAuthenticationContinuationEventArgs(url);

    return true;
}
```

The `AuthenticationContinuationHelper` is from the MSAL library, and it's there to help us coordinate the authentication flow.

### Android Specific Steps

> Please note: MSAL 1.1.0-preview will throw an exception with any version of Android using the version 25.x and above support libraries. Hopefully by the time you read this there is a newer version out there, and everything works great. (Here's the [GitHub issue](https://msou.co/7u) tracking it.)

In the Android app's `MainActivity`, we need to set that `UIParent` property. That's going to be done in the `OnCreate` function and will look like this:

```language-csharp
App.UiParent = new UIParent(Xamarin.Forms.Forms.Context as Activity);
```

This `App.UiParent` allows the MSAL to show the web view using the current Android activity.

Then we need to modify the `AndroidManifest.xml` file.

Add this into the `<application>` element:

```language-xml
<activity android:name="microsoft.identity.client.BrowserTabActivity">
    <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="msal{YOUR_APPLICATION_ID}" android:host="auth" />
    </intent-filter>
</activity>
```

That new `<activity>` element is defining a browser "window" that can open ... and it's going to be used for the web view that lets users sign up or sign in to our app.

### Sending Requests to Azure AD B2C

There are 2 functions that are members of the `PublicClientApplication` class from the MSAL library which we're going to use to get the correct tokens in order to make the Web API call.

The first is: `AcquireTokenSilentAsync` and the other is: `AcquireTokenAsync`.

Obviously they are both getting tokens somehow - but the silent version will first check to see if there already is a token stored on the device first before invoking showing the web view and having the user sign in.

Both of those functions will return an `AuthenticationResult` object. And it's that object which will hold the Access token we need.

Check out the Xamarin.Forms [sample app](https://msou.co/7l) to see all of the backing code.

### Sending the Token to the Web API

Finally - we have to send the access token that was retrieved from Azure AD B2C to the Web API - so we can invoke the function we're after.

That involves going into the Xamarin.Forms core project and refactoring how we performed the HTTP call from a [previous post](https://msou.co/61) into the following:

```language-csharp
var baseAddr = new Uri(location);
var client = new HttpClient { BaseAddress = baseAddr };

var reviewUri = new Uri(baseAddr, "api/reviews");
var request = new HttpRequestMessage(HttpMethod.Get, reviewUri);
request.Headers.Authorization = new AuthenticationHeaderValue("Bearer", authResult.AccessToken);

var response = await client.SendAsync(request);
response.EnsureSuccessStatusCode();

var reviewJson = await response.Content.ReadAsStringAsync();
```

Let's walk through this...

The `location` variable is the base URL of the Web API we want to get at - so we make a `Uri` out of it.

Then the `reviewUri` is the full URL to the exact endpoint we want to invoke - in this case the `reviews` controller.

Then the new stuff:

Instead of just using the `HttpClient` to invoke the Web API - I'm creating a new `HttpRequestMessage` object - and I'm saying it will perform a GET operation - and what URL to hit.

Then I'm setting the `Headers.Authorization` property - or the authorization headers. And it's going to be a `"Bearer"` with the value of the token obtained via the one of the `AquireToken` MSAL calls.

Finally - I send the request through the `client` and then get a `response` from that. Read the `response.Content` and that's all there is to it!!

## Running the App

I've added a Login button to the [sample app](https://msou.co/7l), that allows a user to log in before attempting to retrieve data.

Here's 3 screenshots from the app running on iOS. 

The first is showing the Login button. 

![](https://res.cloudinary.com/code-mill-technologies-inc/image/upload/bo_2px_solid_rgb:000000,c_scale,h_650/v1512687821/Simulator_Screen_Shot_-_iPhone_6s_-_2017-12-07_at_17.01.29_uz4iui.png)

The second is the screen which appears after the login button is pressed - this gives the user the ability to sign-in if they already have an account - or to create a new account.

![](https://res.cloudinary.com/code-mill-technologies-inc/image/upload/bo_2px_solid_rgb:000000,c_scale,h_650/v1512687821/Simulator_Screen_Shot_-_iPhone_6s_-_2017-12-07_at_17.01.45_p7quev.png)

And finally, the third screen allows the user to create a new account. Notice the __Display Name__ and __Job Title__ fields, those are from the __User Attributes__ portion of the __Sign Up and Sign In__ policy.

![](https://res.cloudinary.com/code-mill-technologies-inc/image/upload/bo_2px_solid_rgb:000000,c_scale,h_650/v1512687821/Simulator_Screen_Shot_-_iPhone_6s_-_2017-12-07_at_17.01.52_evxkd5.png)

### Some Quick Notes

Both the Web API and the Xamarin.Forms app, as they stand right now, are for demo purposes only. What I mean by that is they work ... but you wouldn't use them in a real world situation - not yet at least.

One of the reasons why is login information is not persisted across app launches, you always need to login.

Another is that the web pages which present the sign-in and sign-up info are basic - we'll need to customize those.

Plus, we'll eventually want to restrict access to certain parts of the app and Web API based on who is logged in.

If those some like something that will be covered in future posts ... you're right on with that!

Also - at least for now, the MSAL does not work on Android, but that will be fixed soon.

And check out the comments in the code, the iOS simulator since iOS 10 is a bit funky with the way it handles persisting things - although on device it works fine.

## Conclusion

This was a long post - and luckily most everything done in Step 1 and Step 2, will only have to be done once. Number 3 and 4 are done when you're developing the app - so those will become second nature after a while.

The quick rundown again is:

1. Setup Azure AD B2C in the portal - creating the policies and defining the user attributes to collect & return.
1. Setup the Azure AD B2C application in the portal - defining various callback URLs and scopes.
1. Get that Web API to use authorization via Azure AD B2C.
1. Enable the mobile app to do the same - including with the Microsoft Client Identity Library - or MSAL.

## What Comes Next?

This post serves as something of a bridge ... it lays a lot of necessary groundwork so we can get to the cool stuff.

So what comes next? 

Well - we'll definitely want to fix the issues I note in the _Some Quick Notes_ section.

I also plan on adding in the ability to authenticate via social providers like Twitter or Facebook. 2-Factor authentication is a very important security measure, and I'll implement that as well.

I will also cover how to integrate Azure AD B2C into various Azure App Services, such as Functions and Mobile App Service.