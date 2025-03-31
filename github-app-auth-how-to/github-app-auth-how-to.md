# How to Authenticate GitHub Apps: A Practical Guide

This is a practical guide to GitHub client flow authentication. GitHub Apps can do much more than this, but we're going to quickly cover only one case: authenticating a backend service, integration, or any application that acts and authenticates as an application against GitHub.

## Create a New GitHub App on GitHub

### Quick Creation Guide

**Navigate to Settings -> Developer settings -> GitHub Apps.**

!alt text

Click on **New GitHub App**, give it a name, a description (optional), a homepage URL (if you don't have any frontend, simply add your repository URL), and assign the needed permissions (remember the principle of least privilege).

### Install Concept

If you're here, you probably need the app to authenticate itself to GitHub and perform actions as a backend service. You'll need to install the app to authenticate it and perform actions as a backend service. The installation concept allows the app to work and be deployed at a user or organization level scope. 

It won't install your backend service, integration, or whatever, but **bind this "principal" within a high-level scope on GitHub.** We may say that **the installation is some sort of instance of the app within a scope.**

### Back to Quick Creation Guide

Choose whether your app can be installed only in your account/org or by any user/organization (you'll probably opt for the first). Create the app.

## Generate a Private Key

Go to the newly created GitHub app, scroll down, and create a new private key. This will automatically download a `.pem` file you'll need to generate a token.

## Install the Application

On the left menu, click on **Install App**. You'll see all the scopes where you can install your app (allowed scope depends on the choice you made two steps above). Install the app wherever best suits your needs!

## Generate an App Token

Now it's time to write some code. Here's a C# version, but you can adapt it to your preferred language or ask Copilot to help translate it. 😄

```csharp
using System.Security.Cryptography;
using System.Text;
using System.Text.Json;

var appId = "your-app-id";
var privateKeyPath = "path-to-your-pem-file";

// Create JWT header
var header = new
{
    alg = "RS256",
    typ = "JWT"
};
var headerJson = JsonSerializer.Serialize(header);
var headerBase64 = Base64UrlEncode(Encoding.UTF8.GetBytes(headerJson));

// Create JWT payload
var payload = new
{
    iat = DateTimeOffset.UtcNow.AddSeconds(-10).ToUnixTimeSeconds(),
    exp = DateTimeOffset.UtcNow.AddMinutes(10).ToUnixTimeSeconds(), // JWT expiration time (10 minutes)
    iss = appId
};
var payloadJson = JsonSerializer.Serialize(payload);
var payloadBase64 = Base64UrlEncode(Encoding.UTF8.GetBytes(payloadJson));

// Read private key and sign the data
var privateKey = File.ReadAllText(privateKeyPath);
using RSA rsa = RSA.Create();
rsa.ImportFromPem(privateKey);
var dataToSign = $"{headerBase64}.{payloadBase64}";
byte[] signatureBytes = rsa.SignData(Encoding.UTF8.GetBytes(dataToSign), HashAlgorithmName.SHA256, RSASignaturePadding.Pkcs1);
var signatureBase64 = Base64UrlEncode(signatureBytes);

// Combine header, payload, and signature to form the JWT
var jwt = $"{headerBase64}.{payloadBase64}.{signatureBase64}";
Console.WriteLine(jwt);

// Helper method for URL-safe Base64 encoding
string Base64UrlEncode(byte[] input)
{
    return Convert.ToBase64String(input)
        .TrimEnd('=')
        .Replace('+', '-')
        .Replace('/', '_');
}
```

You already have the `.pem` file saved on your file system, downloaded during the step above, and can get the app ID from the general tab of your GitHub app. Note that the App ID is different from the Client ID; you want to use the App ID.

(Note also that this code handles the token expiration; adjust it to your own needs!)

Now you have the application token, but one last step is still needed!

## Get the Installation Token

Now it's time to get the specific installation token. If you remember from above, the installation is our specific instance of a GitHub App within a certain scope. **We're now getting a token to operate within that scope.**

Make a POST HTTP call:

```bash
POST https://api.github.com/app/installations/:installation_id/access_tokens
```

Use the JWT generated beforehand with the script as the Authorization header with the 'Bearer ' prefix.

Finding the installation ID can be tricky! If you're having a hard time finding it, go back to **Settings -> Developer settings -> GitHub Apps -> Install App**. Here you'll see all the installations. Click on the gear icon; this will bring you to the installation page. Take a look at the URL in your browser, and you'll have the installation ID.

The URL will follow this pattern:

```
https://github.com/organizations/your-scope/settings/installations/the-installation-id
```

This API call response is going to contain an access token with the permissions you set initially when you created the GitHub App within the scope defined by the installation.

**You can now use the access token to authenticate the client against GitHub and integrate with GitHub APIs. 😄**

(Just to be super clear: use this access token in the Authorization header of your next API calls with the 'Bearer ' prefix)

## Summary:

- Create and configure a GitHub App
- Generate a private key
- Install the app within a scope
- Generate an app JWT
- Get the installation token
- Use it!