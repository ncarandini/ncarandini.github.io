---
layout: post
title: "Extend UserSecrets to Xamarin Forms"
subtitle: "And keep them away from your GitHub repository."
date: 2020-06-17 21:00:00 +0200
background: '/img/posts/P3/background.jpg'
---

Secrets should never be committed and pushed to a Git Versioning system like GitHub because is extremely difficult to erase them from the history of the project.

Sometimes we tend to neglet this simple fact and we commit sensible data eventually deleting it later, forgetting that that sensible data is still there, on one or more previous versions of the project. This is so common that GitHub scans repositories for known types of secrets, to prevent fraudulent use of secrets that were committed accidentally.

Moreover, sometimes we need to debug/test our project with configuration settings that are user specific and should not be committed and pushed to the remote repository.

On ASP.NET Core projects we can use UserSecrets to solve these scenarios, so I've decided to do something to use it also with Xamarin Forms.

Because the UserSecrets json data is embedded into the app, it can be read easily.
Moreover, because the UserSecrets file stay outside the git root folder, it's never pushed to the GitHub repository.

## Create a "UserSecrets" file
Visual Studio has an easy way to create UserSecrets by right clicking on the Xamarin Forms common project and select `Manage User Secrets` :

![folder creation](/img/posts/P3/001.png)

Elas, Visual studio for Mac don't have any command to manage user secrets, but we can use the .NET Core CLI (Command-Line Interface):

1) Open the Terminal  
2) Change directory to the Xamarin Forms common project folder  
3) Execute `dotnet user-secrets init`

![folder creation](/img/posts/P3/002.png)

Using the .NET CLI, we can also add one or more secrets, like with:  
`dotnet user-secrets set "MySecret" "HGttG:42"`

![folder creation](/img/posts/P3/003.png)

More info on user secrets can be found in the [Microsoft documentation](https://docs.microsoft.com/en-us/aspnet/core/security/app-secrets?view=aspnetcore-3.1).

## The "secrets.json" file

The good thing of using user secrets is that a `secrets.json file` is created and stored outside of the solution folder in a place not managed by the git versioning system.
- On Windows platform the path is `%APPDATA%\Microsoft\UserSecrets\<user_secrets_id>\secrets.json`
- On Unix and Mac OSX platforms the path is `~/.microsoft/usersecrets/<user_secrets_id>/secrets.json`

The `UserSecretsId` is a Guid (Globally Unique Identifier) assigned during the user secrets initialization and stored on the .csproj file:

![folder creation](/img/posts/P3/004.png)

## MSBuild customization

To add the `secret.json` file as an `EmbeddedResource` to the Xamarin Forms common project we need to execute some steps before the build process:

1) Check that we are building a `debug` version  
2) Verify that the project is using UserSecrets  
3) Add the file to the `EmbeddedResource` file list.

In order to do that, we have multiple choices:

- Modify the `.csproj` file
- Create a `.targets` file and add an `import` command at the end of the `.csproj` file
- Create a `Directory.Build.targets` file on the same folder of the `.csproj` file 

The last one is preferable because we only needs to add a file that we can just copy and paste on every project where we want to use UserSecrets, without touching the `.csproj` file.

When MSBuild runs, *Microsoft.Common.targets* searches the directory structure for the `Directory.Build.targets` file. If it finds one, it imports the targets without the need to explicitly import them on the `.csproj` file. More info about the `Directory.Build.props` and `Directory.Build.targets` files can be found on [Microsoft Docs](https://docs.microsoft.com/en-us/visualstudio/msbuild/customize-your-build).

Here is the `Directory.Build.targets` file that I've composed to implement those steps (also thanks to [Jonathan Dick](https://twitter.com/redth) help):

```xml
<Project>
  <Target Name="AddUserSecrets"
          BeforeTargets="PrepareForBuild"
          Condition=" '$(Configuration)' == 'Debug' And '$(UserSecretsId)' != '' ">
    <PropertyGroup>
      <UserSecretsFilePath Condition=" '$(OS)' == 'Windows_NT' ">
        $([System.Environment]::GetFolderPath(SpecialFolder.UserProfile))\AppData\Roaming\Microsoft\UserSecrets\$(UserSecretsId)\secrets.json
      </UserSecretsFilePath>   
      <UserSecretsFilePath Condition=" '$(OS)' == 'Unix' ">
        $([System.Environment]::GetFolderPath(SpecialFolder.UserProfile))/.microsoft/usersecrets/$(UserSecretsId)/secrets.json
      </UserSecretsFilePath>
    </PropertyGroup>
    <ItemGroup>
      <EmbeddedResource Include="$(UserSecretsFilePath)" Condition="Exists($(UserSecretsFilePath))"/>
    </ItemGroup>
  </Target>
</Project>
```

It's worth noting that it works on both Windows and Unix/OSX platforms and that because it uses the `UserSecretsId` property set on the .csproj file, so it can be copied and used "as is" in any project, without any change.

Having this done, every time the Xamarin Forms common project is built and the conditions are meet, the `secrets.json` file will be embedded into the compiled project.

## Read the "secrets.json" file from the App
Now that the `secrets.json` file has been embedded into the compiled Xamarin Forms common project, we can read his content from our app. As an example of reading from the embedded file, I've used the work done by Andrew Hoefling, well described on his post [Xamarin App Configuration: Control Your App Settings](https://www.andrewhoefling.com/Blog/Post/xamarin-app-configuration-control-your-app-settings) from where I've borrowed just a class that I've renamed `UserSecretsManager`, called from the code behind of the `MainPage` to retrieve the value of `MySecret`.

```csharp
using System;
using System.Diagnostics;
using System.IO;
using System.Reflection;
using Newtonsoft.Json.Linq;

namespace TPCWare.XFUserSecrets.Utils
{
    public class UserSecretsManager
    {
        private static UserSecretsManager _instance;
        private JObject _secrets;

        // Default namespace of the project
        private const string Namespace = "TPCWare.XFUserSecrets";

        // Filename of the embedded resource file
        private const string UserSecretsFileName = "secrets.json"; 

        private UserSecretsManager()
        {
            try
            {
                var assembly = IntrospectionExtensions.GetTypeInfo(typeof(UserSecretsManager)).Assembly;
                var stream = assembly.GetManifestResourceStream($"{Namespace}.{UserSecretsFileName}");
                using (var reader = new StreamReader(stream))
                {
                    var json = reader.ReadToEnd();
                    _secrets = JObject.Parse(json);
                }
            }
            catch (Exception ex)
            {
                Debug.WriteLine($"Unable to load secrets file: {ex.Message}");
            }
        }

        public static UserSecretsManager Settings
        {
            get
            {
                if (_instance == null)
                {
                    _instance = new UserSecretsManager();
                }

                return _instance;
            }
        }

        public string this[string name]
        {
            get
            {
                try
                {
                    var path = name.Split(':');

                    JToken node = _secrets[path[0]];
                    for (int index = 1; index < path.Length; index++)
                    {
                        node = node[path[index]];
                    }

                    return node.ToString();
                }
                catch (Exception)
                {
                    Debug.WriteLine($"Unable to retrieve secret '{name}'");
                    return string.Empty;
                }
            }
        }
    }
}
```

Clearly this just a sample code, on a real app we'll have a mechanism where user secrets will be used to set/ovverride the configuration data on a debug configuration, and a CI/CD pipeline to inject the production configuration data on a release configuration.

## How to use UserSecrets in your Xamarin Forms app

To use User Secrets in your Xamarin Forms app you need to:

1) Init the UserSecrets with Visual Studio (only on your PC) or .NET Core CLI (on your PC or Mac)  
2) Add the `Directory.Build.targets` file at the root of your Xamarin Forms common project

Then you can copy and use the `UserSecretsManager` class to read the secrets.json embedded file content or create something more suited to your needs.

In any case, using the UserSecrets you'll be sure that sensible data won't be inadvertently pushed to the GitHub repository anymore.

That's all folks!