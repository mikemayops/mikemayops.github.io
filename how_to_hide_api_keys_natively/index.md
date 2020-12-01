# How to Hide Api Keys Natively


<div align="center" ><em>Photo by Christina Morillo from Pexels</em></div>

\
If you're like me you have many API integrations you've created, and for each of these you probably need to keep some kind of credential or secret key that needs to remain hidden. Furthermore, you probably have a bunch on these scripts running on scheduled tasks on some server. As you probably know, it is not a good idea to leave this key in clear text within your scripts. How do you keep all your keys safe?

There are a few very nice PowerShell modules being developed that help you with managing secrets like [Microsoft.PowerShell.SecretManagement][1] or [PSSecretStore][2]. However, I always like to find out first how much I can accomplish with PowerShell natively so then I don't have to worry about installing or keeping track of  dependencies on my servers. In this post I'm covering a native way to encrypt and store your secrets, or any other kind of credential, only using a couple of PowerShell functions. 

{{<admonition tip "Key Cmdlets and .Net Types Used in this Post" false>}}
-  [`ConvertTo-SecureString`](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.security/convertto-securestring?view=powershell-5.1)
-  [`ConvertFrom-SecureString`](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.security/convertfrom-securestring?view=powershell-5.1)
-  [`[System.Runtime.InteropServices.Marshal]`](https://docs.microsoft.com/en-us/dotnet/api/system.runtime.interopservices.marshal?view=net-5.0)
{{</admonition>}}

## Encrypting Secrets
First, we need to create a script to encrypt our secrets so we can store them safely in a config file. In some cases, it would also be a good idea to hide our API *Url*. Lets say these are the secrets we need to hide:

\
&nbsp;&nbsp; **Url:**     ht<span>tps://</span>website.com/api/v1

&nbsp;&nbsp; **Key:**     siufj0a0s83hascn87cha98x=

\
We're going to be using `ConvertTo-SecureString` and `ConvertFrom-SecureString` to encrypt the strings (click on each for more info):

```powershell

# For demonstration purposes we'll keep the strings clear text for now
$url = "https://website.com/api/v1s"
$key = "U3VwZXJTZWNyZXRQYXNzd29yZCE="

# Lets store the values in a Hashtable so we can later convert to Json (more on that later)
$secureValue = @{
    url = $null
    key = $null
}

# Encrypt the values
$secureValue.url = $url | ConvertTo-SecureString -AsPlainText -Force | ConvertFrom-SecureString
$secureValue.key = $key | ConvertTo-SecureString -AsPlainText -Force | ConvertFrom-SecureString

```
Now, before we move on, there's something that it's important to note from above. All that `ConvertTo-SecureString` is doing is obfuscating the string, it is _**not**_ encrypting its conents. By obfuscating, all the PowerShell host is going to do is print `System.Secure.SecureString` when referencing it. It's placing this value in a "Special" place in memory:

\
{{< figure src="SecureStringEx1.JPG" title="Secure String" >}}

To actually encrypt the string ( as counter intuitive as it sounds ) you need to pass this generated `SecureSting` to  `ConvertFrom-SecureString`. This was a bit confusing for me at first because "obviously" if you want to convert *from* a `SecureString` it will just turn into the ***un-secure*** version of it, duh! But actually, this command is *encrypting* the string. 

{{<admonition info "Important">}}
If no key is specified the `ConvertFrom-SecureString` Cmdlet encrypts the data with the Windows Data Protection API (DPAPI), which uses  _**Machine**_ and _**User**_ specific cyphers, meaning that you can't decypher the generated encrypted string from a different machine if the file is copied. 
{{</admonition>}}

## Saving Your Secrets
Now that this is out of the way and the values are encrypted, lets save those values as a Json file within the user's `%localappdata%` folder:

```powershell

# Cleanup and create new folder in LocalAppData folder
Remove-Item -Path "$env:localappdata\ApiConfig" -Force -ErrorAction Ignore
New-Item -Path "$env:localappdata\ApiConfig" -Force -ItemType Directory

$secureValue | ConvertTo-Json | Out-File (Join-Path $env:localappdata "ApiConfig\ApiConfig.json")

```

Now lets turn this into a simple script and call it `NewApiConfigFile.ps1`. This script will be very useful when creating scheduled tasks on servers running under service account. In order to keep it safe and simple we will be prompting for the secure values interactively (using `Read-Host`) so we only need to provide the secrets once. In you own environment, feel free to add as many secrure values as you need:

\
{{< gist mikemayops be6ed68836c0c8d5780f6322d267e88e >}}


\
Note in the above script I found that we're able to use the parameter `-AsSecureString` on the `Read-Host` Cmdlet, this will obfuscate the charactes when typed and convert the values to a Secure String, we no longer need to run `ConvertTo-SecureString`! Also, I added a prompt for a **Configuration Name** that will be used to create the folder and Json file name. We will need to make a note of this value since we're going to need it later.

\
{{< figure src="NewApiConfigEx.JPG" title="NewApiConfigFile.ps1 in Action" >}}


Now you have a script you can run once from the PowerShell console save your Secrets. Just make sure you run it on the computer and under the user account you intend to run your API scripts!

## Retrieving Your Secrets

Now that we have our API keys tucked away safely we need a way to un-encrypt and use them. For this we're making use of `[System.Runtime.InteropServices.Marshal]` .Net Type. For this example, I'm leaning towards making the unecryption process Function since you would pontentially be needing to retrieve the secrets multiple times throughout your script.

\
{{< gist mikemayops 04013275455c7ff4a9591b525fb3c579 >}}



\
Now that you've decrypted your ***Key*** and ***Url*** you can use it to later on your script build your API Authorization headers. But, before we call it a day, it's important that after you're done using your secrets you clear the encrypted variables from memory :(far fa-smile-wink):: 


```powershell
# Clear encrypted variables from memory
$Marshal::ZeroFreeBSTR($BstrUrl)
$Marshal::ZeroFreeBSTR($BstrKey)
$Script:ApiUrl = $null
$Script:ApiKey = $null

```
## Conclusion
\
Now that you have an easy and native way to store and retrive your secrets. You can run `NewApiConfigFile.ps1` as your service account on the *Computer* and as the *User* running the task. This is a fast an easy way to keep your secrets safe and it's even better if you integrate these into your existing modules as you'll be able to keep these secrets within your module's scope (`$Script:Variable`) and add an exra layer of security. 


[1]: https://www.powershellgallery.com/packages/Microsoft.PowerShell.SecretManagement/0.5.5-preview6
[2]: https://www.powershellgallery.com/packages/PSSecretStore/0.0.3
