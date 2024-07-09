# Zigbee Gateway Example

This test code shows how to configure Zigbee Coordinator and use it as a Home Automation gateway for receiving data from a client.

## How does Zigbee work?

Zigbee introduces several concepts:
- Node
- Endpoint
- Cluster
- Attribute

### Node
A node is a single ESP32-H2 based product. It represents a **network node** in the Zigbee network. A single node can expose multiple endpoints.

### Endpoint (~ group of objects)

Within each node are endpoints. Endpoints, **identified by a number** between 1 and 240, define each application running in a ZigBee node. Endpoints serve three purposes in ZigBee:
- Endpoints allow for different application profiles to exist within each node.
- Endpoints allow for separate control points to exist within each node.
- Endpoints allow for **separate devices to exist within each node**.

### Cluster (~ object)

Clusters are application objects. Endpoints are addressing concepts, the cluster **defines application meaning**.
- An endpoint can have multiple clusters.
- Clusters, in addition to the identifier, have direction. In the SimpleDescriptor which describes an endpoint, a cluster is listed as either input or output.
- Clusters **contain both code (commands) and data (attributes)**. Commands cause action. Attributes keep track of the current state of that cluster.

If you are using the same objects (e.g., 2 LEDs), you should use 2 different endpoints because a cluster is just an identifier for an object. If you use the same cluster, you will have the same identifier twice.

ID | Name | Description
---|------|------------
0x0006 | On/Off | Attributes and commands for switching devices between ‘On’ and ‘Off’ states.
0x0007 | On/Off | Switch Configuration Attributes and commands for configuring on/off switching devices
0x0008 | Level | Level Control for Lighting Attributes and commands for controlling a characteristic of devices that can be set to a level between fully ‘On’ and fully ‘Off’.
0x001C | Pulse | Width Modulation Level also with frequency control

Using 1 endpoint (won't work):
- Endpoint1
  - LED1 (0x0006)
  - LED2 (0x0006)
  
Using 2 endpoints (should work :) ):
- Endpoint1
  - LED1 (0x0006)
- Endpoint2
  - LED2 (0x0006)
  - Temp Sensor (0x0002)

<a href="https://zigbeealliance.org/wp-content/uploads/2021/10/07-5123-08-Zigbee-Cluster-Library.pdf">ZLC Documentation</a>

### Attribute (~ data)

Attributes are identified by a 16-bit number, store the current “state” of a given cluster. A data entity which represents a physical quantity or state.
- A cluster can have multiple attributes.
- There are **generic ZCL (Zigbee Cluster Library) commands** to read and write attributes on any given cluster.
- Attributes can even be set up to **report automatically at regular intervals, if they change, or both**.

Full Espressif documentation <a href="https://docs.espressif.com/projects/esp-zigbee-sdk/en/latest/esp32/developing.html">here</a>

## How does this example work ?

We will use the default Zigbee configuration for using a thermostat and temperature measurement. Configurations are defined by Zigbee conventions.

We must create at least one endpoint for our application.

```c
/* Create customized thermostat endpoint */
esp_zb_thermostat_cfg_t thermostat_cfg = ESP_ZB_DEFAULT_THERMOSTAT_CONFIG();
esp_zb_ep_list_t *esp_zb_thermostat_ep = custom_thermostat_ep_create(HA_THERMOSTAT_ENDPOINT, &thermostat_cfg);
```

To create our endpoint, we need the Zigbee SDK function `esp_zb_ep_list_create()`. We will create and register our clusters to this new endpoint with a custom function `custom_thermostat_clusters_create(...)`. Then, we will add this configured endpoint to our application with the SDK function `esp_zb_ep_list_add_ep()`.
```c
esp_zb_ep_list_t *ep_list = esp_zb_ep_list_create();  
esp_zb_ep_list_add_ep(
    ep_list, 
    custom_thermostat_clusters_create(thermostat), 
    endpoint_id, 
    ESP_ZB_AF_HA_PROFILE_ID, 
    ESP_ZB_HA_THERMOSTAT_DEVICE_ID);
return ep_list;
```

To create endpoint clusters, we will create a `basic` cluster that holds all endpoint identification data. In this example, we define the manufacturer name as `Espressif` and the model identifier as `esp32h2`.


```c
// Create a cluster list for register all endpoint clusters
esp_zb_cluster_list_t *cluster_list = esp_zb_zcl_cluster_list_create();
// Create the basic cluster
esp_zb_attribute_list_t *basic_cluster = esp_zb_basic_cluster_create(&(thermostat->basic_cfg));
// Add 2 attributes to the basic cluster
ESP_ERROR_CHECK(esp_zb_basic_cluster_add_attr(basic_cluster, ESP_ZB_ZCL_ATTR_BASIC_MANUFACTURER_NAME_ID, MANUFACTURER_NAME));
ESP_ERROR_CHECK(esp_zb_basic_cluster_add_attr(basic_cluster, ESP_ZB_ZCL_ATTR_BASIC_MODEL_IDENTIFIER_ID, MODEL_IDENTIFIER));
// Add the basic cluster to the cluster list
ESP_ERROR_CHECK(esp_zb_cluster_list_add_basic_cluster(cluster_list, basic_cluster, ESP_ZB_ZCL_CLUSTER_SERVER_ROLE));
// Add some ways to identify clusters
ESP_ERROR_CHECK(esp_zb_cluster_list_add_identify_cluster(cluster_list, esp_zb_identify_cluster_create(&(thermostat->identify_cfg)), ESP_ZB_ZCL_CLUSTER_SERVER_ROLE));
ESP_ERROR_CHECK(esp_zb_cluster_list_add_identify_cluster(cluster_list, esp_zb_zcl_attr_list_create(ESP_ZB_ZCL_CLUSTER_ID_IDENTIFY), ESP_ZB_ZCL_CLUSTER_CLIENT_ROLE));
// Add the thermostat cluster
ESP_ERROR_CHECK(esp_zb_cluster_list_add_thermostat_cluster(cluster_list, esp_zb_thermostat_cluster_create(&(thermostat->thermostat_cfg)), ESP_ZB_ZCL_CLUSTER_SERVER_ROLE));
/* Add temperature measurement cluster for attribute reporting */
ESP_ERROR_CHECK(esp_zb_cluster_list_add_temperature_meas_cluster(cluster_list, esp_zb_temperature_meas_cluster_create(NULL), ESP_ZB_ZCL_CLUSTER_CLIENT_ROLE));
```

After that, we can register this created endpoint with the gateway
```c
static void esp_zb_task(void *pvParameters){
  ...
  esp_zb_device_register(esp_zb_thermostat_ep);
  ...
}
```

We will register a handler for Zigbee events (in this case, when a report is received by the gateway from the client).

```c
static esp_err_t zb_action_handler(esp_zb_core_action_callback_id_t callback_id, const void *message)
{
    esp_err_t ret = ESP_OK;
    switch (callback_id) {
    case ESP_ZB_CORE_REPORT_ATTR_CB_ID:
        ret = zb_attribute_reporting_handler((esp_zb_zcl_report_attr_message_t *)message);
        break;    
    default:
        ESP_LOGW(TAG, "Receive Zigbee action(0x%x) callback", callback_id);
        break;
    }
    return ret;
}


static void esp_zb_task(void *pvParameters){
  ...
  esp_zb_core_action_handler_register(zb_action_handler);
  ...
}
```

When a report is received, we will print it on the logs.

```c
static esp_err_t zb_attribute_reporting_handler(const esp_zb_zcl_report_attr_message_t *message)
{
    ESP_RETURN_ON_FALSE(message, ESP_FAIL, TAG, "Empty message");
    ESP_RETURN_ON_FALSE(message->status == ESP_ZB_ZCL_STATUS_SUCCESS, ESP_ERR_INVALID_ARG, TAG, "Received message: error status(%d)",
                        message->status);
    ESP_LOGI(TAG, "Received report from address(0x%x) src endpoint(%d) to dst endpoint(%d) cluster(0x%x)",
             message->src_address.u.short_addr, message->src_endpoint,
             message->dst_endpoint, message->cluster);
    esp_app_zb_attribute_handler(message->cluster, &message->attribute);
    return ESP_OK;
}
```

Then we will print attributes from clusters (Basic and Temperature)

```c
static void esp_app_zb_attribute_handler(uint16_t cluster_id, const esp_zb_zcl_attribute_t* attribute)
{
    /* Basic cluster attributes */
    if (cluster_id == ESP_ZB_ZCL_CLUSTER_ID_BASIC) {
        if (attribute->id == ESP_ZB_ZCL_ATTR_BASIC_MANUFACTURER_NAME_ID &&
            attribute->data.type == ESP_ZB_ZCL_ATTR_TYPE_CHAR_STRING &&
            attribute->data.value) {
            zbstring_t *zbstr = (zbstring_t *)attribute->data.value;
            char *string = (char*)malloc(zbstr->len + 1);
            memcpy(string, zbstr->data, zbstr->len);
            string[zbstr->len] = '\0';
            ESP_LOGI(TAG, "Peer Manufacturer is \"%s\"", string);
            free(string);
        }
        if (attribute->id == ESP_ZB_ZCL_ATTR_BASIC_MODEL_IDENTIFIER_ID &&
            attribute->data.type == ESP_ZB_ZCL_ATTR_TYPE_CHAR_STRING &&
            attribute->data.value) {
            zbstring_t *zbstr = (zbstring_t *)attribute->data.value;
            char *string = (char*)malloc(zbstr->len + 1);
            memcpy(string, zbstr->data, zbstr->len);
            string[zbstr->len] = '\0';
            ESP_LOGI(TAG, "Peer Model is \"%s\"", string);
            free(string);
        }
    }

    /* Temperature Measurement cluster attributes */
    if (cluster_id == ESP_ZB_ZCL_CLUSTER_ID_TEMP_MEASUREMENT) {
        if (attribute->id == ESP_ZB_ZCL_ATTR_TEMP_MEASUREMENT_VALUE_ID &&
            attribute->data.type == ESP_ZB_ZCL_ATTR_TYPE_S16) {
            int16_t value = attribute->data.value ? *(int16_t *)attribute->data.value : 0;
            ESP_LOGI(TAG, "Measured Value is %.2f degrees Celsius", zb_s16_to_temperature(value));
        }
        if (attribute->id == ESP_ZB_ZCL_ATTR_TEMP_MEASUREMENT_MIN_VALUE_ID &&
            attribute->data.type == ESP_ZB_ZCL_ATTR_TYPE_S16) {
            int16_t min_value = attribute->data.value ? *(int16_t *)attribute->data.value : 0;
            ESP_LOGI(TAG, "Min Measured Value is %.2f degrees Celsius", zb_s16_to_temperature(min_value));
        }
        if (attribute->id == ESP_ZB_ZCL_ATTR_TEMP_MEASUREMENT_MAX_VALUE_ID &&
            attribute->data.type == ESP_ZB_ZCL_ATTR_TYPE_S16) {
            int16_t max_value = attribute->data.value ? *(int16_t *)attribute->data.value : 0;
            ESP_LOGI(TAG, "Max Measured Value is %.2f degrees Celsius", zb_s16_to_temperature(max_value));
        }
        if (attribute->id == ESP_ZB_ZCL_ATTR_TEMP_MEASUREMENT_TOLERANCE_ID &&
            attribute->data.type == ESP_ZB_ZCL_ATTR_TYPE_U16) {
            uint16_t tolerance = attribute->data.value ? *(uint16_t *)attribute->data.value : 0;
            ESP_LOGI(TAG, "Tolerance is %.2f degrees Celsius", 1.0 * tolerance / 100);
        }
    }
}
```

So, here is a schema of the Zigbee node we produced with those pieces of code.

* Node :
  * Endpoint (ID=1 / PROFIL_ID=260 / DEVICE_ID=769 (Thermostat) )
    * Basic Cluster (ID=0x0000)
      * Attribute ID=4 : Manufacturer name
      * Attribute ID=5 : Model identifier
    * Thermostat Cluster (ID=0x0201)
      * Default attributes
    * Temperature Measure Cluster (ID=0x0402)
      * Default attributes



## Example Output

As you run the example, you will see the following log:

```
I (5076) ESP_ZB_THERMOSTAT: New device commissioned or rejoined (short: 0xc4c8)
I (5126) ESP_ZB_THERMOSTAT: Found temperature sensor
I (5126) ESP_ZB_THERMOSTAT: Request temperature sensor to bind us
I (5126) ESP_ZB_THERMOSTAT: Bind temperature sensor
I (5136) ESP_ZB_THERMOSTAT: Successfully bind the temperature sensor from address(0xc4c8) on endpoint(10)
I (5166) ESP_ZB_THERMOSTAT: The temperature sensor from address(0xc4c8) on endpoint(10) successfully binds us
I (5196) ESP_ZB_THERMOSTAT: ZDO signal: NLME Status Indication (0x32), status: ESP_OK
W (5196) ESP_ZB_THERMOSTAT: Receive Zigbee action(0x1000) callback
I (5626) ESP_ZB_THERMOSTAT: Received report from address(0xc4c8) src endpoint(10) to dst endpoint(1) cluster(0x402)
I (5626) ESP_ZB_THERMOSTAT: Measured Value is 42.00 degrees Celsius
I (10696) ESP_ZB_THERMOSTAT: Received report from address(0xc4c8) src endpoint(10) to dst endpoint(1) cluster(0x402)
I (10706) ESP_ZB_THERMOSTAT: Measured Value is 42.00 degrees Celsius
```

The client sends an update every 5 seconds with its current temperature value by default. If this value changes on the client, you will see it on the gateway.

```
I (35496) ESP_ZB_THERMOSTAT: Measured Value is 40.00 degrees Celsius
I (36526) ESP_ZB_THERMOSTAT: Received report from address(0xc4c8) src endpoint(10) to dst endpoint(1) cluster(0x402)
I (36536) ESP_ZB_THERMOSTAT: Measured Value is 17.00 degrees Celsius
I (37576) ESP_ZB_THERMOSTAT: Received report from address(0xc4c8) src endpoint(10) to dst endpoint(1) cluster(0x402)
I (37586) ESP_ZB_THERMOSTAT: Measured Value is 1.00 degrees Celsius
I (40156) ESP_ZB_THERMOSTAT: Received report from address(0xc4c8) src endpoint(10) to dst endpoint(1) cluster(0x402)
I (40166) ESP_ZB_THERMOSTAT: Measured Value is -10.00 degrees Celsius
```
