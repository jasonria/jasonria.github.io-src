---
title: How to log incoming and outgoing messages for ASP.NET Core Bot
date: 2020-07-03 15:11:28
tags: ['Azure Bot Framework']
---
If we want to log all the incoming and outgoing messages for an Azure bot application, we can achieve this goal by implementing a Bot middleware.
<!-- more -->

1. Take [EchoBot](https://github.com/microsoft/BotBuilder-Samples/tree/master/samples/csharp_dotnetcore/02.echo-bot) as an example. After downloading the source code and opening it in Visual Studio, let's create a folder called **Middlewares**, and then create a new .cs file called **MessageInspectorMiddleware.cs**.
![](/images/How_to_log_incoming_and_outgoing_messages_for_aspnetcore_bot/img001.png)


2. Put the following source code in **MessageInspectorMiddleware.cs**. In **turnContext.OnSendActivities** event handler, we can retrieve the outgoing message which is sent from the bot to the user. In **OnTurnAsync** event but out of the **turnContext.OnSendActivities** event handler, we are able to retrieve the incoming message which is sent from user to bot.
``` CS
using Microsoft.Bot.Builder;
using Microsoft.Bot.Builder.Integration.AspNet.Core;
using Microsoft.Bot.Connector.Authentication;
using Microsoft.Bot.Schema;
using Microsoft.Extensions.Logging;
using Newtonsoft.Json;
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading;
using System.Threading.Tasks;

namespace Microsoft.BotBuilderSamples
{
    public class MessageInspectorMiddleware : IMiddleware
    {
        private readonly ILogger<MessageInspectorMiddleware> _logger;
        public MessageInspectorMiddleware(ILogger<MessageInspectorMiddleware> logger = null) {
            _logger = logger;
        }
        public async Task OnTurnAsync(ITurnContext turnContext, NextDelegate next, CancellationToken cancellationToken = default)
        {
            turnContext.OnSendActivities(async (ctx, activities, nextSend) => 
            {
                foreach(Activity activity in activities)
                {
                    if(activity.Type == ActivityTypes.Message)
                    {
                        // Log the outgoing messages which are sent from bot to user.
                        _logger.LogInformation("===============> Outgoing message: " + JsonConvert.SerializeObject(activity));
                    }
                }
                
                return await nextSend().ConfigureAwait(false);
            });
			
            if(turnContext.Activity.Type == ActivityTypes.Message && !string.IsNullOrEmpty(turnContext.Activity.Text))
            {
                // Log the incoming messages which are sent from user to bot.
                _logger.LogInformation("===============> Incoming message: " + JsonConvert.SerializeObject(turnContext.Activity));
            }
			
            await next(cancellationToken).ConfigureAwait(false);
        }
    }
}
```


3. Open **AdapterWithErrorHandler.cs**. Add one more parameter called **MessageInspectorMiddleware messageInspectorMiddleware** for the method AdapterWithErrorHandler(). And then add **Use(messageInspectorMiddleware);** at Line 15.
``` CS
using Microsoft.AspNetCore.Http;
using Microsoft.Bot.Builder.Integration.AspNet.Core;
using Microsoft.Bot.Builder.TraceExtensions;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.Logging;

namespace Microsoft.BotBuilderSamples
{
    public class AdapterWithErrorHandler : BotFrameworkHttpAdapter
    {
        public AdapterWithErrorHandler(IConfiguration configuration, ILogger<BotFrameworkHttpAdapter> logger, MessageInspectorMiddleware messageInspectorMiddleware)
            : base(configuration, logger)
        {
            // Use MessageInspector middleware
            Use(messageInspectorMiddleware);
            
            OnTurnError = async (turnContext, exception) =>
            {
                logger.LogError(exception, $"[OnTurnError] unhandled error : {exception.Message}");
                await turnContext.SendActivityAsync("The bot encountered an error or bug.");
                await turnContext.SendActivityAsync("To continue to run this bot, please fix the bot source code.");
                await turnContext.TraceActivityAsync("OnTurnError Trace", exception.Message, "https://www.botframework.com/schemas/error", "TurnError");
            };
        }
    }
}
```


4. Open **Startup.cs**. Add **services.AddSingleton\<MessageInspectorMiddleware\>();** at Line 25.
``` CS
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Hosting;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Bot.Builder;
using Microsoft.Bot.Builder.Integration.AspNet.Core;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.BotBuilderSamples.Bots;

namespace Microsoft.BotBuilderSamples
{
    public class Startup
    {
        public Startup(IConfiguration configuration)
        {
            Configuration = configuration;
        }
        public IConfiguration Configuration { get; }

        public void ConfigureServices(IServiceCollection services)
        {
            services.AddMvc().SetCompatibilityVersion(CompatibilityVersion.Version_2_1);
			
            // Create MessageInspectorMiddleware
            services.AddSingleton<MessageInspectorMiddleware>();
			
            services.AddSingleton<IBotFrameworkHttpAdapter, AdapterWithErrorHandler>();
			
            services.AddTransient<IBot, EchoBot>();
        }

        public void Configure(IApplicationBuilder app, IHostingEnvironment env)
        {
            if (env.IsDevelopment())
            {
                app.UseDeveloperExceptionPage();
            }
            else
            {
                app.UseHsts();
            }
            app.UseDefaultFiles();
            app.UseStaticFiles();
            app.UseWebSockets();
            //app.UseHttpsRedirection();
            app.UseMvc();
        }
    }
}

```


5. Then you can run the Echobot inside the Visual Studio. Suppose the user sent **Hello World** to the bot. In the Console window, you can see the incoming message **Hello World** as well as the outgoing message **Echo: Hello world** which is sent by the bot to the user in the console output. 
   ![](/images/How_to_log_incoming_and_outgoing_messages_for_aspnetcore_bot/img002.png)


6. Now we can output all the incoming and outgoing bot messages, the left work is to choose a persistent approach to store the messages. For example you can use SeriLog to log these messages in a txt file on the disk.