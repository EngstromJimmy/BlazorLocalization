# Blazor Localization sample

Localizing our content is sometimes needed to reach a broader audience.
Blazor has nice build-in ways to localize content and there are even nice ways in the Blazor community.
In this sample, we will localize our content using the NuGet package **Toolbelt.Blazor.I18nText** built by fellow MVP Junichi Sakamoto.
The built-in support for localization is using resx files which requires a specific editor to work.
Using this Nuget package we can use JSON files instead which I think is a nicer way to work with translations.

## Blazor server
### 1. Adding Localization
To add localization for Blazor Server we need to set up some things in **program.cs**

1. Add localization  
We make sure to configure the *RequestLocalizationOptions* this way we will be able to read these settings from anywhere, removing the need to have the supported languages in multiple places inside the application.
In Program.cs add the following code:
    ```Csharp
    builder.Services.AddLocalization();
    builder.Services.Configure<RequestLocalizationOptions>(options =>
    {
        var supportedCultures = new[] { "en-US", "sv-SE" };
        var localizationOptions = new RequestLocalizationOptions()
            .SetDefaultCulture(supportedCultures[0])
            .AddSupportedCultures(supportedCultures)
            .AddSupportedUICultures(supportedCultures);
    });
    ```
2. Configure RequestLocalization  
In this case, we are telling ASP.NET that we have two cultures US English and Swedish, mostly because those are two languages I know =).
We are also setting the default language to English.
In Program.cs add the following code:
     ```CSharp
    var localizationOptions = app.Services.GetService<IOptions<RequestLocalizationOptions>>();
    if (localizationOptions != null)
    {
        app.UseRequestLocalization(localizationOptions.Value);
    }
    ```

3. Add Minimal API  
In the official samples from Microsoft (at the time of writing this) they are using controllers. I think it looks a little bit better using Minimal APIs instead.
In Program.cs add the following code:
    ```Csharp
    app.MapGet("Culture/Set", (HttpRequest request, [FromQuery] string culture, [FromQuery] string redirectUri) =>
    {
        if (culture != null)
        {
            request.HttpContext.Response.Cookies.Append(
                CookieRequestCultureProvider.DefaultCookieName,
                CookieRequestCultureProvider.MakeCookieValue(
                 new RequestCulture(culture, culture)),
                  new CookieOptions
                  {
                      Path = "/",
                      Expires = DateTime.Now.AddYears(1)
                  });
        }

        return Results.LocalRedirect(redirectUri);
    });
    ```
Here we are adding a cookie and also setting the expiration time to one year, this means that when your user comes back to the site, the site will remember the setting (at least if they do that within a year).

### 2. Adding a component to change the language
It wouldn't be a great demo if we didn't have a way to change the language?
In this section, we will add a component to switch languages.
In program.cs we added ```IOptions<RequestLocalizationOptions>>``` which gives us access to the RequestLocalizationOptions from any component that injects the ```IOptions<RequestLocalizationOptions>>``` this means that we can get the supported languages from that setting and we don't have to duplicate the code.
1. Add a new component  
In the Pages folder add a new Razor component called **CultureSelector.razor**  
Replace the content with:
    ```csharp
    @using System.Globalization
    @using Microsoft.Extensions.Options
    @inject NavigationManager Nav
    @inject IOptions<RequestLocalizationOptions> RequestLocalizationOptions
    <p>
        <label>
            Select your locale:
            <select @bind="Culture">
                @foreach (var culture in supportedCultures)
                {
                    <option value="@culture">@culture.DisplayName</option>
                }
            </select>
        </label>
    </p>

    @code
    {
        private CultureInfo[] supportedCultures = new CultureInfo[0];

        protected override void OnInitialized()
        {
            if (RequestLocalizationOptions != null && RequestLocalizationOptions.Value.SupportedCultures!=null)
            {
                supportedCultures = RequestLocalizationOptions.Value.SupportedCultures.ToArray();
            }
            Culture = CultureInfo.CurrentCulture;
        }

        private CultureInfo Culture
        {
            get => CultureInfo.CurrentCulture;
            set
            {
                if (CultureInfo.CurrentCulture != value)
                {
                    var uri = new Uri(Nav.Uri)
                        .GetComponents(UriComponents.PathAndQuery, UriFormat.Unescaped);
                    var cultureEscaped = Uri.EscapeDataString(value.Name);
                    var uriEscaped = Uri.EscapeDataString(uri);

                    Nav.NavigateTo(
                        $"Culture/Set?culture={cultureEscaped}&redirectUri={uriEscaped}",
                        forceLoad: true);
                }
            }
        }
    }
    ```
This example is taken from Microsoft Docs with some minor changes. We have added ```@inject IOptions<RequestLocalizationOptions> RequestLocalizationOptions``` so we can get the supported languages from the setting we created earlier.

2. Add a new component to the layout
In the file Shared/MainLayout just befor the closing </main> add the the component
    ```html
      <div class="bottom-row px-4">
          <CultureSelector />
      </div>
    ```

### 3. Adding translations
We can now change language but we don't have any translations yet. .NET offers localization using .RESX which requires a specific editor. J. Sakamoto has created a library that uses JSON or CSV files instead which simplifies the process a bit.
The latest version (12) uses source generators to build the resources.
1. Add the **Toolbelt.Blazor.I18nText** Nuget package
2. Add folder called **i18ntext** 
3. In the i18ntext folder add a file called **MyText.en.json**
Replace the content with:
    ```JSON
    {
	    "Greeting": "Hello world!"
    }
    ```
4. In the i18ntext folder add a file called **MyText.sv.json**
Replace the content with:
    ```JSON
    {
	    "Greeting": "Hej värld!"
    }
    ```
If you take a look in the solution explorer under **Dependencies**, **Toolbelt.Blazor.I18nText.SourceGenerator** and expand the sub-nodes we will find a file called **BlazorServer.I18nText.MyText.g.cs**. This file contains the autogenerated class that we can call to get our translations in the next step.

### 4. Getting the translations
To be able to get the translations we need to hook up some things.
1. Add Toolbelt
In Program.cs add:
     ```csharp
     builder.Services.AddI18nText(options =>
    {
        options.PersistanceLevel = PersistanceLevel.None;
        options.GetInitialLanguageAsync = (_, _) => ValueTask.FromResult(CultureInfo.CurrentUICulture.Name);
    });
    ```
By default, I18nText uses a variable from local storage or session storage (if we want to persist the selected value.)
By overriding the *GetInitialLanguageAsync* method we tell it to use the CurrentUICulture. By doing so we connect the build-in localization to the Toolbelt.Blazor.I18nText implementation. 
2. Add any missing namespaces ```using Toolbelt.Blazor.Extensions.DependencyInjection;```
3. In **Pages/Index.razor** inject I18nText
    ```csharp
    @inject Toolbelt.Blazor.I18nText.I18nText I18nText
    ```
4. Add code block
Add the following code:
    ```csharp
    @code{
        I18nText.MyText MyText = new();
        protected async override Task OnInitializedAsync()
        {
            MyText = await I18nText.GetTextTableAsync<I18nText.MyText>(this);
            await base.OnInitializedAsync();
        }
    }
    ```
In this snippet, we get the TextTable i.e our translation.

5. Show translation  
The last step is to show the translation by adding the following code:
    ```html
    <h1>@MyText.Greeting</h1>
    ```

We now have a fully localizable solution for Blazor Server.

## Coming up
- [ ] Add WebAssembly Sample
