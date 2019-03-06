---
layout: post
title: A Little Bit DEF (Data Exchange Framework)
tags: [sitecore, DEF, data exchange framework, SQL]
---

It's time to back off [migration](/2019-01-22-sitecore-content-migration-part-1), and time to get DEF (to anyone who gets that, sorry). As are things with every job ever, requirements change, timelines are delayed, and opinions conflict. All of that being said, I'm waiting for a consensus on what information is needed to carry over, so migration is delayed. Instead we'll focus on requirements to read and process data from SQL which is needed when certain items are rendered. The site is currently doing this in the controller, but, in the vain of microservices, I've gotten a solid agreement that it's okay for some data to eventually be available. So, why not avoid the extra round trip to the db while serving up a page and schedule some collection of the data ahead of time? In comes Data Exchange Framework.

Sitecore has some good [documentation](https://doc.sitecore.com/developers/def/21/data-exchange-framework/en/index-en.html) out there on the basics of DEF, but it is basic. After digging through the code for the SQL Provider, I found there isn't an implementation for stored procedures, which I'm hoping to utilize in this process, and also means creating a class or two to get this done.

There's a few pieces involved in a pipeline step in DEF, a Converter, Plugins and a Processor. The converter takes a template's data and maps it to various plugins, which act a lot like DTOs. The processor then consumes the plugins and does some work. That means the starting point is a plugin which somehow lines up with a slice of a template.

Looking at what's needed to call a proc, we need to create a template that has a field for the proc name, and another for the parameters, the basic SQL info, and probably some more I won't think of until way too late. To deal with the SQL params, I'm thinking of using a Name Value List, instead of writing a custom field.

<p class="mb-0">There's already some plugins for endpoint settings and database connection endpoint settings, so all the procedure info will go into a new plugin.</p>

```cs
    using System.Collections;
    using System.Data;
    using System.Data.SqlClient;
    using Sitecore.DataExchange;

    public class SqlStoredProcedureSettings : IPlugin
    {
        public SqlStoredProcedureSettings()
        {
            // Defaulting to proc.
            CommandType = CommandType.StoredProcedure;
            SqlParameters = new List<SqlParameter>();
        }

        public CommandType CommandType { get; }

        public string StoredProcedureName { get; set; }

        public List<SqlParameter> SqlParameters { get; set; }
    }
```

Looking at what it takes to use a SqlParameter, I'm going to use the value field of the Name Value List to hold the value and the data type with some delimiter between.  The key will then be just the key.

<p class="mb-0">So, we've got a container to pass along to the processor, and we need to convert the template to the plugins. The converter will inherit the BasePipelineStepConverter class. With converters there's also a concept of supported templates, which can be defined in an attribute, or added to the pipeline through pipeline methods.</p>

```cs
    using System;
    using System.Collections.Generic;
    using System.Data;
    using System.Data.SqlClient;
    using System.Web;
    using Sitecore.DataExchange;
    using Sitecore.DataExchange.Converters.PipelineSteps;
    using Sitecore.DataExchange.Models;
    using Sitecore.DataExchange.Plugins;
    using Sitecore.DataExchange.Repositories;
    using Sitecore.Services.Core.Model;
    using Sitecore.Web;

    // GlassMapper Template
    [SupportedIds(Templates.ReadStoredProcedureDataPipelineStep.TemplateIdString)]
    public class ReadStoredProcedureDataPipelineStepConverter : BasePipelineStepConverter
    {
        // Fields in the template.
        public const string FieldNameEndpointFrom = "EndpointFrom";
        public const string FieldStoredProcedure = "StoredProcedure";
        public const string FieldSqlParameters = "SqlParameters";

        // Field in the DEF Sql Provided Stored Procedure Template
        public const string FieldStoredProcedureName = "StoredProcedureName";

        public ReadStoredProcedureDataPipelineStepConverter(IItemModelRepository repository) : base(repository)
        {
        }

        protected override void AddPlugins(ItemModel source, PipelineStep pipelineStep)
        {
            var endpointSettings = (IPlugin) GetEndpointSettings(source);
            if (endpointSettings != null)
                pipelineStep.AddPlugin(endpointSettings);

            var storedProcedureSettings = (IPlugin) GetStoredProcedureSettings(source);
            if (storedProcedureSettings != null)
                pipelineStep.AddPlugin(storedProcedureSettings);
        }

        // Setting were the data will be found.
        protected override EndpointSettings GetEndpointSettings(ItemModel source)
        {
            var endpointSettings = new EndpointSettings();
            var model = ConvertReferenceToModel<Endpoint>(source, FieldNameEndpointFrom);
            if (model != null)
                endpointSettings.EndpointFrom = model;
            return endpointSettings;
        }

        // Creating the params.
        protected virtual SqlStoredProcedureSettings GetStoredProcedureSettings(ItemModel source)
        {
            var sqlParameters = GetStringValue(source, FieldSqlParameters);
            var parametersCollection = WebUtil.ParseUrlParameters(sqlParameters);
            var parameters = new List<SqlParameter>();
            // Build the params
            foreach (var key in parametersCollection.AllKeys)
            {
                // Make sure there's a 
                var valueWithType = parametersCollection[key].Split(':');
                if (valueWithType.Length < 1) continue;

                var paramValue = valueWithType[0];
                if (string.IsNullOrWhiteSpace(paramValue)) continue;

                SqlDbType sqlType;
                if (valueWithType.Length < 2)
                {
                    sqlType = SqlDbType.VarChar;
                }
                else if (!Enum.TryParse(valueWithType[1], out sqlType))
                {
                    sqlType = SqlDbType.VarChar;
                }

                var parameter = new SqlParameter
                {
                    ParameterName = key,
                    Value = paramValue,
                    SqlDbType = sqlType
                };
                parameters.Add(parameter);
            }

            // Put it in plugin format.
            return new SqlStoredProcedureSettings()
            {
                StoredProcedureName =
                    GetStringValueFromReference(source, FieldStoredProcedure, FieldStoredProcedureName),
                ParametersDictionary = parametersDictionary
            };
        }
    }
```

<p class="mb-0">At this point we have template fields converted to plugins which can now be consumed by a processor.  The processor will define the plugins it expects, handle the real logic, and make the data available for later processing.</p>

```cs
    using System;
    using System.Data;
    using Sitecore.DataExchange.Attributes;
    using Sitecore.DataExchange.Contexts;
    using Sitecore.DataExchange.Extensions;
    using Sitecore.DataExchange.Models;
    using Sitecore.DataExchange.Plugins;
    using Sitecore.DataExchange.Processors.PipelineSteps;
    using Sitecore.DataExchange.Providers.Sql.Endpoints;
    using Sitecore.Services.Core.Diagnostics;
    using Sitecore.DataExchange.Providers.Sql.ReadData;

    [RequiredEndpointPlugins(typeof(DatabaseConnectionEndpointSettings))]
    [RequiredPipelineStepPlugins(typeof(EndpointSettings), typeof(SqlStoredProcedureSettings))]
    public class ReadStoredProcedureDataPipelineStepProcessor : BasePipelineStepProcessor
    {
        protected override void ProcessPipelineStep(PipelineStep pipelineStep = null,
            PipelineContext pipelineContext = null, ILogger logger = null)
        {
            // Verify Param Values
            if (pipelineStep == null)
                throw new ArgumentNullException(nameof(pipelineStep));
            if (pipelineContext == null)
                throw new ArgumentNullException(nameof(pipelineContext));

            var endpointSettings = pipelineStep.GetEndpointSettings();
            if (endpointSettings == null)
            {
                Log(logger.Error, pipelineContext,
                    "Pipeline step processing will abort because the pipeline step is missing a plugin.",
                    $"plugin: {typeof(EndpointSettings).FullName}");
            }
            else
            {
                // Get the End Point (Sql Db to interact with).
                var endpoint = GetEndpoint(endpointSettings);
                var plugin1 = endpoint.GetPlugin<DatabaseConnectionEndpointSettings>();
                if (string.IsNullOrWhiteSpace(plugin1.ConnectionString))
                {
                    Log(logger.Error, pipelineContext,
                        "No connection string is set on the endpoint.",
                        $"endpoint: {endpoint.Name}");
                    pipelineContext.CriticalError = true;
                }
                else
                {
                    // Get the proc and params.
                    var plugin2 = pipelineStep.GetPlugin<SqlStoredProcedureSettings>();
                    if (string.IsNullOrWhiteSpace(plugin2.StoredProcedureName))
                    {
                        Log(logger.Error, pipelineContext, "No Procedure has been set on the step",
                            $"pipelineStep: {(object) pipelineStep.Name}");
                        pipelineContext.CriticalError = true;
                    }
                    else
                    {
                        // Build the command.
                        var connection = GetConnection(plugin1, pipelineContext, logger);
                        var command = GetCommand(plugin1, pipelineContext, logger);
                        command.CommandType = plugin2.CommandType;
                        command.CommandText = plugin2.StoredProcedureName;
                        command.Connection = connection;
                        // Set the parameters.
                        SetParameterCollection(plugin2, command);
                        // Fire it.
                        ExecuteAction(connection, command, pipelineContext, logger);
                    }
                }
            }
        }

        protected virtual IDbConnection GetConnection(DatabaseConnectionEndpointSettings settings,
            PipelineContext pipelineContext, ILogger logger)
        {
            // New up a Sql Connection.
            return Activator.CreateInstance(settings.ConnectionType, settings.ConnectionString,
                settings.ConnectionParameters) as IDbConnection;
        }

        protected virtual IDbCommand GetCommand(DatabaseConnectionEndpointSettings settings,
            PipelineContext pipelineContext, ILogger logger)
        {
            // New up a command.
            return Activator.CreateInstance(settings.CommandType, settings.CommandParameters) as IDbCommand;
        }

        protected virtual void SetParameterCollection(SqlStoredProcedureSettings settings, IDbCommand command)
        {
            // Set the params to pass to the proc.
            foreach (var sqlParameter in settings.ParametersDictionary.Values)
            {
                command.Parameters.Add(sqlParameter);
            }
        }

        protected override Endpoint GetEndpoint(EndpointSettings endpointSettings)
        {
            // Return the End Point (Sql Db to interact with).
            return endpointSettings.EndpointFrom;
        }

        protected override void ExecuteAction(IDbConnection connection, IDbCommand command,
            PipelineContext pipelineContext, ILogger logger)
        {
            // Set the data returned from the proc onto the iterable data storage of the pipeline.
            pipelineContext.AddPlugin(
                new IterableDataSettings(new ReadDataEnumerator(command)));
        }
    }
```

With these pieces in place we just need to hook it all up in Sitecore, which will have to wait until next time... Fortunately, setting up DEF isn't too intense, so it shouldn't be too long before we can start testing this.

I'll be back with a follow up soon.  See you there.

-Ben