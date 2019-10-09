# simple-cqrs-pattern
A simple nuget package for implementing CQRS request dispatching in your asynchronous .Net Core workflow.

## Install the nuget package in any project using this pattern
```
PM> Install-Package Simple.CQRS -Version 1.0.0
```

## Simply add this to your .Net core Startup.cs in configure services

```C#
services.AddCommandQueryHandlers(typeof(ICommandHandler<>));
services.AddCommandQueryHandlers(typeof(IQueryHandler<,>));

services.AddScoped<ICommandDispatcher, CommandDispatcher>();
services.AddScoped<IQueryDispatcher, QueryDispatcher>();

MapCQRSDependencies(services);
```

## And then register your queries or commands (you can use another method or class as the list grows):

```C#
private void MapCQRSDependencies(IServiceCollection services)
{
    services.AddScoped<IQueryHandler<GetFooQuery, GetFooResult>, GetFooAsyncHandler>();
    services.AddScoped<ICommandHandler<BarCommand>, BarAsyncHandler>();
}
```

## Make sure your handler classes inherit from the Simple.CQRS Interfaces:

```C#
public class GetFooAsyncHandler : IQueryHandler<GetFooQuery, GetFooResult>
{
    public async GetFooResult HandleAsync(GetFooQuery query)
    {
        return new GetFooResult { Foo = query.Foo, Bar = query.Bar };
    }
}
```

```C#
public class GetBarAsyncHandler : ICommandHandler<BarCommand>
{
    public async Task HandleAsync(BarCommand command)
    {
        // do stuff with command
        // if fails throw an exception, this will terminate the request handling pipeline
        // maybe add a retry policy
        // you can also build your own exception handling into your pipeline
    }
}
```

## Finally, dispatch requests using the dispatcher in your controller:

```C#
public class ExampleController : ControllerBase
{
        private readonly IQueryDispatcher _queryDispatcher;
        private readonly ICommandDispatcher _commandDispatcher;


        public AccountsController(IQueryDispatcher queryDispatcher, 
        ICommandDispatcher commandDispatcher)
        {
            _queryDispatcher = queryDispatcher;
            _commandDispatcher = commandDispatcher;
        }

        /// <summary>
        /// Bar endpoint
        /// </summary>
        /// <returns></returns>
	[HttpPost("{userId}")]
        [Authorize]
        public async Task<ActionResult> GetBarAsync([FromQuery, BindRequired] long userId)
        {
            await _commandDispatcher.HandleAsync(new BarCommand { UserId = userId });

            return Ok();
        }
		
	/// <summary>
        /// Foo endpoint
        /// </summary>
        /// <returns></returns>
	[HttpGet]
        [Authorize]
        public async Task<ActionResult> GetBarAsync(Request request)
        {
            var result = await _queryDispatcher.HandleAsync<GetFooQuery, GetFooResult>(new GetFooQuery { Foo = request.Foo, Bar =  request.Bar });

            return Ok(result);
        }
}
```
