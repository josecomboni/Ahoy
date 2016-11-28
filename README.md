Swashbuckle
=========

[![Build status](https://ci.appveyor.com/api/projects/status/xpsk2cj1xn12c0r7?svg=true)](https://ci.appveyor.com/project/domaindrivendev/ahoy)

[Swagger](http://swagger.io) tooling for API's built with ASP.NET Core. Generate beautiful API documentation, including a UI to explore and test operations, directly from your routes, controllers and models.

In addition to its [Swagger](http://swagger.io/specification/) generator, Swashbuckle also provides an embedded version of the awesome [swagger-ui](https://github.com/swagger-api/swagger-ui) that's powered by the generated Swagger JSON. This means you can complement your API with living documentation that's always in sync with the latest code. Best of all, it requires minimal coding and maintenance, allowing you to focus on building an awesome API.

And that's not all ...

Once you have an API that can describe itself in Swagger, you've opened the treasure chest of Swagger-based tools including a client generator that can be targeted to a wide range of popular platforms. See [swagger-codegen](https://github.com/swagger-api/swagger-codegen) for more details.

# Getting Started #

1. Install the standard Nuget package into your ASP.NET Core application.

    ```
    Install-Package Swashbuckle
    ```

2. In the _ConfigureServices_ method of _Startup.cs_, register the Swagger Generator, defining one or more Swagger documents.

    ```csharp
    services.AddSwaggerGen(c =>
    {
        c.SwaggerDoc("v1", new Info { Title = "My API", Version = "v1" });
    });
    ```

3. In the _Configure_ method, insert middleware to expose the generated Swagger as JSON endpoint(s)

    ```csharp
    app.UseSwagger();
    ```

    _At this point, you can spin up your application and view the generated Swagger JSON at "/swagger/v1/swagger.json."_

4. Optionally insert the swagger-ui middleware if you want to expose interactive documentation, specifying the Swagger JSON endpoint(s) to power it from.

    ```csharp
    app.UseSwaggerUi(c =>
    {
        c.SwaggerEndpoint("/swagger/v1/swagger.json", "My API V1");
    })
    ```

    _Now you can restart your application and check out the auto-generated, interactive docs at "/swagger"._

# The Anatomy of Swashbuckle #

Swashbuckle consists of three packages - a Swagger generator, middleware to expose the generated Swagger as JSON endpoints and middleware to expose a swagger-ui that's powered by those endpoints. They can be installed together, via the "Swashbuckle" meta-package, or independently according to your specific requirements. See the table below for more details.

|Package|Description|
|---------|-----------|
|__Swashbuckle.Swagger__|Exposes C# SwaggerDocuments as a JSON API. It expects an implementation of ISwaggerProvider to be registered and queries it to retrieve documents by name before returning as serialized JSON|
|__Swashbuckle.SwaggerGen__|Injects an implementation of ISwaggerProvider that can be used by the above component. This particular implementation generates SwaggerDocument(s) directly from your routes, controllers and models|
|__Swashbuckle.SwaggerUi__|Exposes an embedded version of the swagger-ui. You specify the API endpoints where it can obtain Swagger JSON and it uses them to power interactive docs for your API|

# Configuration & Customization #

The steps described above will get you up and running with minimal setup. However, Swashbuckle offers a lot of flexibility to customize as you see fit. Check out the table below for a complete listing of configuration 
and customization options:

* [Swashbuckle.Swagger](#)
 
    * [Change the Path for Swagger JSON Endpoints](#)
    * [Dynamic Swagger with Request Context](#)
 
* [Swashbuckle.SwaggerGen](#)
 
    * [List Operations Responses](#)
    * [Include Descriptions from XML Comments](#)
    * [Provide Global API Metadata](#)
    * [Generate Multiple Swagger Documents](#)
    * [Omit Obsolete Operations and/or Schema Properties](#)
    * [Omit Arbitrary Operations](#)
    * [Customize Operation Tags (for UI Grouping)](#)
    * [Change Operations Sort Order](#)
    * [Customize Schema Id's](#)
    * [Customize Schema for Enum Types](#)
    * [Override Schema for Specific Types](#)
    * [Extend Generator with Operation, Schema & Document Filters](#)
    * [Add Security Definitions and Requirements](#)

* [Swashbuckle.SwaggerUi](#)
    * [Change the Path for Accessing the UI](#)

## Swashbuckle.Swagger ##

### Change the Path for Swagger JSON Endpoints ###

By default, Swagger JSON will be exposed at the following route - "/swagger/{documentName}/swagger.json". If neccessary, you can alter this when enabling the Swagger middleware. Custom routes MUST include the {documentName} parameter.

```csharp
app.UseSwagger("api-docs/{documentName}.json");
```

_NOTE: If you're using the SwaggerUi middleware, you'll also need to update it's configuration to reflect the new endpoints:_

```csharp
app.UseSwaggerUi(c =>
{
    c.SwaggerEndpoint("/api-docs/v1.json", "My API V1");
})
```

### Dynamic Swagger with Request Context ###

If you need to set some Swagger metadata based on the current request, you can configure a document filter that's executed prior to deserialization:

```csharp
app.UseSwagger(documentFilter: (swaggerDoc, req) =>
{
    swaggerDoc.Host = req.Host.Value;
})
```

The SwaggerDocument and the current HttpRequest are passed to the filter. This provides a lot of flexibilty. For example, you can assign the "host" property (as shown) or you could inspect session information or an Authoriation header and remove operations based on user permissions. 

## Swashbuckle.SwaggerGen ##

### List Operation Responses ###

By default, Swashbuckle will generate a "200" response for each operation. If the action returns a response DTO, then this will be used to generate a "schema" for the response body. For example ...

```csharp
[HttpPost("{id}")]
public Product GetById(int id)
```

Will produce the following response metadata:

```
responses: {
  200: {
    description: "Success",
    schema: {
      $ref: "#/definitions/Product"
    }
  }
}
```

#### Explicit Responses ####

If you need to specify a different status code and/or additional responses, or your actions return _IActionResult_ instead of a response DTO, you can describe explicit responses with the _ProducesResponseTypeAttribute_ that ships with ASP.NET Core. For example ...

```csharp
[HttpPost("{id}")]
[ProducesResponseType(typeof(Product), 200)]
[ProducesResponseType(typeof(IDictionary<string, string>), 400)]
[ProducesResponseType(typeof(void), 500)]
public IActionResult GetById(int id)
```

Will produce the following response metadata:

```
responses: {
  200: {
    description: "Success",
    schema: {
      $ref: "#/definitions/Product"
    }
  },
  400: {
    description: "Bad Request",
    schema: {
      type: "object",
      additionalProperties: {
        type: "string"
      }
    }
  },
  500: {
    description: "Server Error"
  }
}
```

### Include Descriptions from XML Comments ###

To enhance the generated docs with human-friendly descriptions, you can annotate controllers and models with [Xml Comments](http://msdn.microsoft.com/en-us/library/b2s063f7(v=vs.110).aspx), and configure Swashbuckle to incorporate those comments into the outputted Swagger JSON:

1. Open the Properties dialog for your project, click the "Build" tab and ensure that "XML documentation file" is checked. This will produce a file containing all XML comments at build-time.

    _At this point, any classes or methods that are NOT annotated with XML comments will trigger a build warning. To supress this, enter the warning code "1591" into the "Supress warnings" field in the properties dialog._

2. Configure Swashbuckle to incorporate the XML comments on file into the generated Swagger JSON:

    ```csharp
    services.AddSwaggerGen(c =>
    {
        c.SwaggerDoc("v1",
            new Info
            {
                Title = "My API - V1",
                Version = "v1"
            }
         );

         var filePath = Path.Combine(PlatformServices.Default.Application.ApplicationBasePath, "MyApi.xml");
         c.IncludeXmlComments(filePath);
    }
    ```

3. Annotate your actions with summary, remarks and response tags

    ```csharp
    /// <summary>
    /// Retrieves a specific product by unique id
    /// </summary>
    /// <remarks>Awesomeness!</remarks>
    /// <response code="200">Product created</response>
    /// <response code="400">Product has missing/invalid values</response>
    /// <response code="500">Oops! Can't create your product right now</response>
    [HttpGet("{id}")]
    [ProducesResponseType(typeof(Product), 200)]
    [ProducesResponseType(typeof(IDictionary<string, string>), 400)]
    [ProducesResponseType(typeof(void), 500)]
    public Product GetById(int id)
    ```

4. Rebuild your project to update the XML Comments file and navigate to the Swagger JSON endpoint. Note how the descriptions are mapped onto corresponding Swagger fields.

_You can also provide Swagger Schema descriptions by annotating your API models and their properties with summary tags. If you have multiple XML comments files (e.g. separate libraries for controllers and models), you can invoke the IncludeXmlComments method multiple times and they will all be merged into the outputted Swagger JSON._

### Provide Global API Metadata ###

In addition to Paths, Operations and Responses, which Swashbuckle generates for you, Swagger also supports global metadata (see http://swagger.io/specification/#swaggerObject). For example, you can provide a full description for your API, termsOfService or even contact and licensing information:

```csharp
c.SwaggerDoc("v1",
    new Info
    {
        Title = "My API - V1",
        Version = "v1",
        Description = "A sample API to demo Swashbuckle",
        TermsOfService = "Knock yourself out",
        Contact = new Contact
        {
            Name = "Joe Developer",
            Email = "joe.developer@tempuri.org"
        },
        License = new License
        {
            Name = "Apache 2.0",
            Url = "http://www.apache.org/licenses/LICENSE-2.0.html"
        }
    }
)
```

_Use IntelliSense to see what other fields are available._

### Generate Multiple Swagger Documents ###

With the setup described above, the generator will include all API operations in a single Swagger document. However, you can create multiple documents if necessary. For example, you may want a separate document for each version of your API. To do this, define multiple Swagger documents and for each one provide a predicate function that determines which API actions to include:

```csharp
services.AddSwaggerGen(c =>
{
    c.SwaggerDoc("v1",
        new Info
        {
            Title = "My API - V1",
            Version = "v1"
        },
        (api => api.RelativePath.Contains("v1"))
     );

     c.SwaggerDoc("v2",
        new Info
        {
            Title = "My API - V2",
            Version = "v2" 
        },
        (api => api.RelativePath.Contains("v2"))
     );
})
```

_The first argument to __SwaggerDoc__ should be a unique, URI-friendly name as it's used in the path for requesting the corresponding Swagger JSON. For example, with the default setup, the above documents will be available at "/swagger/v1/swagger.json" and "/swagger/v2/swagger.json"._

To expose the interactive docs for both versions, you also need to configure the __SwaggerUi__ middleware to list both versions:

```
app.UseSwaggerUi(c =>
{
    c.SwaggerEndpoint("/swagger/v2/swagger.json", "My API V2");
    c.SwaggerEndpoint("/swagger/v1/swagger.json", "My API V1");
})
```

### Omit Obsolete Operations and/or Schema Properties ###

The [Swagger spec](http://swagger.io/specification/) includes a "deprecated" flag for indicating that an operation is deprecated and should be refrained from use. The Swagger generator will automatically set this flag if the corresponding action is decorated with the _ObsoleteAttribute_. However, instead of setting a flag, you can configure the generator to ignore obsolete actions altogether:

```csharp
services.AddSwaggerGen(c =>
{
    ...

    c.IgnoreObsoleteActions();
};
```

A similar approach can also be used to omit obsolete properties from Schema's in the Swagger output. That is, you can decorate model properties with the _ObsoleteAttribute_ and configure Swashbuckle to omit those properties when generating a corresponding JSON Schema:

```csharp
services.AddSwaggerGen(c =>
{
    ...

    c.IgnoreObsoleteProperties();
};
```

### Omit Arbitrary Operations ###

To omit arbitrary operations from your API docs, you have a couple of options:

#### Decorate Individual Actions ####

Because Swashbuckle is built on top of the _ApiExplorer_ that ships with ASP.NET Core, you can decorate the corresponding action with an _ApiExplorerSettingsAttribute_ and set the _IgnoreApi_ flag:

```csharp
[HttpGet("{id}")]
[ApiExplorerSettings(IgnoreApi = true)]
public Product GetById(int id)
```

#### Apply a Predicate Function to Filter Actions ####

For a given _SwaggerDoc_, you can optionally provide a predicate function to filter actions. This can be useful if you want to omit actions by convention instead of individually. For example, the following configuration will omit all DELETE operations:


```csharp
c.SwaggerDoc("v1",
    new Info
    {
        Title = "My API - V1",
        Version = "v1"
    },
    (api => api.HttpMethod != "DELETE")
);
```

_The predicate function is optional - if you don't provide one then ALL actions will be included._

### Customize Operation Tags (e.g. for UI Grouping) ###

The [Swagger spec](http://swagger.io/specification/) allows one or more "tags" to be assigned to an operation. The Swagger generator will assign the controller name as the default tag. This is particularly interesting if you're using the _SwaggerUi_ middleware as it will group operations in the UI by that value.

You can override the default tag by providing a function that applies tags by convention. For example, the following configuration will tag, and therefore group operations in the UI, by HTTP method:

```csharp
services.AddSwaggerGen(c =>
{
    ...

    c.TagActionsBy(api => api.HttpMethod);
};
```

### Change Operations Sort Order ###

TODO:

### Customize Schema Id's ###

TODO:

### Customize Schema for Enum Types ###

TODO:

### Override Schema for Specific Types ###

TODO:

### Extend Generator with Operation, Schema & Document Filters ###

TODO:

### Add Security Definitions and Requirements ###

TODO: