---
title: Add correlation IDs to IoT messages w/distributed tracing (pre)
description: Learn how to use the distributed tracing ability to trace IoT messages throughout the Azure services used by your solution.
author: kgremban
manager: briz
ms.service: iot-hub
services: iot-hub
ms.topic: conceptual
ms.date: 01/26/2022
ms.author: kgremban
ms.custom: [amqp, mqtt, fasttrack-edit, iot]
---

# Trace Azure IoT device-to-cloud messages with distributed tracing (preview)

Microsoft Azure IoT Hub currently supports distributed tracing as a [preview feature](https://azure.microsoft.com/support/legal/preview-supplemental-terms/).

IoT Hub is one of the first Azure services to support distributed tracing. As more Azure services support distributed tracing, you'll be able trace IoT messages throughout the Azure services involved in your solution. For a background on distributed tracing, see [Distributed Tracing](../azure-monitor/app/distributed-tracing.md).

Enabling distributed tracing for IoT Hub gives you the ability to:

- Precisely monitor the flow of each message through IoT Hub using [trace context](https://github.com/w3c/trace-context). This trace context includes correlation IDs that allow you to correlate events from one component with events from another component. It can be applied for a subset or all IoT device messages using [device twin](iot-hub-devguide-device-twins.md).
- Automatically log the trace context to [Azure Monitor Logs](monitor-iot-hub.md).
- Measure and understand message flow and latency from devices to IoT Hub and routing endpoints.
- Start considering how you want to implement distributed tracing for the non-Azure services in your IoT solution.

In this article, you use the [Azure IoT device SDK for C](iot-hub-device-sdk-c-intro.md) with distributed tracing. Distributed tracing support is still in progress for the other SDKs.

## Prerequisites

- The preview of distributed tracing is currently only supported for IoT Hubs created in the following regions:

  - **North Europe**
  - **Southeast Asia**
  - **West US 2**

- This article assumes that you're familiar with sending telemetry messages to your IoT hub. Make sure you've completed the [Send telemetry C Quickstart](../iot-develop/quickstart-send-telemetry-iot-hub.md?pivots=programming-language-ansi-c).

- Register a device with your IoT hub (steps available in the Quickstart) and save the connection string.

- Install the latest version of [Git](https://git-scm.com/download/).

## Configure IoT Hub

In this section, you configure an IoT Hub to log distributed tracing attributes (correlation IDs and timestamps).

1. Navigate to your IoT hub in the [Azure portal](https://portal.azure.com/).

1. In the left pane for your IoT hub, scroll down to the **Monitoring** section and click **Diagnostics settings**.

1. Click **Add diagnostic setting**.

1. In the **Diagnostic setting name** field, enter a name for a new diagnostic setting. For example, **DistributedTracingSettings**.

   :::image type="content" source="media/iot-hub-distributed-tracing/diagnostic-setting-name.png" alt-text="Screenshot showing where to add a name for your diagnostic settings." lightbox="media/iot-hub-distributed-tracing/diagnostic-setting-name.png":::

1. Choose one or more of the following options under **Destination details** to determine where the logging will be sent:

    - **Archive to a storage account**: Configure a storage account to contain the logging information.
    - **Stream to an event hub**: Configure an event hub to contain the logging information.
    - **Send to Log Analytics**: Configure a log analytics workspace to contain the logging information.

1. In the **Logs** section, select the operations you want to log.

   Include **Distributed Tracing** and configure a **Retention** for how many days you want the logging retained. Log retention affects storage costs.

   :::image type="content" source="media/iot-hub-distributed-tracing/select-distributed-tracing.png" alt-text="Screenshot showing where the Distributed Tracing operation is for IoT Hub diagnostic settings.":::

1. Select **Save** for this new setting.

1. (Optional) To see the messages flow to different places, set up [routing rules to at least two different endpoints](iot-hub-devguide-messages-d2c.md).

Once the logging is turned on, IoT Hub records a log when a message containing valid trace properties is encountered in any of the following situations:

- The message arrives at IoT Hub's gateway.
- The message is processed by the IoT Hub.
- The message is routed to custom endpoints. Routing must be enabled.

To learn more about these logs and their schemas, see [Monitor IoT Hub](monitor-iot-hub.md) and [Distributed tracing in IoT Hub resource logs](monitor-iot-hub-reference.md#distributed-tracing-preview).

## Set up device

In this section, you prepare a development environment for use with the [Azure IoT C SDK](https://github.com/Azure/azure-iot-sdk-c). Then, you modify one of the samples to enable distributed tracing on your device's telemetry messages.

These instructions are for building the sample on Windows. For other environments, see [Compile the C SDK](https://github.com/Azure/azure-iot-sdk-c/blob/master/iothub_client/readme.md#compile) or [Prepackaged C SDK for Platform Specific Development](https://github.com/Azure/azure-iot-sdk-c/blob/master/iothub_client/readme.md#prepackaged-c-sdk-for-platform-specific-development).

### Clone the source code and initialize

1. Install ["Desktop development with C++" workload](/cpp/build/vscpp-step-0-installation?view=vs-2019&preserve-view=true) for Visual Studio 2022. Visual Studio 2019 is also supported.

1. Install [CMake](https://cmake.org/). Ensure it's in your `PATH` by typing `cmake -version` from a command prompt.

1. Open a command prompt or Git Bash shell. Run the following commands to clone the latest release of the [Azure IoT C SDK](https://github.com/Azure/azure-iot-sdk-c) GitHub repository:

    ```cmd
    git clone -b public-preview https://github.com/Azure/azure-iot-sdk-c.git
    cd azure-iot-sdk-c
    git submodule update --init
    ```

    Expect this operation to take several minutes to complete.

1. Run the following commands from the `azure-iot-sdk-c` directory to create a `cmake` subdirectory and navigate to the `cmake` folder. 

    ```cmd
    mkdir cmake
    cd cmake
    cmake ..
    ```

    If `cmake` can't find your C++ compiler, you might get build errors while running the above command. If that happens, try running this command in the [Visual Studio command prompt](/dotnet/framework/tools/developer-command-prompt-for-vs). 

    Once the build succeeds, the last few output lines will look similar to the following output:

    ```cmd
    $ cmake ..
    -- Building for: Visual Studio 15 2017
    -- Selecting Windows SDK version 10.0.16299.0 to target Windows 10.0.17134.
    -- The C compiler identification is MSVC 19.12.25835.0
    -- The CXX compiler identification is MSVC 19.12.25835.0

    ...

    -- Configuring done
    -- Generating done
    -- Build files have been written to: E:/IoT Testing/azure-iot-sdk-c/cmake
    ```

### Edit the send telemetry sample to enable distributed tracing

> [!div class="button"]
> <a href="https://github.com/Azure-Samples/azure-iot-distributed-tracing-sample/blob/master/iothub_ll_telemetry_sample-c/iothub_ll_telemetry_sample.c" target="_blank">Get the sample on GitHub</a>

1. Use an editor to open the `azure-iot-sdk-c/iothub_client/samples/iothub_ll_telemetry_sample/iothub_ll_telemetry_sample.c` source file.

1. Find the declaration of the `connectionString` constant:

   :::code language="c" source="~/samples-iot-distributed-tracing/iothub_ll_telemetry_sample-c/iothub_ll_telemetry_sample.c" range="56-60" highlight="2":::

   Replace the value of the `connectionString` constant with the device connection string you saved in the [register a device](../iot-develop/quickstart-send-telemetry-iot-hub.md?pivots=programming-language-ansi-c#register-a-device) section of the [Send telemetry C Quickstart](../iot-develop/quickstart-send-telemetry-iot-hub.md?pivots=programming-language-ansi-c).

1. Change the `MESSAGE_COUNT` define to `5000`:

   :::code language="c" source="~/samples-iot-distributed-tracing/iothub_ll_telemetry_sample-c/iothub_ll_telemetry_sample.c" range="56-60" highlight="3":::

1. Find the line of code that calls `IoTHubDeviceClient_LL_SetConnectionStatusCallback` to register a connection status callback function before the send message loop. Add code under that line as shown below to call `IoTHubDeviceClient_LL_EnablePolicyConfiguration` enabling distributed tracing for the device:

   :::code language="c" source="~/samples-iot-distributed-tracing/iothub_ll_telemetry_sample-c/iothub_ll_telemetry_sample.c" range="144-152" highlight="5":::

   The `IoTHubDeviceClient_LL_EnablePolicyConfiguration` function enables policies for specific IoTHub features that are configured via [device twins](./iot-hub-devguide-device-twins.md). Once `POLICY_CONFIGURATION_DISTRIBUTED_TRACING` is enabled with the line of code above, the tracing behavior of the device will reflect distributed tracing changes made on the device twin.

1. To keep the sample app running without using up all your quota, add a one-second delay at the end of the send message loop:

   :::code language="c" source="~/samples-iot-distributed-tracing/iothub_ll_telemetry_sample-c/iothub_ll_telemetry_sample.c" range="177-186" highlight="8":::

### Compile and run

1. Navigate to the *iothub_ll_telemetry_sample* project directory from the CMake directory (`azure-iot-sdk-c/cmake`) you created earlier, and compile the sample:

    ```cmd
    cd iothub_client/samples/iothub_ll_telemetry_sample
    cmake --build . --target iothub_ll_telemetry_sample --config Debug
    ```

1. Run the application. The device sends telemetry supporting distributed tracing.

    ```cmd
    Debug/iothub_ll_telemetry_sample.exe
    ```

1. Keep the app running. Optionally observe the message being sent to IoT Hub by looking at the console window.

<!-- For a client app that can receive sampling decisions from the cloud, check out [this sample](https://aka.ms/iottracingCsample).  -->

### Workaround for third-party clients

Implementing the distributed tracing feature without using the C SDK is more complex. Therefore, this approach isn't recommended.

First, you must implement all the IoT Hub protocol primitives in your messages by following the dev guide [Create and read IoT Hub messages](iot-hub-devguide-messages-construct.md). Then, edit the protocol properties in the MQTT/AMQP messages to add `tracestate` as **system property**. 

Specifically:

* For MQTT, add `%24.tracestate=timestamp%3d1539243209` to the message topic, where `1539243209` should be replaced with the creation time of the message in the unix timestamp format. As an example, refer to the implementation [in the C SDK](https://github.com/Azure/azure-iot-sdk-c/blob/6633c5b18710febf1af7713cf1a336fd38f623ed/iothub_client/src/iothubtransport_mqtt_common.c#L761)
* For AMQP, add `key("tracestate")` and `value("timestamp=1539243209")` as message annotation. For a reference implementation, see the [uamqp_messaging.c](https://github.com/Azure/azure-iot-sdk-c/blob/6633c5b18710febf1af7713cf1a336fd38f623ed/iothub_client/src/uamqp_messaging.c#L527) file.

To control the percentage of messages containing this property, implement logic to listen to cloud-initiated events such as twin updates.

## Update sampling options 

To change the percentage of messages to be traced from the cloud, you must update the device twin. Updates can be made, using the JSON editor in the Azure portal or the IoT Hub service SDK. The following subsections provide examples.

### Update using the portal

1. Navigate to your IoT hub in the [Azure portal](https://portal.azure.com/), then select **Devices** from the menu.

1. Choose your device.

1. Look for the gear icon under **Distributed Tracing (preview)**, then select the icon. A new blade opens. Select the **Enable** option. Choose a **Sampling rate** between 0% and 100%, then select **Save**.

   :::image type="content" source="media/iot-hub-distributed-tracing/enable-distributed-tracing.png" alt-text="Screenshot showing how to enable distributed tracing in the Azure portal." lightbox="media/iot-hub-distributed-tracing/enable-distributed-tracing.png":::

1. Wait a few seconds, and hit **Refresh**, then if successfully acknowledged by device, a sync icon with a checkmark appears.

1. Go back to the console window for the telemetry message app. You'll see messages being sent with `tracestate` in the application properties.

   :::image type="content" source="media/iot-hub-distributed-tracing/MicrosoftTeams-image.png" alt-text="Screenshot showing trace state messages." lightbox="media/iot-hub-distributed-tracing/MicrosoftTeams-image.png":::

1. (Optional) Change the sampling rate to a different value, and observe the change in frequency that messages include `tracestate` in the application properties.

### Update using Azure IoT Hub for VS Code

1. With VS Code installed, install the latest version of [Azure IoT Tools](https://marketplace.visualstudio.com/items?itemName=vsciot-vscode.azure-iot-tools) for VS Code.

1. Open VS Code and go to the **Explorer** tab and **Azure IoT Hub** section. Select the `...` next to **Azure IoT Hub** to see a submenu. Choose the **Select IoT Hub** option to retrieve your IoT hub from Azure. A popup window appears at the top of VS Code where you can select your subscription and IoT hub.

   See a demonstration on the [**vscode-azure-iot-toolkit**](https://github.com/Microsoft/vscode-azure-iot-toolkit/wiki/Select-IoT-Hub) GitHub page.

1. Expand your device under **Devices** and look for right-click **Distributed Tracing Setting (Preview)**. Select the option **Update Distributed Tracing Setting (Preview)**. A popup window will appear at the top where you can select **Enable**. You now see **Enable Distributed Tracing: Enabled** under **Desired** of your **Distributed Tracing Setting (Preview)**.

   :::image type="content" source="media/iot-hub-distributed-tracing/enable-distributed-tracing-vsc.png" alt-text="Screenshot showing how to enable distributed tracing in the Azure IoT Hub extension.":::

1. Next a popup will appear for **Sampling Rate**. Add **100**, then press ENTER. You now see **Sample rate: 100(%)** under the **Desired** section as well.

    ![Update sampling rate](./media/iot-hub-distributed-tracing/update-distributed-tracing-setting-3.png)

### Bulk update for multiple devices

To update the distributed tracing sampling configuration for multiple devices, use [automatic device configuration](./iot-hub-automatic-device-management.md). Make sure you follow this twin schema:

```json
{
    "properties": {
        "desired": {
            "azureiot*com^dtracing^1": {
                "sampling_mode": 1,
                "sampling_rate": 100
            }
        }
    }
}
```

| Element name | Required | Type | Description |
|-----------------|----------|---------|-----------------------------------------------------|
| `sampling_mode` | Yes | Integer | Two mode values are currently supported to turn sampling on and off. `1` is On and, `2` is Off. |
| `sampling_rate` | Yes | Integer | This value is a percentage. Only values from `0` to `100` (inclusive) are permitted.  |

## Query and visualize

To see all the traces logged by an IoT Hub, query the log store that you selected in diagnostic settings. This section walks through a couple different options.

### Query using Log Analytics

If you've set up [Log Analytics with resource logs](../azure-monitor/essentials/resource-logs.md#send-to-azure-storage), query by looking for logs in the `DistributedTracing` category. For example, this query shows all the traces logged:

```Kusto
// All distributed traces 
AzureDiagnostics 
| where Category == "DistributedTracing" 
| project TimeGenerated, Category, OperationName, Level, CorrelationId, DurationMs, properties_s 
| order by TimeGenerated asc  
```

Example logs as shown by Log Analytics:

| TimeGenerated | OperationName | Category | Level | CorrelationId | DurationMs | Properties |
|--------------------------|---------------|--------------------|---------------|---------------------------------------------------------|------------|------------------------------------------------------------------------------------------------------------------------------------------|
| 2018-02-22T03:28:28.633Z | DiagnosticIoTHubD2C | DistributedTracing | Informational | 00-8cd869a412459a25f5b4f31311223344-0144d2590aacd909-01 |  | {"deviceId":"AZ3166","messageSize":"96","callerLocalTimeUtc":"2018-02-22T03:27:28.633Z","calleeLocalTimeUtc":"2018-02-22T03:27:28.687Z"} |
| 2018-02-22T03:28:38.633Z | DiagnosticIoTHubIngress | DistributedTracing | Informational | 00-8cd869a412459a25f5b4f31311223344-349810a9bbd28730-01 | 20 | {"isRoutingEnabled":"false","parentSpanId":"0144d2590aacd909"} |
| 2018-02-22T03:28:48.633Z | DiagnosticIoTHubEgress | DistributedTracing | Informational | 00-8cd869a412459a25f5b4f31311223344-349810a9bbd28730-01 | 23 | {"endpointType":"EventHub","endpointName":"myEventHub", "parentSpanId":"0144d2590aacd909"} |

To understand the different types of logs, see [Azure IoT Hub distributed tracing logs](monitor-iot-hub-reference.md#distributed-tracing-preview).

## Understand Azure IoT distributed tracing

### Context

Many IoT solutions, including our own [reference architecture](/azure/architecture/reference-architectures/iot) (English only), generally follow a variant of the [microservice architecture](/azure/architecture/microservices/). As an IoT solution grows more complex, you end up using a dozen or more microservices. These microservices may or may not be from Azure. Pinpointing where IoT messages are dropping or slowing down can become challenging. For example, you have an IoT solution that uses 5 different Azure services and 1500 active devices. Each device sends 10 device-to-cloud messages/second (for a total of 15,000 messages/second), but you notice that your web app sees only 10,000 messages/second. Where is the issue? How do you find the culprit?

### Distributed tracing pattern in microservice architecture

To reconstruct the flow of an IoT message across different services, each service should propagate a *correlation ID* that uniquely identifies the message. Once collected by Azure Monitor in a centralized system, correlation IDs enable you to see message flow. This method is called the [distributed tracing pattern](/azure/architecture/microservices/logging-monitoring#distributed-tracing).

To support wider adoption for distributed tracing, Microsoft is contributing to [W3C standard proposal for distributed tracing](https://w3c.github.io/trace-context/).

### IoT Hub support

Once enabled, distributed tracing support for IoT Hub will follow this flow:

1. A message is generated on the IoT device.
1. The IoT device decides (with help from cloud) that this message should be assigned with a trace context.
1. The SDK adds a `tracestate` to the message property, containing the message creation timestamp.
1. The IoT device sends the message to IoT Hub.
1. The message arrives at IoT hub gateway.
1. IoT Hub looks for the `tracestate` in the message properties, and checks to see if it's in the correct format.
1. If so, IoT Hub generates a globally unique `trace-id` for the message, a `span-id` for the "hop", and logs them to [IoT Hub distributed tracing logs](monitor-iot-hub-reference.md#distributed-tracing-preview) under the operation `DiagnosticIoTHubD2C`.
1. Once the message processing is finished, IoT Hub generates another `span-id` and logs it along with the existing `trace-id` under the operation `DiagnosticIoTHubIngress`.
1. If routing is enabled for the message, IoT Hub writes it to the custom endpoint, and logs another `span-id` with the same `trace-id` under the category `DiagnosticIoTHubEgress`.
1. The steps above are repeated for each message generated.

## Public preview limits and considerations

- Proposal for W3C Trace Context standard is currently a working draft.
- Currently, the only development language supported by client SDK is C.
- Cloud-to-device twin capability isn't available for [IoT Hub basic tier](iot-hub-scaling.md#basic-and-standard-tiers). However, IoT Hub will still log to Azure Monitor if it sees a properly composed trace context header.
- To ensure efficient operation, IoT Hub will impose a throttle on the rate of logging that can occur as part of distributed tracing.

## Next steps

- To learn more about the general distributed tracing pattern in microservices, see [Microservice architecture pattern: distributed tracing](https://microservices.io/patterns/observability/distributed-tracing.html).
- To set up configuration to apply distributed tracing settings to a large number of devices, see [Configure and monitor IoT devices at scale](./iot-hub-automatic-device-management.md).
- To learn more about Azure Monitor, see [What is Azure Monitor?](../azure-monitor/overview.md).
- To learn more about using Azure Monitor with IoT Hub, see [Monitor IoT Hub](monitor-iot-hub.md)
