# HOME_AUTOMATION-USING-ESP32
Final Year Project: A WiFi-based Home Automation System using ESP32 and Embedded C. Designed an HTTP web server for real-time control of multiple home appliances through a browser interface. Demonstrates IoT, networking, and embedded system integration.


#include <stdio.h>
#include <string.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "esp_wifi.h"
#include "esp_event.h"
#include "esp_log.h"
#include "nvs_flash.h"
#include "esp_netif.h"
#include "esp_http_server.h"
#include "driver/gpio.h"

#define WIFI_SSID      "YOUR_WIFI_NAME"
#define WIFI_PASS      "YOUR_WIFI_PASSWORD"

#define LED_GPIO       GPIO_NUM_2

static const char *TAG = "HOME_AUTOMATION";

static httpd_handle_t server = NULL;

/* WiFi Event Handler */
static void wifi_event_handler(void* arg, esp_event_base_t event_base,
                                int32_t event_id, void* event_data)
{
    if (event_base == WIFI_EVENT && event_id == WIFI_EVENT_STA_START) {
        esp_wifi_connect();
    }
}

/* HTTP GET Handler */
esp_err_t on_handler(httpd_req_t *req)
{
    gpio_set_level(LED_GPIO, 1);
    httpd_resp_send(req, "Device ON", HTTPD_RESP_USE_STRLEN);
    return ESP_OK;
}

esp_err_t off_handler(httpd_req_t *req)
{
    gpio_set_level(LED_GPIO, 0);
    httpd_resp_send(req, "Device OFF", HTTPD_RESP_USE_STRLEN);
    return ESP_OK;
}

/* Start Web Server */
httpd_handle_t start_webserver(void)
{
    httpd_config_t config = HTTPD_DEFAULT_CONFIG();
    httpd_start(&server, &config);

    httpd_uri_t on_uri = {
        .uri       = "/on",
        .method    = HTTP_GET,
        .handler   = on_handler
    };
    httpd_register_uri_handler(server, &on_uri);

    httpd_uri_t off_uri = {
        .uri       = "/off",
        .method    = HTTP_GET,
        .handler   = off_handler
    };
    httpd_register_uri_handler(server, &off_uri);

    return server;
}

/* WiFi Init */
void wifi_init(void)
{
    esp_netif_init();
    esp_event_loop_create_default();
    esp_netif_create_default_wifi_sta();

    wifi_init_config_t cfg = WIFI_INIT_CONFIG_DEFAULT();
    esp_wifi_init(&cfg);

    esp_event_handler_instance_register(WIFI_EVENT,
                                         ESP_EVENT_ANY_ID,
                                         &wifi_event_handler,
                                         NULL,
                                         NULL);

    wifi_config_t wifi_config = {
        .sta = {
            .ssid = WIFI_SSID,
            .password = WIFI_PASS,
        },
    };

    esp_wifi_set_mode(WIFI_MODE_STA);
    esp_wifi_set_config(WIFI_IF_STA, &wifi_config);
    esp_wifi_start();
}

/* Main Function */
void app_main(void)
{
    nvs_flash_init();

    gpio_reset_pin(LED_GPIO);
    gpio_set_direction(LED_GPIO, GPIO_MODE_OUTPUT);

    wifi_init();

    vTaskDelay(5000 / portTICK_PERIOD_MS);

    start_webserver();

    ESP_LOGI(TAG, "Webserver started");
}
