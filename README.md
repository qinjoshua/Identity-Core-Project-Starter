# ASP.NET Identity Core Project Starter
## With Custom Fields and Email Authentication

### Introduction
ASP.NET Identity Core is the built-in and solution for individual user accounts and authentication in ASP.NET MVC Core, Microsoft’s cross-platform implementation of the ASP.NET MVC web framework. We will add custom user fields to a new project using Identity Core, customize the user interface to support these additional fields, and add email confirmation to create a comprehensive Identity authentication system that can be integrated with any ASP.NET MVC Core project. We will be using Visual Studio 2017 for this guide.

By default, Identity Core will have the following fields:

`Id` – A GUID string that acts as the primary key for the user account

`UserName` – Unique string that identifies the user for login

`NormalizedUserName` – Username with letters normalized to uppercase characters

`Email` – The user’s email address

`NormalizedEmail` – Email with letters normalized to uppercase characters

`EmailConfirmed` – A boolean determining whether or not the user’s email address has been confirmed

`PasswordHash` – The hashed version of the password used for comparison when the user is authenticated

`SecurityStamp` – A GUID string that represents the current snapshot of the user credentials; used for disabling old authentication cookies, allowing the user to sign out of their account everywhere simultaneously

`ConcurrencyStamp` – Used to prevent concurrent update conflicts, by identifying if the database has changed since the page was loaded

`PhoneNumber` – The user’s phone number

`PhoneNumberConfirmed` – A boolean determining whether or not the user’s phone number has been confirmed

`TwoFactorEnabled` – A boolean representing whether or not two factor authentication has been enabled

`LockoutEnd` – When the lockout period on a user account will end, after too many incorrect password `attempts` – null if the user is not locked out

`LockoutEnabled` – A boolean representing whether or not a user will be locked out after too many incorrect password attempts

`AccessFailedCount` – The number of incorrect login attempts the user has committed so far

For this guide, we’ll be adding these four custom fields to our application user:

`FirstName` – The user’s first name

`LastName` – The user’s last name

`Birthday` – A date representing the user’s birthday

`Country` – The country that the user resides in

### Create Project with Identity Core
Create a new ASP.NET Core application and select MVC as the desired application template. In the same window, click on the “Change Authentication” button.

![Figure 1](https://github.com/qinjoshua/IdentityCoreProjectStarter/blob/main/Screenshots/Figure1.png)

Select Individual User Accounts in the “Change Authentication” window, which will setup Identity Core automatically when the project template is created.

![Figure 1](https://github.com/qinjoshua/IdentityCoreProjectStarter/blob/main/Screenshots/Figure2.png)

### Add Custom Application User
Under the "Models" folder, create a new class called `ApplicationUser` and inherit the `Identity` class. The `ApplicationUser` class will represent the customized user for this application. Add the custom fields that you would like to include in registration to this `ApplicationUser` class.

```c#
using System.ComponentModel.DataAnnotations.Schema;
using Microsoft.AspNetCore.Identity;

namespace IdentityCore.Models
{
    public class ApplicationUser : IdentityUser
    {
        [PersonalData]
        public string FirstName { get; set; }

        [PersonalData]
        public string LastName { get; set; }

        [PersonalData]
        [Column(TypeName = "Date")]
        public DateTime Birthday { get; set; }

        [PersonalData]
        public string Country { get; set; }
    }
}
```

In "ApplicationDbContext.cs" under the Data folder, modify the existing code to inherit from IdentityDbContext with your newly created ApplicationUser class.

```c#
using IdentityCore.Models;

namespace IdentityCore.Data
{
    public class ApplicationDbContext : IdentityDbContext<ApplicationUser>
    {
        public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options)
            : base(options)
        {
        }
    }
}
```

To configure your `ApplicationUser` for use in the project, add this code to your "startup.cs" inside the `ConfigureServices` method.

```c#
public void ConfigureServices(IServiceCollection services)
{
    // Other code - shortened for brevity

    services.AddIdentity<ApplicationUser, IdentityRole>(options =>
    {
        options.Password.RequiredLength = 7;
        options.Password.RequireDigit = true;
        options.Password.RequireUppercase = false;
        options.User.RequireUniqueEmail = true;
        options.SignIn.RequireConfirmedEmail = true;
    }).AddDefaultUI().AddEntityFrameworkStores<ApplicationDbContext>().AddDefaultTokenProviders();

    // Replace this section with the code above
    services.AddDefaultIdentity<IdentityUser>()
            .AddDefaultUI(UIFramework.Bootstrap4)
            .AddEntityFrameworkStores<ApplicationDbContext>();
}
```

Now we can add our model migrations so that our ApplicationUser's custom fields are reflected in the database. To do so, go to the Package Manager Console and type `Add-Migration ApplicationUser` followed by `Update-Database`

![Figure 1](https://github.com/qinjoshua/IdentityCoreProjectStarter/blob/main/Screenshots/Figure3.png)

Finally, in "_LoginPartial.cshtml", replace `IdentityUser` with your new `ApplicationUser`.

```c#
@using Microsoft.AspNetCore.Identity
@inject SignInManager<ApplicationUser> SignInManager
@inject UserManager<ApplicationUser> UserManager
```

### Add EmailSender Service
For this project, we'll be using MailKit by Jeffry Steadfast to send the authentication email.

In the toolbar, click Project → Manage NuGet Packages..., and then click on "Browse." Search for the MailKit package, and then click "Install." Click "Ok" for any install prompts that come up. This should add the MailKit package into your project

![Figure 1](https://github.com/qinjoshua/IdentityCoreProjectStarter/blob/main/Screenshots/Figure4.png)

Create a new folder "Services" in your project. Add an interface called `EmailSettings` in your newly created "Services" folder, and add the following code:

```c#
namespace IdentityCore.Services
{
    public class EmailSettings
    {
        public string MailServer { get; set; }
        public int MailPort { get; set; }
        public string SenderName { get; set; }
        public string Sender { get; set; }
        public string Password { get; set; }
    }
}
```

Add a class `EmailSender` under "Services", and let it inherit `IEmailSender` from the `Microsoft.AspNetCore.Identity.UI.Services` namespace. In `EmailSender`, write the following code:

```c#
using MailKit.Net.Smtp;
using Microsoft.AspNetCore.Hosting;
using Microsoft.Extensions.Options;
using MimeKit;
using Microsoft.AspNetCore.Identity.UI.Services;

namespace IdentityCore.Services
{
    public class EmailSender : IEmailSender
    {
        private readonly EmailSettings _emailSettings;
        private readonly IHostingEnvironment _env;

        public EmailSender(
            IOptions<EmailSettings> emailSettings,
            IHostingEnvironment env)
        {
            _emailSettings = emailSettings.Value;
            _env = env;
        }

        public async Task SendEmailAsync(string email, string subject, string message)
        {
            try
            {
                var mimeMessage = new MimeMessage();
                mimeMessage.From.Add(new MailboxAddress(_emailSettings.SenderName, _emailSettings.Sender));
                mimeMessage.To.Add(new MailboxAddress("User", email));
                mimeMessage.Subject = subject;
                mimeMessage.Body = new TextPart("html")
                {
                    Text = message
                };

                using (var client = new SmtpClient())
                {
                    // For demo-purposes, accept all SSL certificates (in case the server supports STARTTLS)
                    client.ServerCertificateValidationCallback = (s, c, h, e) => true;

                    if (_env.IsDevelopment())
                    {
                        // The third parameter is useSSL (true if the client should make an SSL-wrapped
                        // connection to the server; otherwise, false).
                        await client.ConnectAsync(_emailSettings.MailServer, _emailSettings.MailPort, true);
                    }
                    else
                    {
                        await client.ConnectAsync(_emailSettings.MailServer);
                    }

                    // Note: only needed if the SMTP server requires authentication
                    await client.AuthenticateAsync(_emailSettings.Sender, _emailSettings.Password);
                    await client.SendAsync(mimeMessage);
                    await client.DisconnectAsync(true);
                }

            }
            catch (Exception ex)
            {
                throw new InvalidOperationException(ex.Message);
            }
        }
    }
}
```
For the sake of this sample, we will store the login information for the email server in "appsettings.json". This is not recommended best practice for deployment, since application secrets should never be written in source code. For more information about how to store passwords and other sensitive information, check out this article from Microsoft:
https://docs.microsoft.com/en-us/dotnet/architecture/microservices/secure-net-microservices-web-applications/developer-app-secrets-storage#:~:text=To%20connect%20with%20protected%20resources%20and%20other%20services,,to%20read%20the%20secrets%20from%20more%20secure%20locations.

```json
// Shortened for brevity
"EmailSettings": {
    "MailServer": "mail.example.com",
    "MailPort": 465,
    "SenderName": "Identity Core Sample Account Registration",
    "Sender": "example@example.com",
    "Password": "password123"
},
```

Now it's time to configure the email sender on program startup. Modify `ConfigureServices` in "startup.cs" to include the following code:

```c#
public void ConfigureServices(IServiceCollection services)
{
    // Other code - shortened for brevity

    services.Configure<EmailSettings>(Configuration.GetSection("EmailSettings"));
    services.AddSingleton<IEmailSender, EmailSender>();
}
```

### Scaffold Identity Core
We would like to be able to modify our identity-related pages in order to customzie the user interface and add in custom fields on the frontend to match the custom fields we added in our model. Right click project and click Add → New Scaffolded Item…

![Figure 1](https://github.com/qinjoshua/IdentityCoreProjectStarter/blob/main/Screenshots/Figure5.png)

On the left-hand pane, click Identity. Select Identity from the list of scaffolding options. Check the following options to add to your project:

```
Account\Login
Account\ConfirmEmail
Account\Register
```

Select `ApplicationDbContext` from the dropdown next to “Data context class”, and then press the “Add” button to add the Identity files to your project

![Figure 1](https://github.com/qinjoshua/IdentityCoreProjectStarter/blob/main/Screenshots/Figure6.png)

The files should be added to the project under the “Areas” folder.

![Figure 1](https://github.com/qinjoshua/IdentityCoreProjectStarter/blob/main/Screenshots/Figure7.png)

If you would like to modify the look and feel of any of the registration pages, it can be done from within this folder.

### Add Custom Fields to Registration
Open the file "Register.cshtml.cs" at Areas → Identity → Pages → Account → Register.cshtml.cs → Register.cshtml.cs. Locate the `InputModel` class definition, and modify the code to include your custom fields:

```c#
public class InputModel
{
    [Required]
    [DataType(DataType.Text)]
    [Display(Name = "First name")]
    public string FirstName { get; set; }

    [Required]
    [DataType(DataType.Text)]
    [Display(Name = "Last name")]
    public string LastName { get; set; }

    [Required]
    [DataType(DataType.Date)]
    [Display(Name = "Birthday")]
    public DateTime Birthday { get; set; }

    [Required]
    [DataType(DataType.Text)]
    [Display(Name = "Country")]
    public string Country { get; set; }

    [Required]
    [EmailAddress]
    [Display(Name = "Email")]
    public string Email { get; set; }

    [Required]
    [StringLength(100, ErrorMessage = "The {0} must be at least {2} and at max {1} characters long.", MinimumLength = 6)]
    [DataType(DataType.Password)]
    [Display(Name = "Password")]
    public string Password { get; set; }

    [DataType(DataType.Password)]
    [Display(Name = "Confirm password")]
    [Compare("Password", ErrorMessage = "The password and confirmation password do not match.")]
    public string ConfirmPassword { get; set; }
}
```

### Update UI
Finally, we can update the user interface to accept our additional fields. You can modify the UI however best fits your project, including importing your own custom CSS templates. For this demo, I modified the code as follows:

```html
<div class="col-md-6 offset-md-3">
    <form asp-route-returnUrl="@Model.ReturnUrl" method="post">
        <h4>Create a new account</h4>
        <hr />
        <div asp-validation-summary="All" class="text-danger"></div>
        <div class="row">
            <div class="col-sm-12">
                <div class="form-group">
                    <label asp-for="Input.Email"></label>
                    <input asp-for="Input.Email" class="form-control" />
                    <span asp-validation-for="Input.Email" class="text-danger"></span>
                </div>
            </div>
        </div>
        <div class="row">
            <div class="col-sm-6">
                <div class="form-group">
                    <label asp-for="Input.FirstName"></label>
                    <input asp-for="Input.FirstName" class="form-control" />
                    <span asp-validation-for="Input.FirstName" class="text-danger"></span>
                </div>
            </div>
            <div class="col-sm-6">
                <div class="form-group">
                    <label asp-for="Input.LastName"></label>
                    <input asp-for="Input.LastName" class="form-control" />
                    <span asp-validation-for="Input.LastName" class="text-danger"></span>
                </div>
            </div>
        </div>
        <div class="row">
            <div class="col-12">
                <div class="form-group">
                    <label asp-for="Input.Birthday"></label>
                    <input asp-for="Input.Birthday" type="text" class="form-control" data-provide="datepicker" placeholder="Enter the date" />
                    <span asp-validation-for="Input.Birthday" class="text-danger"></span>
                </div>
            </div>
        </div>
        <div class="row">
            <div class="col-sm-12">
                <div class="form-group">
                    <label class="form-label" asp-for="Input.Country"></label>
                    <div class="form-control-wrap" data-select2-id="35">
                        <div class="form-control-wrap">
                            <select class="form-select form-control" data-search="on" asp-for="Input.Country">
                                <option value="United States of America">United States of America</option>
                                <option value="Afganistan">Afghanistan</option>
                                <!-- ... The rest of the countries are included in these options -->
                            </select>
                        </div>
                    </div>
                    <span asp-validation-for="Input.Country" class="text-danger"></span>
                </div>
            </div>
        </div>
        <div class="row">
            <div class="col-12">
                <div class="form-group">
                    <label asp-for="Input.Password"></label>
                    <input asp-for="Input.Password" class="form-control" />
                    <span asp-validation-for="Input.Password" class="text-danger"></span>
                </div>
            </div>
        </div>
        <div class="row">
            <div class="col-12">
                <div class="form-group">
                    <label asp-for="Input.ConfirmPassword"></label>
                    <input asp-for="Input.ConfirmPassword" class="form-control" />
                    <span asp-validation-for="Input.ConfirmPassword" class="text-danger"></span>
                </div>
            </div>
        </div>
        <button type="submit" class="btn btn-primary">Register</button>
    </form>
</div>

@section Scripts {
    <partial name="_ValidationScriptsPartial" />
    <script src="https://cdnjs.cloudflare.com/ajax/libs/bootstrap-datepicker/1.9.0/js/bootstrap-datepicker.js"></script>
    <script>
        $(document).ready(function () {
            $("#example").datepicker();
        });
    </script>
}
```

Now all that's left to do is to test that the account registration works! You can check that the ApplicationUser migrations were successfully applied by going to "Server Explorer", double-clicking the `aspnet_users` table, and checking to see if your custom fields were applied to the database. Create a sample account to test whether or not the information saves to the database properly, as well as if the confirmation email sends.