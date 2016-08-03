# Add SMS and Email 2FA to your ASP .NET Identity App using Nexmo and SendGrid

2FA is a must nowadays to increase the security within your application. It is seen in all kinds of apps: from the signup process to user action verification. The most common types of 2FA are email verification and phone verification. In this tutorial we'll show how to set up 2FA in your .NET application using ASP .NET Identity, the [SendGrid C# Client](#) for email auth and the [Nexmo C# Client Library](https://github.com/nexmo/nexmo-dotnet) for SMS and Call-based auth.

### [![alt text](https://cloud.githubusercontent.com/assets/328367/17298941/0cd29600-5804-11e6-950c-4542416776bf.png)](https://github.com/sidsharma27/dot-net-2FA-demo/commit/ee500bbadfd803b9d82986a492db367f5b262ced) Setup ASP .NET MVC application

Open Visual Studio and create a new ASP .NET MVC application. For this demo, we'll delete the Contact & About sections from the default generated website. 

### [![alt text](https://cloud.githubusercontent.com/assets/328367/17298941/0cd29600-5804-11e6-950c-4542416776bf.png)](https://github.com/nexmo-community/nexmo-verify-2fa-dotnet-example/commit/ee500bbadfd803b9d82986a492db367f5b262ced) Install the Nexmo Client to your app via NuGet Package Manager 

Add the Nexmo Client to your application via the NuGet Package Console. 

```
PM> Install-Package Nexmo.Csharp.Client ​*-Version 6.3.3*
```

### [![alt text](https://cloud.githubusercontent.com/assets/328367/17298941/0cd29600-5804-11e6-950c-4542416776bf.png)](https://github.com/nexmo-community/nexmo-verify-2fa-dotnet-example/commit/ee500bbadfd803b9d82986a492db367f5b262ced) Install the SendGrid client via NuGet Package Manager 

When doing this be sure to use the 6.3.3 version rather than the current 7.0.2 as there has been some difficulty with the namespaces for .NET 4.5.2

**TODO: Can you install a specific version from the command line?**

```
PM> Install-Package SendGrid via NuGet Package Manager 
```

### [![alt text](https://cloud.githubusercontent.com/assets/328367/17298941/0cd29600-5804-11e6-950c-4542416776bf.png)](https://github.com/nexmo-community/nexmo-verify-2fa-dotnet-example/commit/ee500bbadfd803b9d82986a492db367f5b262ced) Add Nexmo and SendGrid credentials

For the purpose of the demo we'll put the Nexmo and SendGrid credentials in the '<appSettings>' section of the'Web.config' file. If we were developing this application for distribution we may chose to enter these credentials in our Azure portal.

```xml
<add key="Nexmo.Url.Rest" value="https://rest.nexmo.com"/>
<add key="Nexmo.Url.Api" value="https://api.nexmo.com"/>
<add key="Nexmo.api_key" value="NEXMO_API_KEY"/>
<add key="Nexmo.api_secret" value="NEXMO_API_SECRET"/>
<add key="SMSAccountFrom" value="SMS_FROM_NUMBER"/>
<add key="mailAccount" value="SENDGRID_USERNAME"/>
<add key="mailPassword" value="SENDGRID_PASSWORD"/>
```

### [![alt text](https://cloud.githubusercontent.com/assets/328367/17298941/0cd29600-5804-11e6-950c-4542416776bf.png)](https://github.com/nexmo-community/nexmo-verify-2fa-dotnet-example/commit/6d6242e587a0d5312fa90e3c13f749fc487945b2) Plug in Nexmo in the SMS Service, SendGrid in the Email Service

Inside the `IdentityConfig.cs` file, add the SendGrid configuration in the `SMSService` method. Then, plug in the Nexmo Client inside the `SMSService` method of the `IdentityConfig.cs` file. Add the using directives for the Nexmo and SendGrid namespaces.

```cs
public class EmailService : IIdentityMessageService
{
    public async Task SendAsync(IdentityMessage message)
    {
        // Plug in your email service here to send an email.
        await configSendGridasync(message);
    }
    private async Task configSendGridasync(IdentityMessage message)
    {
        var myMessage = new SendGridMessage();
        myMessage.AddTo(message.Destination);
        myMessage.From = new System.Net.Mail.MailAddress(
                        "demo@nexmo.com", "Nexmo2FADemo");
        myMessage.Subject = message.Subject;
        myMessage.Text = message.Body;
        myMessage.Html = message.Body;
        var credentials = new NetworkCredential(
            ConfigurationManager.AppSettings["mailAccount"],
            ConfigurationManager.AppSettings["mailPassword"]
            );

        // Create a Web transport for sending email.
        var transportWeb = new Web(credentials);

        // Send the email.
        if (transportWeb != null)
        {
            await transportWeb.DeliverAsync(myMessage);
        }
        else
        {
            Trace.TraceError("Failed to create Web transport.");
            await Task.FromResult(0);
        }
    }
}

public class SmsService : IIdentityMessageService
{
    public Task SendAsync(IdentityMessage message)
    {
        var sms = SMS.Send(new SMS.SMSRequest
        {
            from = ConfigurationManager.AppSettings["SMSAccountFrom"],
            to = message.Destination,
            text = message.Body
        });
        return Task.FromResult(0);
    }
}

```

### [![alt text](https://cloud.githubusercontent.com/assets/328367/17298941/0cd29600-5804-11e6-950c-4542416776bf.png)](https://github.com/nexmo-community/nexmo-verify-2fa-dotnet-example/commit/c380379c50b3b7c107c07bde48e09913695c2324) Add 'SendEmailConfirmationTokenAsync()' method to 'AccountController'

Add the following method to your `AccountController` which will be called on user registration to send a confirmation email to the provided email address.

```cs
private async Task<string> SendEmailConfirmationTokenAsync(string userID, string subject)
{
    string code = await UserManager.GenerateEmailConfirmationTokenAsync(userID);
    var callbackUrl = Url.Action("ConfirmEmail", "Account",  new { userId = userID, code = code }, protocol: Request.Url.Scheme);
    await UserManager.SendEmailAsync(userID, subject, "Please confirm your account by clicking <a href=\"" + callbackUrl + "\">here</a>");
    return callbackUrl;
}
```

### [![alt text](https://cloud.githubusercontent.com/assets/328367/17298941/0cd29600-5804-11e6-950c-4542416776bf.png)](https://github.com/nexmo-community/nexmo-verify-2fa-dotnet-example/commit/f7324744efe7c03f642325fe016600e73cc4a3c8) Update 'Register' action method

Inside the `Register` method of the `AccountController`, add a couple properties to newly created variable of the ApplicationUser type: `TwoFactorEnabled` (`true`), `PhoneNumberConfirmed` (`false`). Once the user is successfully created, store the user ID in a session state and redirect the user to the `AddPhoneNumber` action method in the `ManageController`.

```cs
[HttpPost]
[AllowAnonymous]
[ValidateAntiForgeryToken]
public async Task<ActionResult> Register(RegisterViewModel model)
{
    if (ModelState.IsValid)
    {
        var user = new ApplicationUser { UserName = model.Email, Email = model.Email, TwoFactorEnabled = true, PhoneNumberConfirmed = false};
        var result = await UserManager.CreateAsync(user, model.Password);
        if (result.Succeeded)
        {
            Session["UserID"] = user.Id;
            return RedirectToAction("AddPhoneNumber", "Manage");
        }
        AddErrors(result);
    }
    // If we got this far, something failed, redisplay form
    return View(model);
}
```

### [![alt text](https://cloud.githubusercontent.com/assets/328367/17298941/0cd29600-5804-11e6-950c-4542416776bf.png)](https://github.com/nexmo-community/nexmo-verify-2fa-dotnet-example/commit/7830bd4bf6da255448f2566b1186fbd0a9122cb7) Check DB if phone number is taken and add SMS logic to the AddPhoneNumber action method

Add the `[AllowAnonymous]` attribute to both the GET & POST `AddPhoneNumber` action methods. This allows the user in the process of registering to access the phone number confirmation workflow. Make a query and check the DB if the phone number entered by the user is previously associated with an account. If not, redirect the user to the `VerifyPhoneNumber` action method.

```cs
[HttpPost]
[AllowAnonymous]
[ValidateAntiForgeryToken]
public async Task<ActionResult> AddPhoneNumber(AddPhoneNumberViewModel model)
{
    if (!ModelState.IsValid)
    {
        return View(model);
    }
    var db = new ApplicationDbContext();
    if (db.Users.FirstOrDefault(u => u.PhoneNumber == model.Number) == null)
    {
        // Generate the token and send it
        var code = await UserManager.GenerateChangePhoneNumberTokenAsync((string)Session["UserID"], model.Number);
        if (UserManager.SmsService != null)
        {
            var message = new IdentityMessage
            {
                Destination = model.Number,
                Body = "Your security code is: " + code
            };
            await UserManager.SmsService.SendAsync(message);
        }
        return RedirectToAction("VerifyPhoneNumber", new { PhoneNumber = model.Number });
    }
    else
    {
        ModelState.AddModelError("", "The provided phone number is associated with another account.");
        return View();
    }
}
```

### [![alt text](https://cloud.githubusercontent.com/assets/328367/17298941/0cd29600-5804-11e6-950c-4542416776bf.png)](https://github.com/nexmo-community/nexmo-verify-2fa-dotnet-example/commit/9c212c41fe55bcd9e38f14348af55ddb774d1419) Update VerifyPhoneNumber Action method

Add the `[AllowAnonymous]` attribute to the action method and delete everything in the method but the return statement that directs the verification flow,

```cs
[AllowAnonymous]
public async Task<ActionResult> VerifyPhoneNumber(string phoneNumber)
{
    return phoneNumber == null ? View("Error") : View(new VerifyPhoneNumberViewModel { PhoneNumber = phoneNumber });
}
```

Replace `User.Identity.GetUserId()` with `Session["UserID"]` in the method as shown below. If the user successfully enters the pin code, they are directed to the Index view of the Manage controller. The User's boolean property `PhoneNumberConfirmed` is then set to `true`.           
 
 ```cs
[AllowAnonymous]
[HttpPost]
[ValidateAntiForgeryToken]
public async Task<ActionResult> VerifyPhoneNumber(VerifyPhoneNumberViewModel model)
{
    if (!ModelState.IsValid)
    {
        return View(model);
    }
    var result = await UserManager.ChangePhoneNumberAsync((string)Session["UserID"], model.PhoneNumber, model.Code);
    if (result.Succeeded)
    {
        var user = await UserManager.FindByIdAsync((string)Session["UserID"]);
        if (user != null)
        {
            await SignInManager.SignInAsync(user, isPersistent: false, rememberBrowser: false);
        }
        return RedirectToAction("Index", new { Message = ManageMessageId.AddPhoneSuccess });
    }
    // If we got this far, something failed, redisplay form
    ModelState.AddModelError("", "Failed to verify phone");
    return View(model);
}
```
### [![alt text](https://cloud.githubusercontent.com/assets/328367/17298941/0cd29600-5804-11e6-950c-4542416776bf.png)](https://github.com/nexmo-community/nexmo-verify-2fa-dotnet-example/commit/078b28917fbfc9b48f09b01fbd20bd747270efc4) Check if the user has a confirmed email on Login

In the `Login()` action method, check to see if the user has confirmed their email or not. If not, return an error message (ex: "You must have a confirmed email to login.") and redirect the user to the "Info" view. Also, call the `SendEmailConfirmationTokenAsync()` method with the following parameters: user ID (Ex: user.Id) and the email subject (Ex: "Confirm your account.") 

```cs
var user = await UserManager.FindByNameAsync(model.Email);
if (user != null)
{
    if (!await UserManager.IsEmailConfirmedAsync(user.Id))
    {
        string callbackUrl = await SendEmailConfirmationTokenAsync(user.Id, "Confirm your account");
        ViewBag.title = "Check Email";
        ViewBag.message = "You must have a confirmed email to login.";
        return View("Info");
    }
}
```

### [![alt text](https://cloud.githubusercontent.com/assets/328367/17298941/0cd29600-5804-11e6-950c-4542416776bf.png)](https://github.com/nexmo-community/nexmo-verify-2fa-dotnet-example/commit/00dc4aa91f7ec73ee52db1f6ef23c136d83de6ae) Add Info View

Inside the Account folder of the Views folder, create a new View named 'Info' that the user will be redirected to if their email has not been confirmed. The view should contain the following code:

```xml
<h2>@ViewBag.Title.</h2>
<h3>@ViewBag.Message</h3>
```

### [![alt text](https://cloud.githubusercontent.com/assets/328367/17298941/0cd29600-5804-11e6-950c-4542416776bf.png)](https://github.com/nexmo-community/nexmo-verify-2fa-dotnet-example/commit/8501c9884faa97139dc4b30f2f41b4c448cd2641) Update Login View
Inside the Account folder of the Views folder, edit the Login and VerifyCode views. In both files, delete the <div> containing the 'Remember Me' checkbox. This will restrict the user from bypassing 2FA verification.  Also, delete the corresponding variable in each of the view models ('SendCodeViewModel' and 'VerifyCodeViewModel').

### Conclusion

With that, you have a web app using ASP .NET Identity that is 2 Factor Authentication enabled using Nexmo Verify and SendGrid Email as the different methods of verification.  

![alt text](https://giphy.com/gifs/d3g1n80UrhF12Rzi/fullscreen)

2FA adds a layer of security to correctly identify users and further protect sensitive user information. Using Nexmo's C# Client Library and SendGrid's C# Client, you can add both email and phone verification for your 2FA solution with ease. Feel free to send me any thoughts/questions on Twitter @sidsharma_27 or email me at sidharth.sharma@nexmo.com!