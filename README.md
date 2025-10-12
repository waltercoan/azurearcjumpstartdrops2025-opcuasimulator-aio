# OPC UA Simulator connected to Azure IoT Operations

## Overview

![Device](https://github.com/waltercoan/azurearcjumpstartdrops2025-opcuasimulator-aio/blob/main/imgs/dispositivo.jpg?raw=true "Device")

This Jumpstart Drops provides a tutorial construct a [.NET Aspire application](https://learn.microsoft.com/pt-br/dotnet/aspire/get-started/aspire-overview?wt.mc_id=AZ-MVP-5003638) written in C# [.net 9](https://dotnet.microsoft.com/pt-br/download/dotnet/9.0?wt.mc_id=AZ-MVP-5003638), running on a Raspberry Pi ARM device, that simulate a OPC UA Server and generate telemetry data from a Home Brew equipment, to monitor temperatures, and send to [Azure IoT Operations](https://learn.microsoft.com/en-us/azure/iot-operations/overview-iot-operations?wt.mc_id=AZ-MVP-5003638). Than we process the data locally using AIO Data Flows, and send to [Azure Data Explorer](https://learn.microsoft.com/pt-br/azure/data-explorer/?wt.mc_id=AZ-MVP-5003638) to allow monitoring.


## Prerequisites

1. Edge device server running Ubuntu 24.04 or latter
2. Raspberry Pi B v3 (or newer) running Debian
3. Azure IoT Operations install in the edge device

## Architecture

![Architecture](https://github.com/waltercoan/azurearcjumpstartdrops2025-opcuasimulator-aio/blob/main/imgs/arquitetura.png?raw=true "Architecture")

- The Raspberry Pi is running the .NET Aspire C# 9 application, which has an Blazor interface to configure the temperatures of the brewing, fermentation and bottling stages;
- The Blazor interface, send a request of any temperature change to a .NET Minimal API application, that sends over Inter Processes Communication to the OPC UA Server Application;
- The OPC UA Server application exposes the port 4840 for the Azure IoT Operations to connect and collect the temperature telemetry values;
- Than Azure IoT Operations uses a Assets Endpoint and a Asset to connect to the OPC UA Server, collect the telemetry and send to a Topic into the MQTT broker into Azure IoT Operations edge device;
- A Azure IoT Operations Data flow, collect the telemetry data, change the schema format of the data, to finally send to Azure Data Explorer.

## OPC UA Server

The OPC UA Server was develop

The C# .NET 9 application that simulates the OPC UA Server was built by modeling the device with a file called ModelDesign.xml that describes the ontology of the industrial equipment and its information such as the three temperatures: Brewing, Fermentation and Bottling.
```xml
<?xml version="1.0" encoding="utf-8" ?>
<opc:ModelDesign
xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
xmlns:xsd="http://www.w3.org/2001/XMLSchema"
xmlns:opc="http://opcfoundation.org/UA/ModelDesign.xsd"
xmlns:ua="http://opcfoundation.org/UA/"
xmlns:uax="http://opcfoundation.org/UA/2008/02/Types.xsd"
xmlns="http://opcfoundation.org/BrewPlant"
TargetNamespace="http://opcfoundation.org/BrewPlant"
>

  <opc:Namespaces>
    <opc:Namespace Name="OpcUa" Prefix="Opc.Ua" XmlNamespace="http://opcfoundation.org/UA/2008/02/Types.xsd">http://opcfoundation.org/UA/</opc:Namespace>
    <opc:Namespace Name="BrewPlant" Prefix="BrewPlant">http://opcfoundation.org/BrewPlant</opc:Namespace>
  </opc:Namespaces>

    <opc:ObjectType SymbolicName="GenericSensorType" BaseType="ua:BaseObjectType">
    <opc:Description>A generic sensor that read a process value.</opc:Description>
    <opc:Children>
      <opc:Variable SymbolicName="Output" DataType="ua:Double" ValueRank="Scalar" TypeDefinition="ua:AnalogItemType" />
      <opc:Property SymbolicName="Units" DataType="ua:String" ValueRank="Scalar"  AccessLevel="ReadWrite" />
    </opc:Children>
    </opc:ObjectType>

  <opc:ObjectType SymbolicName="TemperatureIndicatorType" BaseType="GenericSensorType">
    <opc:Description>A sensor that reports the temperature.</opc:Description>
  </opc:ObjectType>
  

  	<opc:ObjectType SymbolicName="ManufacturingFacilityType" BaseType="ua:FolderType">
		<opc:Children>
			<opc:Object SymbolicName="Temperature1Indicator" TypeDefinition="TemperatureIndicatorType" SupportsEvents="true">
				<opc:BrowseName>TP001</opc:BrowseName>
			</opc:Object>
			<opc:Object SymbolicName="Temperature2Indicator" TypeDefinition="TemperatureIndicatorType" SupportsEvents="true">
				<opc:BrowseName>TP002</opc:BrowseName>
			</opc:Object>
			<opc:Object SymbolicName="Temperature3Indicator" TypeDefinition="TemperatureIndicatorType" SupportsEvents="true">
				<opc:BrowseName>TP003</opc:BrowseName>
			</opc:Object>
		</opc:Children>
		<opc:References>
			<opc:Reference>
				<opc:ReferenceType>ua:HasNotifier</opc:ReferenceType>
				<opc:TargetId>ManufacturingFacilityType_Temperature1Indicator</opc:TargetId>
			</opc:Reference>
		</opc:References>
	</opc:ObjectType>

    
	<opc:ObjectType SymbolicName="BrewPlantType" BaseType="ua:BaseObjectType" SupportsEvents="true">
		<opc:Description>A production batch plant.</opc:Description>
		<opc:Children>

			<opc:Object SymbolicName="ManufacturingFacility1" TypeDefinition="ManufacturingFacilityType" SupportsEvents="true">
				<opc:BrowseName>ManufacturingFacilityX001</opc:BrowseName>
				<opc:Children>
					<opc:Object SymbolicName="Temperature1Indicator">
						<opc:Children>
							<opc:Variable SymbolicName="Output" />
						</opc:Children>
                        <opc:Children>
							<opc:Variable SymbolicName="Units" />
						</opc:Children>
					</opc:Object>
					<opc:Object SymbolicName="Temperature2Indicator">
						<opc:Children>
							<opc:Variable SymbolicName="Output" />
						</opc:Children>
                        <opc:Children>
							<opc:Variable SymbolicName="Units" />
						</opc:Children>
					</opc:Object>
					<opc:Object SymbolicName="Temperature3Indicator">
						<opc:Children>
							<opc:Variable SymbolicName="Output" />
						</opc:Children>
                        <opc:Children>
							<opc:Variable SymbolicName="Units" />
						</opc:Children>
					</opc:Object>
				</opc:Children>
			</opc:Object>

			<opc:Method SymbolicName="StartProcess" ModellingRule="Mandatory"></opc:Method>
            <opc:Method SymbolicName="StopProcess" ModellingRule="Mandatory"></opc:Method>

		
		</opc:Children>
		

	</opc:ObjectType>

	<opc:Object SymbolicName="BrewPlant1" TypeDefinition="BrewPlantType" SupportsEvents="true">
		<opc:BrowseName>Brew Plant #1</opc:BrowseName>
		<opc:Children>
			<opc:Object SymbolicName="ManufacturingFacility1" TypeDefinition="ManufacturingFacilityType" SupportsEvents="true">
				<opc:DisplayName>ManufacturingFacilityX001</opc:DisplayName>
			</opc:Object>
		</opc:Children>


		<opc:References>
			<opc:Reference IsInverse="true">
				<opc:ReferenceType>ua:Organizes</opc:ReferenceType>
				<opc:TargetId>ua:ObjectsFolder</opc:TargetId>
			</opc:Reference>
		</opc:References>

	</opc:Object>

</opc:ModelDesign>
```

This Model was compiled using [OPC Foundation UA Model Compiler available in this repository](https://github.com/OPCFoundation/UA-ModelCompiler) to then generate a set of classes that were incorporated into the .NET application. In the Program.cs class, in addition to starting the OPC UA Server, an Inter Processes Communication is also started using the NamedPipeServerStream class, to receive updates from the .NET Minimal API application.

```cs
    class PipeServer
        {
            public async Task Start()
            {
                using var pipeServer = new NamedPipeServerStream("brewdata_pipe", PipeDirection.In);
                Console.WriteLine("Pipe server aguardando conexão...");
                await pipeServer.WaitForConnectionAsync();

                using var reader = new StreamReader(pipeServer, Encoding.UTF8);
                while (pipeServer.IsConnected)
                {
                    var line = await reader.ReadLineAsync();
                    if (line != null)
                    {
                        Console.WriteLine($"Recebido via pipe: {line}");
                        var data = System.Text.Json.JsonSerializer.Deserialize<BrewData>(line);
                        if (data != null)
                        {
                            BrewDataSingleton.Instance.UpdateBrewData(data);
                            Console.WriteLine($"BrewData atualizado: Temp={data.BrewingTemperatureC}, Pressure={data.FermentationTemperatureC}, FlowRate={data.BottlingTemperatureC}");
                        }
                    }
                }
            }
        }
```

The BrewNodeManager class that publishes the updated Brewing, Fermentation and Bottling temperature values ​​within the OPC UA Server objects.

```cs
using System;
using System.Reflection;
using BrewPlant;
using Opc.Ua;
using Opc.Ua.Server;

namespace opcuasimulator.OPCUAServer.Server;

public class BrewNodeManager : CustomNodeManager2
{
    
    private BrewPlantServerConfiguration _configuration;
    private static BrewPlantState brewPlant1;
    private Timer simulationTimer;
    private BrewData lastBrewData = new BrewData(0.0f, 0.0f, 0.0f);

    public BrewNodeManager(IServerInternal server, ApplicationConfiguration configuration)
        : base(server, configuration)
    {
        SystemContext.NodeIdFactory = this;

        string[] namespaceUrls = new string[]
        {
            BrewPlant.Namespaces.BrewPlant,
            BrewPlant.Namespaces.BrewPlant + "/Instance"
        };
        SetNamespaces(namespaceUrls);

        _configuration = configuration.ParseExtension<BrewPlantServerConfiguration>();
        if (_configuration == null)
        {
            _configuration = new BrewPlantServerConfiguration();
        }
    }
    protected override NodeStateCollection LoadPredefinedNodes(ISystemContext context)
    {
        Console.WriteLine("Loading predefined nodes...");
        NodeStateCollection predefinedNodes = new NodeStateCollection();
        string path = Path.Combine(Path.GetDirectoryName(Assembly.GetExecutingAssembly().Location), @"Models/BrewPlant.PredefinedNodes.uanodes");
        Console.WriteLine($"Loaded predefined nodes from: {path}");
        predefinedNodes.LoadFromBinaryResource(context,
        path,
        typeof(BrewNodeManager).GetTypeInfo().Assembly,
        true);
        Console.WriteLine($"Loaded predefined nodes from: {path}");
        return predefinedNodes;
    }

    public override void CreateAddressSpace(IDictionary<NodeId, IList<IReference>> externalReferences)
    {
        Console.WriteLine("Creating address space...");
        lock (Lock)
        {
            LoadPredefinedNodes(SystemContext, externalReferences);

            BaseObjectState passiveNode =
                (BaseObjectState)FindPredefinedNode(new NodeId(BrewPlant.Objects.BrewPlant1,
                    NamespaceIndexes[0]), typeof(BaseObjectState));
            brewPlant1 = new BrewPlantState(null);
            brewPlant1.Create(SystemContext, passiveNode);

            AddPredefinedNode(SystemContext, brewPlant1);

            brewPlant1.StartProcess.OnCallMethod = new GenericMethodCalledEventHandler(OnStartProcess);
            brewPlant1.StopProcess.OnCallMethod = new GenericMethodCalledEventHandler(OnStopProcess);

            simulationTimer = new System.Threading.Timer(UpdateSimulation, null, 10000, 10000);

        }
    }

    private void UpdateSimulation(object? state)
    {
        var data = BrewDataSingleton.Instance.CurrentBrewData;
        if(data.Equals(lastBrewData))
        {
            // No changes
            return;
        }
        var value = data.BrewingTemperatureC;
        Console.WriteLine($"Updating Temperature1 Indicator to {value}");
        brewPlant1.ManufacturingFacility1.Temperature1Indicator.Output.Value = value;
        brewPlant1.ManufacturingFacility1.Temperature1Indicator.Output.Timestamp = DateTime.UtcNow;
        brewPlant1.ManufacturingFacility1.Temperature1Indicator.Output.StatusCode = StatusCodes.Good;
        brewPlant1.ManufacturingFacility1.Temperature1Indicator.Output.ClearChangeMasks(SystemContext, false);

        value = data.FermentationTemperatureC;
        Console.WriteLine($"Updating Temperature2 Indicator to {value}");
        brewPlant1.ManufacturingFacility1.Temperature2Indicator.Output.Value = value;
        brewPlant1.ManufacturingFacility1.Temperature2Indicator.Output.Timestamp = DateTime.UtcNow;
        brewPlant1.ManufacturingFacility1.Temperature2Indicator.Output.StatusCode = StatusCodes.Good;
        brewPlant1.ManufacturingFacility1.Temperature2Indicator.Output.ClearChangeMasks(SystemContext, false);

        value = data.BottlingTemperatureC;
        Console.WriteLine($"Updating Temperature3 Indicator to {value}");
        brewPlant1.ManufacturingFacility1.Temperature3Indicator.Output.Value = value;
        brewPlant1.ManufacturingFacility1.Temperature3Indicator.Output.Timestamp = DateTime.UtcNow;
        brewPlant1.ManufacturingFacility1.Temperature3Indicator.Output.StatusCode = StatusCodes.Good;
        brewPlant1.ManufacturingFacility1.Temperature3Indicator.Output.ClearChangeMasks(SystemContext, false);
    }

    private ServiceResult OnStartProcess(
        ISystemContext context,
        MethodState method,
        IList<object> inputArguments,
        IList<object> outputArguments)
    {
        Console.WriteLine("Starting brewing process...");
        return ServiceResult.Good;
    }

    private ServiceResult OnStopProcess(
        ISystemContext context,
        MethodState method,
        IList<object> inputArguments,
        IList<object> outputArguments)
    {
        Console.WriteLine("Stopping brewing process...");
        return ServiceResult.Good;
    }
}
```

When the application is running, we can use an OPC UA client to connect to the server and navigate through the hierarchical structure of the objects defined in the models and discover the unique address of each temperature sensor.

![OPC UA Objects Tree](https://github.com/waltercoan/azurearcjumpstartdrops2025-opcuasimulator-aio/blob/main/imgs/opcuaobjects.png?raw=true "OPC UA Objects Tree")


## OPC UA Server Human Machine Interface in Blazor

![OPC UA Server Human Machine Interface in Blazor](https://github.com/waltercoan/azurearcjumpstartdrops2025-opcuasimulator-aio/blob/main/imgs/blazorapp.png?raw=true "OPC UA Server Human Machine Interface in Blazor")


## Installing Azure IoT Operations
In this article [Installing Azure IoT Operations](https://learn.microsoft.com/en-us/azure/iot-operations/deploy-iot-ops/howto-deploy-iot-operations?wt.mc_id=AZ-MVP-5003638) describe one of the possible processes for installing Azure IoT Operations on an edge device running a Kubernetes cluster.

## Configuring Azure IoT Operations

The first step is to configure Azure IoT Operations to connect to the UPC UA application on port 4840.

![Config Asset endpoint](https://github.com/waltercoan/azurearcjumpstartdrops2025-opcuasimulator-aio/blob/main/imgs/aio01.png?raw=true "Config Asset endpoint")

For the connection to occur, a mutual certificate exchange must be configured between the OPC UA server and Azure IoT Operations. These certificates must be obtained from the OPC UA server and then uploaded via the AIO portal to be stored on the Kubernetes cluster device and synchronized to Azure Key Vault.

![Certificate](https://github.com/waltercoan/azurearcjumpstartdrops2025-opcuasimulator-aio/blob/main/imgs/aio02.png?raw=true "Certificate")

With the endpoint configured, we can create an AIO Asset, which represents the device, linking it to the AIO Asset Endpoint.

![AIO Asset](https://github.com/waltercoan/azurearcjumpstartdrops2025-opcuasimulator-aio/blob/main/imgs/aio03.png?raw=true "AIO Asset")

Still in the AIO Asset configuration, on the Tags tab, we need to register a tag for each object in the OPC UA Server that represents an object with the device's temperature value. To do this, we must enter the unique identifier of the object created by the OPC UA Server in the Node Id field.

![AIO Asset Tags](https://github.com/waltercoan/azurearcjumpstartdrops2025-opcuasimulator-aio/blob/main/imgs/aio04.png?raw=true "AIO Asset Tags")

Once the AIO Asset is configured, the AIO will connect to the OPC UA server, and any changes to temperature values ​​made in the Blazor application interface will be published to the AIO's MQTT broker. In this image, we're using the MQTTx Client to subscribe to all topics and receive device temperature telemetry update messages.

![MQTT Topic](https://github.com/waltercoan/azurearcjumpstartdrops2025-opcuasimulator-aio/blob/main/imgs/aio05.png?raw=true "MQTT Topic")

## Configuring Azure IoT Operations Data Flow + Azure Data Explorer

The next step is to configure AIO to send the collected data to Azure Data Explorer. To do this, we need to create an Azure Data Explorer instance and use a User Managed Identity to authorize AIO to send the data stream to Azure Data Explorer. Once this is done, we can return to the AIO operator portal and register a Data Flow Endpoint pointing to the instance in Azure.

![Data Flow Endpoint](https://github.com/waltercoan/azurearcjumpstartdrops2025-opcuasimulator-aio/blob/main/imgs/aio06.png?raw=true "Data Flow Endpoint")

Next, in the AIO operator portal, we must create a new Data Flow to get the Asset telemetry and forward it to Azure Data Explorer.

![Data Flow](https://github.com/waltercoan/azurearcjumpstartdrops2025-opcuasimulator-aio/blob/main/imgs/aio08.png?raw=true "Data Flow")

In the AIO Data Flow transformation step, we use the rename step to map the attributes of the message received from the OPC UA Server in the MQTT broker to the format expected by the Azure Data Explorer table.

![Data Flow - Transform](https://github.com/waltercoan/azurearcjumpstartdrops2025-opcuasimulator-aio/blob/main/imgs/aio09.png?raw=true "Data Flow - Transform")

In the destination step, we point to the Azure Data Explorer Data Flow Endpoint, provide the table name, and a schema file that describes the format of the table's attributes in Azure Data Explorer.

![Data Flow - Destination](https://github.com/waltercoan/azurearcjumpstartdrops2025-opcuasimulator-aio/blob/main/imgs/aio09.png?raw=true "Data Flow - Destination")

Finally, we demonstrate the data being stored in Azure Data Explorer, allowing it to be queried with a query in Kusto Query Language.

![Azure Data Explorer](https://github.com/waltercoan/azurearcjumpstartdrops2025-opcuasimulator-aio/blob/main/imgs/azuredataexplorer.png?raw=true "Azure Data Explorer")

