# SWF siwx917

- `main()`:

    ```C
    int main(void)
    {
    int32_t status = RSI_SUCCESS;
    sl_cts_local_time_info_t local_time = { 0 };

    // Driver initialization
    status = rsi_driver_init(driver_buf, SL_DRIVER_BUF_LEN);
    if (( status < 0 ) || ( status > SL_DRIVER_BUF_LEN )) {
        return status;
    }

    // Silicon labs module intialisation
    status = rsi_device_init(LOAD_NWP_FW);
    if (status != RSI_SUCCESS) {
        SL_LOG_ERR("Device Initialization Failed, Error Code: 0x%08X",
                (uint32_t)(status));
        return status;
    }

    sl_log_init();
    sl_rtc_init();
    sl_sched_init();
    tmc2300_init();

    // Initialize default time zone: GTM +7
    local_time.time_zone = SL_TIME_ZONE;
    local_time.dst_offset = SL_DST_OFFSET;
    sl_sntp_set_offset_time(local_time);
    sl_softtimer_init();

    SL_LOG_INFO("Device Initialization Success");

    // Create application task
    rsi_task_create((rsi_task_function_t)(main_app_task),
                    (uint8_t *)("main_task"),
                    SL_MAIN_TASK_STACK_SIZE,
                    NULL,
                    SL_MAIN_TASK_PRIORITY,
                    &sl_main_task_handle);

    // Start the scheduler
    rsi_start_os_scheduler( );
    }

    ```

  - The main function first call `rsi_driver_init()` to init the wiSeConnect Driver. This is a **non-blocking** API. Designate memory to all driver components from the buffer `driver_buf`. And then it initialize Scheduler, Events, and Queues needed.

    ```C
    #define SL_DRIVER_BUF_LEN 15000 /**< Driver buffer length */

    /**
    * Memory buffer for driver initialization
    */
    static uint8_t driver_buf[SL_DRIVER_BUF_LEN] = { 0 };
    ```

    - The API receive first parameter as a pointer to buffer from application, Driver uses this buffer to hold driver control for its operation. And second parameter is  a length of the buffer.
    - This API returns the memory used, which is less than or equal to buffer length provided. If failure, it return maximum sockets is greater than 10.

  - After driver initialization, device is init with API `rsi_device_init()`. This API power cycle the module and set the firmware image type be loaded for WiSeConnect features. Initialize the module SPI. This is a blocking API.
    - `rsi_device_init()` must be called before this API.
    - The API only receives one parameter that provides two options to load Firmware image:
      - `LOAD_NWP_FW`: To load firmware image.
      - `LOAD_DEFAULT_NWP_FW_ACTIVE_LOW`: To load active low Firmware image.
      - Active low firmware will generate active low interrupts to indicate that packets are pending on the module, instead of the default active high.

  - Next, we initialize logging system `sl_log_init()` that function create the mutex for logging and create UART pipeline for logging:

    ```C
    void sl_log_init()
    {
    #if SL_ENABLE_LOG_DIRECT_VIA_UART
    rsi_mutex_create(&log_lock);
    #endif

    sl_uart_init(SL_UART_LOG, 115200);
    }
    ```

  - After init the logging system for debug purpose, we need to init RTC module `sl_rtc_init();` this function first select clock source for FSM low frequency clock `RSI_PS_FsmLfClkSel(KHZ_RC_CLK_SEL)`, we use RC clock for FSM (Clock source can be RO or RC or XTAL).