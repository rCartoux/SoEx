# Hello Grpc


Create Project
```
dotnet new grpc -n DaprDemo.D002.HelloGrpc
cd DaprDemo.D002.HelloGrpc
```

Add Dapr library

```
dotnet add package Dapr.AspNetCore 
```

Update startup.cs

```csharp
public void ConfigureServices(IServiceCollection services)
{
    ...
    services.AddDaprClient();
}
```

Modify GreeterService.cs 

Add using statements
```csharp
using Dapr.AppCallback.Autogen.Grpc.v1;
using Dapr.Client.Autogen.Grpc.v1;
using Google.Protobuf.WellKnownTypes;
```

Changing the base class as below

```csharp
public class GreeterService :  AppCallback.AppCallbackBase
```

remove override from  Say Hello Method
```csharp
public Task<HelloReply> SayHello(HelloRequest request, ServerCallContext context)
{
    return Task.FromResult(new HelloReply
    {
        Message = "Hello " + request.Name
    });
}
```
implement oninvoke
```csharp
public override async Task<InvokeResponse> OnInvoke(InvokeRequest request, ServerCallContext context)
{
    var response = new InvokeResponse();
    switch (request.Method)
    {
        case "sayhello":                
            var input = request.Data.Unpack<HelloRequest>();
            var output = await SayHello(input, context);
            response.Data = Any.Pack(output);
            break;       
        default:
            break;
    }
    return response;
}
```

Implement ListTopicSubscriptions to keep dapr happy
```csharp
public override Task<ListTopicSubscriptionsResponse> ListTopicSubscriptions(Empty request, ServerCallContext context)
{            
    return Task.FromResult(new ListTopicSubscriptionsResponse());
}
```



As we are using grpc access from a browser won't work, we will cheat and add a client call to this project
Create a file LocalDaprClient
```csharp
using System;
using System.Threading;
using System.Threading.Tasks;
using Dapr.Client;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;

namespace DaprDemo.D002.HelloGrpc
{
    public class LocalDaprClient : IHostedService
    {       
        public LocalDaprClient(ILogger<LocalDaprClient> logger, IHostApplicationLifetime hostApplicationLifetime)
        {            
            hostApplicationLifetime.ApplicationStarted.Register( async () => {
            using var client = new DaprClientBuilder().Build();
            var response = await client.InvokeMethodGrpcAsync<HelloRequest, HelloReply>("DaprDemo-D002-HelloGrpc","sayhello",new HelloRequest(){ Name = "[Your name here]"});
            logger.LogInformation($"Grpc Response: {response.Message}");
            });
        }

        public Task StartAsync(CancellationToken cancellationToken)
        {
            return Task.CompletedTask;
        }

        public Task StopAsync(CancellationToken cancellationToken)
        {
            return Task.CompletedTask;        
        }
    }
}
```

in startup.cs add
```csharp
public void ConfigureServices(IServiceCollection services)
{
    ...
    services.AddHostedService<LocalDaprClient>();
}
```

run
```
dapr run --app-id DaprDemo-D002-HelloGrpc --app-port 5000 --app-protocol grpc -- dotnet run
```

Observe output in the logs

```
== APP == info: DaprDemo.D002.HelloGrpc.LocalDaprClient[0]

== APP ==       Grpc Response: Hello [Your name here]
```