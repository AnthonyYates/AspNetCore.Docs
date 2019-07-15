---
title: Upload files in ASP.NET Core
author: guardrex
description: How to use model binding and streaming to upload files in ASP.NET Core MVC.
monikerRange: '>= aspnetcore-2.1'
ms.author: riande
ms.custom: mvc
ms.date: 05/10/2019
uid: mvc/models/file-uploads
---
# Upload files in ASP.NET Core

By [Luke Latham](https://github.com/guardrex), [Steve Smith](https://ardalis.com/), and [Rutger Storm](https://github.com/rutix)

ASP.NET Core supports uploading one or more files using buffered model binding for smaller files and unbuffered streaming for larger files.

[View or download sample code](https://github.com/aspnet/AspNetCore.Docs/tree/master/aspnetcore/mvc/models/file-uploads/samples/) ([how to download](xref:index#how-to-download-a-sample))

## Security considerations

Use caution when providing users with the ability to upload files to a server. Attackers may execute [denial of service](/windows-hardware/drivers/ifs/denial-of-service) attacks, attempt to upload viruses or malware, or attempt to compromise networks and servers in other ways.

Security steps that reduce the likelihood of a successful attack are:

* Upload files to a dedicated file upload area on the system, preferably to a non-system drive. Use of a dedicated location makes it easier to impose security measures on uploaded files. Disable execute permissions on the file upload location.&dagger;
* Never persist uploaded files in the same directory tree as the app.&dagger;
* Use a safe file name determined by the app. Don't use a file name provided by user input or the untrusted file name of the uploaded file.&dagger;
* Only allow a specific set of approved file extensions.&dagger;
* Check the file format signature to prevent a user from uploading a masqueraded file (for example, uploading an *.exe* file with a *.txt* extension).&dagger;
* Verify that client-side checks are also performed on the server. Client-side checks are easy to circumvent.&dagger;
* Check the size of the upload and prevent uploads that are larger than expected.&dagger;
* When files shouldn't be overwritten by an uploaded file with the same name, check the file name against the database or physical storage before uploading the file.
* **Run a virus/malware scanner on uploaded content before the file is stored.**

&dagger;The sample app demonstrates an approach that meets the criteria.

> [!WARNING]
> Uploading malicious code to a system is frequently the first step to executing code that can:
>
> * Completely takeover a system.
> * Overload a system with the result that the system crashes.
> * Compromise user or system data.
> * Apply graffiti to a public UI.
>
> For information on reducing the attack surface area when accepting files from users, see the following resources:
>
> * [Unrestricted File Upload](https://www.owasp.org/index.php/Unrestricted_File_Upload)
> * [Azure Security: Ensure appropriate controls are in place when accepting files from users](/azure/security/azure-security-threat-modeling-tool-input-validation#controls-users)

For more information on implementing security measures, including examples from the sample app, see the [Validation](#validation) section.

## Storage scenarios

Common storage options for files include:
 
* Database

  * For small file uploads, often faster than physical storage (file system or network share) options.
  * Often more convenient than physical storage options because retrieval of a database record for user data can concurrently supply the file content (for example, an avatar image).
  * Potentially less expensive than using a data storage service.

* Physical storage (file system or network share)

  * For large file uploads:
    * Database limits may restrict the size of the upload.
    * Often less economical than storage in a database.
  * Potentially less expensive than using a data storage service.
  * The app's process must have read and write permissions to the storage location. **Never grant execute permission.**

* Data storage service (for example, [Azure Blob Storage](https://azure.microsoft.com/services/storage/blobs/))

  * Improved scalability and resiliency over on-premises solutions that are often subject to single points of failure.
  * Potentially lower cost in large storage infrastructure scenarios.

  For more information, see [Quickstart: Use .NET to create a blob in object storage](/azure/storage/blobs/storage-quickstart-blobs-dotnet). The topic demonstrates <xref:Microsoft.Azure.Storage.File.CloudFile.UploadFromFileAsync*>, but you can use <xref:Microsoft.Azure.Storage.File.CloudFile.UploadFromStreamAsync*> to save a <xref:System.IO.FileStream> to blob storage when working with a <xref:System.IO.Stream>.

## File upload scenarios

Two general approaches for uploading files are buffering and streaming.

**Buffering**

The entire file is read into an <xref:Microsoft.AspNetCore.Http.IFormFile>, which is a C# representation of the file used to process or save the file.

Any single buffered file exceeding 64 KB is moved from memory to a temp file on disk.

The resources (disk, memory) used by file uploads depend on the number and size of concurrent file uploads. If an app attempts to buffer too many uploads, the site crashes when it runs out of memory or disk space. If the size or frequency of file uploads is exhausting app resources, use streaming.

Buffering small files is covered in the following sections of this topic:

* [Physical storage](#upload-small-files-with-buffered-model-binding-to-physical-storage)
* [Database](#upload-small-files-with-buffered-model-binding-to-a-database)

**Streaming**

The file is received from a multipart request and directly processed or saved by the app. Streaming doesn't improve performance significantly&mdash;streaming reduces the demands for memory or disk space when uploading files.

For more information, see the [Upload large files with streaming](#upload-large-files-with-streaming) section.

### Upload small files with buffered model binding to physical storage

To upload small files, use a multipart form or construct a POST request using JavaScript.

The following example from the sample app demonstrates the use of a Razor Pages form to upload a single file.

*Pages/BufferedSingleFileUploadPhysical.cshtml*:

```cshtml
<form enctype="multipart/form-data" method="post">
    <dl>
        <dt>
            <label asp-for="FileUpload.FormFile"></label>
        </dt>
        <dd>
            <input asp-for="FileUpload.FormFile" type="file">
            <span asp-validation-for="FileUpload.FormFile"></span>
        </dd>
    </dl>
    <input asp-page-handler="Upload" class="btn" type="submit" value="Upload" />
</form>
```

The following example (*not in the sample app*) is analogous to the prior example except that it uses JavaScript to submit the form's data:

```cshtml
<form method="post" action="BufferedSingleFileUploadPhysical?handler=Upload" 
    asp-antiforgery="true" enctype="multipart/form-data" 
    onsubmit="AJAXSubmit(this);return false;">
    <dl>
        <dt>
            <label for="FileUpload_FormFile">File</label>
        </dt>
        <dd>
            <input id="FileUpload_FormFile" type="file" 
                name="FileUpload.FormFile">
        </dd>
    </dl>

    <input class="btn" type="submit" value="Upload">

    <div style="margin-top:15px">
        <output name="result"></output>
    </div>
</form>

@section Scripts {
    @{await Html.RenderPartialAsync("_ValidationScriptsPartial");}

    <script>
        "use strict";

        function AJAXSubmit (oFormElement) {
          var oReq = new XMLHttpRequest();
          oReq.onload = function(e) { 
            oFormElement.elements.namedItem("result").value = 
            'Result: ' + this.status + ' ' + this.statusText;
          };
          oReq.open("post", oFormElement.action);
          oReq.send(new FormData(oFormElement));
        }

        function getCookie (name) {
          var value = "; " + document.cookie;
          var parts = value.split("; " + name + "=");
          if (parts.length == 2) return parts.pop().split(";").shift();
        }
    </script>
}
```

The `<form>` element attribute `asp-antiforgery` is set to `true` so that the form receives an antiforgery request token. The token isn't automatically provided when a form's `action` is set explicitly. Setting the attribute to `true` overrides the default behavior. When the token is present in the form, it's automatically sent with the POST to the server to validate the request. For more information, see <xref:security/anti-request-forgery>.

In order to support file uploads, HTML forms must specify an encoding type (`enctype`) of `multipart/form-data`.

For a `files` input element to support uploading multiple files provide the `multiple` attribute on the `<input>` element:

```cshtml
<input asp-for="FileUpload.FormFiles" type="file" multiple>
```

The individual files uploaded to the server can be accessed through [Model Binding](xref:mvc/models/model-binding) using <xref:Microsoft.AspNetCore.Http.IFormFile>. The sample app demonstrates multiple file uploads for database and physical storage scenarios.

> [!WARNING]
> Don't rely on or trust the `FileName` property of <xref:Microsoft.AspNetCore.Http.IFormFile> without validation. The `FileName` property should only be used for display purposes and only after HTML encoding the value.
>
> The examples provided don't take into account security considerations. See the [Security considerations](#security-considerations) and [Validation](#validation) sections and the sample app for information and examples of file upload validation.

When uploading files using model binding and <xref:Microsoft.AspNetCore.Http.IFormFile>, the action method can accept:

* A single <xref:Microsoft.AspNetCore.Http.IFormFile>.
* An <xref:System.Collections.IEnumerable>\<<xref:Microsoft.AspNetCore.Http.IFormFile>> or [List](xref:System.Collections.Generic.List`1)\<<xref:Microsoft.AspNetCore.Http.IFormFile>> representing several files. 

The following example:

* Loops through one or more uploaded files.
* Saves the files to the local file system.
* Returns the total number and size of files uploaded.

```csharp
public async Task<IActionResult> OnPostUploadAsync(List<IFormFile> files)
{
    long size = files.Sum(f => f.Length);

    // Full path to file, including the file name
    var filePath = Path.GetTempFileName();

    foreach (var formFile in files)
    {
        if (formFile.Length > 0)
        {
            using (var stream = new FileStream(filePath, FileMode.Create))
            {
                await formFile.CopyToAsync(stream);
            }
        }
    }

    // Process uploaded files
    // Don't rely on or trust the FileName property without validation.

    return Ok(new { count = files.Count, size, filePath });
}
```

The path passed to the <xref:System.IO.FileStream> *must* include the file name. If the file name isn't provided, an <xref:System.UnauthorizedAccessException> is thrown at runtime.

Files uploaded using the <xref:Microsoft.AspNetCore.Http.IFormFile> technique are buffered in memory or on disk on the server before processing. Inside the action method, the <xref:Microsoft.AspNetCore.Http.IFormFile> contents are accessible as a <xref:System.IO.Stream>. In addition to the local file system, files can be saved to a network share or to a file storage service, such as [Azure Blob storage](/azure/visual-studio/vs-storage-aspnet5-getting-started-blobs).

> [!WARNING]
> [Path.GetTempFileName](xref:System.IO.Path.GetTempFileName*) throws an <xref:System.IO.IOException> if more than 65,535 files are created without deleting previous temporary files. The limit of 65,535 files is a per-server limit. For more information on this limit on Windows OS, see the remarks in the following topics:
>
> * [GetTempFileNameA function](/windows/desktop/api/fileapi/nf-fileapi-gettempfilenamea#remarks)
> * <xref:System.IO.Path.GetTempFileName*>

### Upload small files with buffered model binding to a database

To store binary file data in a database using [Entity Framework](/ef/core/index), define a <xref:System.Byte> array property on the entity. The following example is similar to a scenario demonstrated in the sample app (*Upload one buffered file with one file upload control*):

```csharp
public class AppFile
{
    public int Id { get; set; }
    public byte[] Content { get; set; }
}
```

Specify a page model property for the class that includes an <xref:Microsoft.AspNetCore.Http.IFormFile>:

```csharp
public class BufferedSingleFileUploadDbModel : PageModel
{
    ...

    [BindProperty]
    public BufferedSingleFileUploadDb FileUpload { get; set; }

    ...
}

public class BufferedSingleFileUploadDb
{
    [Required]
    [Display(Name="File")]
    public IFormFile FormFile { get; set; }
}
```

> [!NOTE]
> <xref:Microsoft.AspNetCore.Http.IFormFile> can be used directly as an action method parameter or as a bound model property. The prior example uses a bound model property.

The `FileUpload` is used in the Razor Pages form:

```cshtml
<form enctype="multipart/form-data" method="post">
    <dl>
        <dt>
            <label asp-for="FileUpload.FormFile"></label>
        </dt>
        <dd>
            <input asp-for="FileUpload.FormFile" type="file">
        </dd>
    </dl>
    <input asp-page-handler="Upload" class="btn" type="submit" value="Upload">
</form>
```

When the form is POSTed to the server, copy the <xref:Microsoft.AspNetCore.Http.IFormFile> to a stream and save it as a byte array in the database:

```csharp
public async Task<IActionResult> OnPostUploadAsync()
{
    using (var memoryStream = new MemoryStream())
    {
        await FileUpload.FormFile.CopyToAsync(memoryStream);
    
        var file = new AppFile()
        {
            Content = memoryStream.ToArray()
        };
        
        _context.File.Add(file);
        await _context.SaveChangesAsync();
    }
}
```

> [!WARNING]
> Use caution when storing binary data in relational databases, as it can adversely impact performance.
>
> Don't rely on or trust the `FileName` property of <xref:Microsoft.AspNetCore.Http.IFormFile> without validation. The `FileName` property should only be used for display purposes and only after HTML encoding.
>
> The examples provided don't take into account security considerations. See the [Security considerations](#security-considerations) and [Validation](#validation) sections and the sample app for information and examples of file upload validation.

### Upload large files with streaming

The following example demonstrates using JavaScript to stream a file to a controller action. The file's antiforgery token is generated using a custom filter attribute and passed to the client HTTP headers instead of in the request body. Because the action method processes the uploaded data directly, model binding is disabled by another custom filter. Within the action, the form's contents are read using a `MultipartReader`, which reads each individual `MultipartSection`s, processing the file or storing the contents as appropriate. Once all of the sections are read, the action performs its own model binding.

The initial page response loads the form and saves an antiforgery token in a cookie (via the `GenerateAntiforgeryTokenCookieAttribute` attribute). The attribute uses ASP.NET Core's built-in [antiforgery support](xref:security/anti-request-forgery) to set a cookie with a request token:

[!code-csharp[](file-uploads/samples/2.x/SampleApp/Filters/Antiforgery.cs?name=snippet_GenerateAntiforgeryTokenCookieAttribute)]

The `DisableFormValueModelBindingAttribute` is used to disable model binding:

[!code-csharp[](file-uploads/samples/2.x/SampleApp/Filters/ModelBinding.cs?name=snippet_DisableFormValueModelBindingAttribute)]

`GenerateAntiforgeryTokenCookieAttribute` and `DisableFormValueModelBindingAttribute` are applied as filters to the page application models of `/StreamedSingleFileUploadDb` and `/StreamedSingleFileUploadPhysical` in `Startup.ConfigureServices` using [Razor Pages conventions](xref:razor-pages/razor-pages-conventions):

[!code-csharp[](file-uploads/samples/2.x/SampleApp/Startup.cs?name=snippet_AddMvc&highlight=8-11,17-20)]

Since model binding is disabled, the action method doesn't accept parameters. The action method works directly with the `Request` property. A `MultipartReader` is used to read each section. Key/value data is stored in a `KeyValueAccumulator`. After all of the sections are read, the contents of the `KeyValueAccumulator` are used to bind the form data to a model type.

The complete `StreamingController.UploadDatabase` method for streaming to a database with EF Core:

[!code-csharp[](file-uploads/samples/2.x/SampleApp/Controllers/StreamingController.cs?name=snippet_UploadDatabase)]

The complete `StreamingController.UploadPhysical` method for streaming to a physical location:

[!code-csharp[](file-uploads/samples/2.x/SampleApp/Controllers/StreamingController.cs?name=snippet_UploadPhysical)]

In the sample app, validation checks are handled by `FileHelpers.ProcessStreamedFile`.

For more information, see the [Security considerations](#security-considerations) and [Validation](#validation) sections and the sample app.

## Validation

The sample app's `FileHelpers` class demonstrates a several checks for buffered <xref:Microsoft.AspNetCore.Http.IFormFile> and streamed file uploads. For processing <xref:Microsoft.AspNetCore.Http.IFormFile> buffered file uploads in the sample app, see the `ProcessFormFile` method in the *Utilities/FileHelpers.cs* file. For processing streamed files, see the `ProcessStreamedFile` method in the same file.

> [!WARNING]
> The validation processing methods demonstrated in the sample app don't scan the content of uploaded files. In most production scenarios, a virus/malware scanner API is used on the file before making the file available to users or other systems.

### Content validation

Use a third party virus/malware scanning API on uploaded content.

Scanning files is demanding on server resources in high volume scenarios. If request processing performance is diminished due to file scanning, consider offloading the scanning work to a [background service](xref:fundamentals/host/hosted-services), possibly a service running on a server different from the app's server. Typically, uploaded files are held in a quarantined area until the background virus scanner checks them. When a file passes, the file is moved to the normal file storage location. These steps are usually performed in conjunction with a database record that indicates the scanning status of a file. By using such an approach, the app and app server remain focused on responding to requests.

### File extension validation

The uploaded file's extension should be checked against a list of permitted extensions. The sample app uses an array to determine if an extension is valid:

```csharp
private string[] permittedExtensions = { ".txt", ".pdf", };

var ext = Path.GetExtension(uploadedFileName);

if (string.IsNullOrEmpty(ext) || !permittedExtensions.Contains(ext))
{
    // The extension is invalid ... discontinue processing the file
}
```

### File signature validation

A file's signature is determined by the first few bytes that make up the file. These bytes can be used to indicate if the extension matches the content of the file. The sample app checks file signatures for common file types. In the following example, the file signature for a Microsoft Word document is checked against the file:

```csharp
private Dictionary<string, byte[]> _fileSignature = 
    new Dictionary<string, byte[]>
{
    { ".docx", new byte[] { 0x50, 0x4B, 0x03, 0x04 } },
};

using (var reader = new BinaryReader(uploadedFileData))
{
    var signature = _fileSignature[uploadedFileExtension];
    reader.BaseStream.Position = 0;

    if (!signature.SequenceEqual(reader.ReadBytes(signature.Length)))
    {
        // The file's signature doesn't match its extension ...
        // discontinue processing the file
    }
}
```

### Name validation

Remove invalid characters from file names with [Path.GetInvalidFileNameChars](xref:System.IO.Path.GetInvalidFileNameChars*) and [Linq](/dotnet/csharp/programming-guide/concepts/linq/):

```csharp
var invalidFileNameChars = Path.GetInvalidFileNameChars();
var trustedFileName = WebUtility.HtmlEncode(
        invalidFileNameChars.Aggregate(
            formFile.FileName, (current, c) => 
                current.Replace(c, '_')));
```

In a production scenario, further processing or different processing is probably required. For example, an app can supply a GUID for the file name and only use the human-readable file name for display. Even if the invalid characters of a file name are removed, it's recommended to set a minimum and maximum file name length. Many implementations must include a check that the file exists; otherwise, the file is overwritten by a file of the same name. The sample app only demonstrates removal of invalid characters. Supply additional logic to meet your app's specifications.

### Size validation

Limit the size of uploaded files.

In the sample app, the size of the file is limited to 2 MB (indicated in bytes). The limit is supplied via [Configuration](xref:fundamentals/configuration/index) from the *appsettings.json* file:

```json
{
  "FileSizeLimit": 2097152
}
```

```csharp
public class BufferedSingleFileUploadPhysicalModel : PageModel
{
    private readonly long _fileSizeLimit;

    public BufferedSingleFileUploadPhysicalModel(IConfiguration config)
    {
        _fileSizeLimit = config.GetValue<long>("FileSizeLimit");\
    }

    ...
}
```

```csharp
if (formFile.Length > _fileSizeLimit)
{
    // The file is too large ... discontinue processing the file
}
```

### Match name attribute value to parameter name of POST method

In non-Razor forms that POST form data or use JavaScript's `FormData` directly, the name specified in the form's element or `FormData` must match the name of the parameter in the controller's action.

In the following example:

* When using an `<input>` element, the `name` attribute is set to the value `battlePlans`:

  ```html
  <input type="file" name="battlePlans" multiple>
  ```

* When using `FormData` in JavaScript, the name is set to the value `battlePlans`:

  ```javascript
    var formData = new FormData();

    for (var file in files) {
        formData.append("battlePlans", file, file.name);
    }
    ```

Use a matching name for the parameter of the C# method (`battlePlans`):

* For a Razor Pages page handler method named `Upload`:

  ```csharp
  public async Task<IActionResult> OnPostUploadAsync(List<IFormFile> battlePlans)
  ```

* For an MVC POST controller action method:

  ```csharp
  public async Task<IActionResult> Post(List<IFormFile> battlePlans)
  ```

## Server and app configuration

### Multipart body length limit

<xref:Microsoft.AspNetCore.Http.Features.FormOptions.MultipartBodyLengthLimit> sets the limit for the length of each multipart body. Form sections that exceed this limit throw an <xref:System.IO.InvalidDataException> when parsed. The default is 134,217,728 (128 MB). Customize the limit using the <xref:Microsoft.AspNetCore.Http.Features.FormOptions.MultipartBodyLengthLimit> setting in `Startup.ConfigureServices`:

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddMvc();
    services.Configure<FormOptions>(options =>
    {
        // Set the limit to 256 MB
        options.MultipartBodyLengthLimit = 268435456;
    });
}
```

<xref:Microsoft.AspNetCore.Mvc.RequestFormLimitsAttribute> is used to set the <xref:Microsoft.AspNetCore.Http.Features.FormOptions.MultipartBodyLengthLimit> for a single page or action.

In a Razor Pages app, apply the filter with a [convention](xref:razor-pages/razor-pages-conventions) in `Startup.ConfigureServices`:

```csharp
services.AddMvc()
    .AddRazorPagesOptions(options =>
        {
            options.Conventions
                .AddPageApplicationModelConvention("/FileUploadPage",
                    model.Filters.Add(
                        new RequestFormLimitsAttribute()
                        {
                            // Set the limit to 256 MB
                            MultipartBodyLengthLimit = 268435456
                        });
        })
    .SetCompatibilityVersion(CompatibilityVersion.Version_2_2);
```

In a Razor pages app or an MVC app, apply the filter to the page handler class or action method:

```csharp
// Set the limit to 256 MB
[RequestFormLimits(MultipartBodyLengthLimit = 268435456)]
public class BufferedSingleFileUploadPhysicalModel : PageModel
{
    ...
}
```

### Kestrel maximum request body size

The default maximum request body size is 30,000,000 bytes, which is approximately 28.6 MB. Customize the limit using the [MaxRequestBodySize](xref:fundamentals/servers/kestrel#maximum-request-body-size) Kestrel server option:

::: moniker range=">= aspnetcore-3.0"

```csharp
public static IHostBuilder CreateHostBuilder(string[] args) =>
    Host.CreateDefaultBuilder(args)
        .ConfigureWebHostDefaults(webBuilder =>
        {
            webBuilder.UseStartup<Startup>();
        })
        .ConfigureKestrel((context, options) =>
        {
            // Handle requests up to 50 MB
            options.Limits.MaxRequestBodySize = 52428800;
        });
```

::: moniker-end

::: moniker range="< aspnetcore-3.0"

```csharp
public static IWebHostBuilder CreateWebHostBuilder(string[] args) =>
    WebHost.CreateDefaultBuilder(args)
        .UseStartup<Startup>()
        .ConfigureKestrel((context, options) =>
        {
            // Handle requests up to 50 MB
            options.Limits.MaxRequestBodySize = 52428800;
        });
```

::: moniker-end

The Kestrel limits for [maximum client connections](xref:fundamentals/servers/kestrel#maximum-client-connections) may also apply.

<xref:Microsoft.AspNetCore.Mvc.RequestSizeLimitAttribute> is used to set the [MaxRequestBodySize](xref:fundamentals/servers/kestrel#maximum-request-body-size) for a single page or action.

In a Razor Pages app, apply the filter with a [convention](xref:razor-pages/razor-pages-conventions) in `Startup.ConfigureServices`:

```csharp
services.AddMvc()
    .AddRazorPagesOptions(options =>
        {
            options.Conventions
                .AddPageApplicationModelConvention("/FileUploadPage",
                    model =>
                    {
                        // Handle requests up to 50 MB
                        model.Filters.Add(
                            new RequestSizeLimitAttribute(52428800));
                    });
        })
    .SetCompatibilityVersion(CompatibilityVersion.Version_2_2);
```

In a Razor pages app or an MVC app, apply the filter to the page handler class or action method:

```csharp
// Handle requests up to 50 MB
[RequestSizeLimit(52428800)]
public class BufferedSingleFileUploadPhysicalModel : PageModel
{
    ...
}
```

### IIS content length limit

The default request limit (`maxAllowedContentLength`) is 30,000,000 bytes, which is approximately 28.6MB. Customize the limit in the *web.config* file:

```xml
<system.webServer>
  <security>
    <requestFiltering>
      <!-- Handle requests up to 50 MB -->
      <requestLimits maxAllowedContentLength="52428800" />
    </requestFiltering>
  </security>
</system.webServer>
```

This setting only applies to IIS. The behavior doesn't occur by default when hosting on Kestrel. For more information, see [Request Limits \<requestLimits>](/iis/configuration/system.webServer/security/requestFiltering/requestLimits/).

## Troubleshoot

Below are some common problems encountered when working with uploading files and their possible solutions.

### Not Found error when deployed to an IIS server

The following error indicates that the uploaded file exceeds the server's configured content length:

```
HTTP 404.13 - Not Found
The request filtering module is configured to deny a request that exceeds the request content length.
```

For more information on increasing the limit, see the [IIS content length limit](#iis-content-length-limit) section.

### Connection failure

A connection error and a reset server connection probably indicates that the uploaded file exceeds Kestrel's maximum request body size. For more information, see the [Kestrel maximum request body size](#kestrel-maximum-request-body-size) section. Kestrel client connection limits may also require adjustment.

### Null Reference Exception with IFormFile

If the controller is accepting uploaded files using <xref:Microsoft.AspNetCore.Http.IFormFile> but the value is `null`, confirm that the HTML form is specifying an `enctype` value of `multipart/form-data`. If this attribute isn't set on the `<form>` element, the file upload doesn't occur and any bound <xref:Microsoft.AspNetCore.Http.IFormFile> arguments are `null`. Also confirm that the [upload naming in form data matches the app's naming](#match-name-attribute-value-to-parameter-name-of-post-method).

## Additional resources

* [Unrestricted File Upload](https://www.owasp.org/index.php/Unrestricted_File_Upload)
* [Azure Security: Security Frame: Input Validation | Mitigations](/azure/security/azure-security-threat-modeling-tool-input-validation)
* [Azure Cloud Design Patterns: Valet Key pattern](/azure/architecture/patterns/valet-key)
