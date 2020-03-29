# WIFI


/*               Notes about WiFi Programming
 *
 *  The esp32 WiFi programming model can be depicted as following picture:
 *
 *
 *                            default handler              user handler
 *  -------------             ---------------             ---------------
 *  |           |   event     |             | callback or |             |
 *  |   tcpip   | --------->  |    event    | ----------> | application |
 *  |   stack   |             |     task    |  event      |    task     |
 *  |-----------|             |-------------|             |-------------|
 *                                  /|\                          |
 *                                   |                           |
 *                            event  |                           |
 *                                   |                           |
 *                                   |                           |
 *                             ---------------                   |
 *                             |             |                   |
 *                             | WiFi Driver |/__________________|
 *                             |             |\     API call
 *                             |             |
 *                             |-------------|
 *
 */
##Station
* init
nvs_flash_init();

    s_wifi_event_group = xEventGroupCreate();
	esp_netif_init();	//Init TCP/IP, 实际上没用
	esp_event_loop_create_default()); //create default event handler s_default_loop
    esp_netif_create_default_wifi_sta();

    wifi_init_config_t cfg = WIFI_INIT_CONFIG_DEFAULT();
    esp_wifi_init(&cfg); 
	/* 注册了 WIFI_EVENT->WIFI_EVENT_STA_START-> wifi_create_and_start_sta
			  WIFI_EVENT->WIFI_EVENT_AP_START->wifi_create_and_start_ap
			  WIFI_EVENT->WIFI_EVENT_STA_START-> wifi_default_action_sta_start;
			WIFI_EVENT, WIFI_EVENT_STA_STOP, wifi_default_action_sta_stop
			WIFI_EVENT, WIFI_EVENT_STA_CONNECTED, wifi_default_action_sta_connected
			WIFI_EVENT, WIFI_EVENT_STA_DISCONNECTED, wifi_default_action_sta_disconnected
			WIFI_EVENT, WIFI_EVENT_AP_START, wifi_default_action_ap_start
			WIFI_EVENT, WIFI_EVENT_AP_STOP, wifi_default_action_ap_stop
			IP_EVENT, IP_EVENT_STA_GOT_IP, wifi_default_action_sta_got_ip
			esp_register_shutdown_handler((shutdown_handler_t)esp_wifi_stop);	
	*/
	//add event and handler to s_default_loop
    esp_event_handler_register(WIFI_EVENT, ESP_EVENT_ANY_ID, &event_handler, NULL));
    esp_event_handler_register(IP_EVENT, IP_EVENT_STA_GOT_IP, &event_handler, NULL));
	
    esp_wifi_set_mode(WIFI_MODE_STA);
    wifi_config_t wifi_config = {
        .sta = {
            .ssid = EXAMPLE_ESP_WIFI_SSID,
            .password = EXAMPLE_ESP_WIFI_PASS
        },
    };	
    esp_wifi_set_config(ESP_IF_WIFI_STA, &wifi_config);
    esp_wifi_start();
	/* get IF MAC from driver
	   set MAC to netif
	   start netif 
	   */
	   
* event handler

    if (event_base == WIFI_EVENT && event_id == WIFI_EVENT_STA_START) {
        esp_wifi_connect();
    } else if (event_base == WIFI_EVENT && event_id == WIFI_EVENT_STA_DISCONNECTED) {
        if (s_retry_num < EXAMPLE_ESP_MAXIMUM_RETRY) {
            esp_wifi_connect();
            xEventGroupClearBits(s_wifi_event_group, WIFI_CONNECTED_BIT);
            s_retry_num++;
        }
    } else if (event_base == IP_EVENT && event_id == IP_EVENT_STA_GOT_IP) {
        ip_event_got_ip_t* event = (ip_event_got_ip_t*) event_data;
        ESP_LOGI(TAG, "got ip:" IPSTR, IP2STR(&event->ip_info.ip));
        s_retry_num = 0;
        xEventGroupSetBits(s_wifi_event_group, WIFI_CONNECTED_BIT);
    }
