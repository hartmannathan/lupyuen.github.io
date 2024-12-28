# Reverse Engineering WiFi on RISC-V BL602

📝 _7 Jul 2021_

Today we shall __Reverse Engineer the WiFi Driver__ on the [__BL602 RISC-V + WiFi SoC__](https://lupyuen.github.io/articles/pinecone) and learn what happens inside... Guided by the (incomplete) source code that we found for the driver.

Why Reverse Engineer the BL602 WiFi Driver?

1.  __Education__: To learn how WiFi Packets are transmitted and received on BL602.

1.  __Troubleshooting__: If the WiFi Driver doesn't work right, we should be able to track down the problem. (Maybe fix it too!)

1.  __Auditing__: To be sure that WiFi Packets are transmitted / received correctly and securely. [(See this non-BL602 example)](https://twitter.com/Yu_Wei_Wu/status/1406940637773979655?s=19)

1.  __Replacement__: Perhaps one day we might replace the Closed-Source WiFi Driver by an Open Source driver. [(Maybe Openwifi?)](https://github.com/open-sdr/openwifi)

Read on and join me on the Reverse Engineering journey!

![Quantitative Analysis of Decompiled BL602 WiFi Firmware](https://lupyuen.github.io/images/wifi-title.jpg)

# BL602 WiFi Demo Firmware

Let's study the source code of the __BL602 WiFi Demo Firmware__ from the BL602 IoT SDK: [__`bl602_demo_wifi`__](https://github.com/lupyuen/bl_iot_sdk/blob/master/customer_app/bl602_demo_wifi)

In the demo firmware we shall...

1.  Register the __WiFi Event Handler__ that will handle WiFi Events

1.  Start the __WiFi Firmware Task__ that will control the BL602 WiFi Firmware

1.  Start the __WiFi Manager Task__ that will manage the WiFi Connection State

1.  Connect to a __WiFi Access Point__

1.  Send a __HTTP Request__

## Register WiFi Event Handler

When the firmware starts, we register a __Callback Function that will handle WiFi Events__: [`main.c`](https://github.com/lupyuen/bl_iot_sdk/blob/master/customer_app/bl602_demo_wifi/bl602_demo_wifi/main.c#L819-L866)

```c
//  Called at startup to init drivers and run event loop
static void aos_loop_proc(void *pvParameters) {
  //  Omitted: Init the drivers
  ...
  //  Register Callback Function for WiFi Events
  aos_register_event_filter(
    EV_WIFI,              //  Event Type
    event_cb_wifi_event,  //  Event Callback Function
    NULL);                //  Event Callback Argument

  //  Start WiFi Networking Stack
  cmd_stack_wifi(NULL, 0, 0, NULL);

  //  Run event loop
  aos_loop_run();
}
```

(We'll see `event_cb_wifi_event` in a while)

This startup code calls __`cmd_stack_wifi`__ to start the WiFi Networking Stack.

Let's look inside...

## Start WiFi Firmware Task

In __`cmd_stack_wifi`__ we start the __WiFi Firmware Task__ like so: [`main.c`](https://github.com/lupyuen/bl_iot_sdk/blob/master/customer_app/bl602_demo_wifi/bl602_demo_wifi/main.c#L729-L747)

```c
//  Start WiFi Networking Stack
static void cmd_stack_wifi(char *buf, int len, int argc, char **argv) {
  //  Check whether WiFi Networking is already started
  static uint8_t stack_wifi_init  = 0;
  if (1 == stack_wifi_init) { return; }  //  Already started
  stack_wifi_init = 1;

  //  Start WiFi Firmware Task (FreeRTOS)
  hal_wifi_start_firmware_task();

  //  Post a WiFi Event to start WiFi Manager Task
  aos_post_event(
    EV_WIFI,                 //  Event Type
    CODE_WIFI_ON_INIT_DONE,  //  Event Code
    0);                      //  Event Argument
}
```

(We'll cover `hal_wifi_start_firmware_task` later in this article)

After starting the task, we post the WiFi Event `CODE_WIFI_ON_INIT_DONE` to __start the WiFi Manager Task__.

Let's look inside the WiFi Event Handler...

## Start WiFi Manager Task

Here's how we handle __WiFi Events__: [`main.c`](https://github.com/lupyuen/bl_iot_sdk/blob/master/customer_app/bl602_demo_wifi/bl602_demo_wifi/main.c#L374-L512)

```c
//  Callback Function for WiFi Events
static void event_cb_wifi_event(input_event_t *event, void *private_data) {

  //  Handle the WiFi Event
  switch (event->code) {

    //  Posted by cmd_stack_wifi to start Wi-Fi Manager Task
    case CODE_WIFI_ON_INIT_DONE:

      //  Start the WiFi Manager Task (FreeRTOS)
      wifi_mgmr_start_background(&conf);
      break;

    //  Omitted: Handle other WiFi Events
```

When we receive the WiFi Event `CODE_WIFI_ON_INIT_DONE`, we start the __WiFi Manager Task__ (in FreeRTOS) by calling `wifi_mgmr_start_background`.

`wifi_mgmr_start_background` comes from the BL602 WiFi Driver. [(See the source code)](https://github.com/lupyuen/bl_iot_sdk/blob/master/components/bl602/bl602_wifidrv/bl60x_wifi_driver/wifi_mgmr.c#L1406-L1415)

## Connect to WiFi Network

Now that we have started both WiFi Background Tasks (WiFi Firmware Task and WiFi Manager Task), let's connect to a WiFi Network!

The demo firmware lets us enter this command to __connect to a WiFi Access Point__...

```text
wifi_sta_connect YOUR_WIFI_SSID YOUR_WIFI_PASSWORD
```

Here's how the __`wifi_sta_connect`__ command is implemented: [`main.c`](https://github.com/lupyuen/bl_iot_sdk/blob/master/customer_app/bl602_demo_wifi/bl602_demo_wifi/main.c#L366-L372)

```c
//  Connect to WiFi Access Point
static void wifi_sta_connect(char *ssid, char *password) {

  //  Enable WiFi Client
  wifi_interface_t wifi_interface
    = wifi_mgmr_sta_enable();

  //  Connect to WiFi Access Point
  wifi_mgmr_sta_connect(
    wifi_interface,  //  WiFi Interface
    ssid,            //  SSID
    password,        //  Password
    NULL,            //  PMK
    NULL,            //  MAC Address
    0,               //  Band
    0);              //  Frequency
}
```

We call [__`wifi_mgmr_sta_enable`__](https://github.com/lupyuen/bl_iot_sdk/blob/master/components/bl602/bl602_wifidrv/bl60x_wifi_driver/wifi_mgmr_ext.c#L202-L217) from the BL602 WiFi Driver to __enable the WiFi Client__.

("STA" refers to "WiFi Station", which means WiFi Client)

Then we call [__`wifi_mgmr_sta_connect`__](https://github.com/lupyuen/bl_iot_sdk/blob/master/components/bl602/bl602_wifidrv/bl60x_wifi_driver/wifi_mgmr_ext.c#L302-L307) (also from the BL602 WiFi Driver) to __connect to the WiFi Access Point__.

(We'll study the internals of `wifi_mgmr_sta_connect` in the next chapter)

## Send HTTP Request

Now we enter this command to __send a HTTP Request__ over WiFi...

```text
httpc
```

Here's the implementation of the __`httpc`__ command: [`main.c`](https://github.com/lupyuen/bl_iot_sdk/blob/master/customer_app/bl602_demo_wifi/bl602_demo_wifi/main.c#L704-L727)

```c
//  Send a HTTP GET Request with LWIP
static void cmd_httpc_test(char *buf, int len, int argc, char **argv) {
  //  Check whether a HTTP Request is already running
  static httpc_connection_t settings;
  static httpc_state_t *req;
  if (req) { return; }  //  Request already running

  //  Init the LWIP HTTP Settings
  memset(&settings, 0, sizeof(settings));
  settings.use_proxy = 0;
  settings.result_fn = cb_httpc_result;
  settings.headers_done_fn = cb_httpc_headers_done_fn;

  //  Send a HTTP GET Request with LWIP
  httpc_get_file_dns(
    "nf.cr.dandanman.com",  //  Host
    80,                     //  Port
    "/ddm/ContentResource/music/204.mp3",  //  URI
    &settings,              //  Settings
    cb_altcp_recv_fn,       //  Callback Function
    &req,                   //  Callback Argument
    &req);                  //  Request
}
```

On BL602 we use [__LWIP, the Lightweight IP Stack__](https://www.nongnu.org/lwip/2_1_x/index.html) to do IP, UDP, TCP and HTTP Networking.

[`httpc_get_file_dns` is documented here](https://www.nongnu.org/lwip/2_1_x/group__httpc.html#gabd4ef2259885a93090733235cc0fa8d6)

For more details on the BL602 WiFi Demo Firmware, check out the docs...

-   [__BL602 WiFi Demo Firmware Docs__](https://pine64.github.io/bl602-docs/Examples/demo_wifi/wifi.html)

Let's reverse engineer the BL602 WiFi Demo Firmware... And learn what happens inside!

![Connecting to WiFi Access Point](https://lupyuen.github.io/images/wifi-connect.png)

# Connect to WiFi Access Point

_What really happens when BL602 connects to a WiFi Access Point?_

To understand how BL602 connects to a WiFi Access Point, let's read the __Source Code from the BL602 WiFi Driver__.

Watch what happens as we...

1.  Send the Connect Request to the __WiFi Manager Task__

1.  Process the Connect Request with __WiFi Manager's State Machine__

1.  Forward the Connect Request to the __WiFi Hardware (LMAC)__

1.  Trigger an __LMAC Interrupt__ to perform the request

## Send request to WiFi Manager Task

Earlier we called __`wifi_mgmr_sta_connect`__ to connect to the WiFi Access Point.

Here's what happens inside: [`wifi_mgmr_ext.c`](https://github.com/lupyuen/bl_iot_sdk/blob/master/components/bl602/bl602_wifidrv/bl60x_wifi_driver/wifi_mgmr_ext.c#L302-L307)

```c
//  Connect to WiFi Access Point
int wifi_mgmr_sta_connect(wifi_interface_t *wifi_interface, char *ssid, char *psk, char *pmk, uint8_t *mac, uint8_t band, uint16_t freq) {
  //  Set WiFi SSID and PSK
  wifi_mgmr_sta_ssid_set(ssid);
  wifi_mgmr_sta_psk_set(psk);

  //  Connect to WiFi Access Point
  return wifi_mgmr_api_connect(ssid, psk, pmk, mac, band, freq);
}
```

We set the WiFi SSID and PSK. Then we call `wifi_mgmr_api_connect` to connect to the access point.

__`wifi_mgmr_api_connect`__ does this: [`wifi_mgmr_api.c`](https://github.com/lupyuen/bl_iot_sdk/blob/master/components/bl602/bl602_wifidrv/bl60x_wifi_driver/wifi_mgmr_api.c#L40-L84)

```c
//  Connect to WiFi Access Point
int wifi_mgmr_api_connect(char *ssid, char *psk, char *pmk, uint8_t *mac, uint8_t band, uint16_t freq) {
  //  Omitted: Copy PSK, PMK, MAC Address, Band and Frequency
  ...
  //  Send Connect Request to WiFi Manager Task
  wifi_mgmr_event_notify(msg);
  return 0;
}
```

![wifi_mgmr_api_connect](https://lupyuen.github.io/images/wifi-connect2.png)

Here we call `wifi_mgmr_event_notify` to __send the Connect Request__ to the WiFi Manager Task.

__`wifi_mgmr_event_notify`__ is defined in [`wifi_mgmr.c`](https://github.com/lupyuen/bl_iot_sdk/blob/master/components/bl602/bl602_wifidrv/bl60x_wifi_driver/wifi_mgmr.c#L1332-L1343) ...

```c
//  Send request to WiFi Manager Task
int wifi_mgmr_event_notify(wifi_mgmr_msg_t *msg) {
  //  Omitted: Wait for WiFi Manager to start
  ...
  //  Send request to WiFi Manager via Message Queue
  if (os_mq_send(
    &(wifiMgmr.mq),  //  Message Queue
    msg,             //  Request Message
    msg->len)) {     //  Message Length
    //  Failed to send request
    return -1;
  }
  return 0;
}
```

_How does `os_mq_send` send the request to the WiFi Manager Task?_

__`os_mq_send`__ calls FreeRTOS to deliver the Request Message to __WiFi Manager's Message Queue__: [`os_hal.h`](https://github.com/lupyuen/bl_iot_sdk/blob/master/components/bl602/bl602_wifidrv/bl60x_wifi_driver/os_hal.h#L174)

```c
#define os_mq_send(mq, msg, len) \
    (xMessageBufferSend(mq, msg, len, portMAX_DELAY) > 0 ? 0 : 1)
```

![wifi_mgmr_event_notify](https://lupyuen.github.io/images/wifi-connect3.png)

## WiFi Manager State Machine

The WiFi Manager runs a __State Machine__ in its Background Task (FreeRTOS) to manage the state of each WiFi Connection.

_What happens when WiFi Manager receives our request to connect to a WiFi Access Point?_

Let's find out in [`wifi_mgmr.c`](https://github.com/lupyuen/bl_iot_sdk/blob/master/components/bl602/bl602_wifidrv/bl60x_wifi_driver/wifi_mgmr.c#L702-L745) ...

```c
//  Called when WiFi Manager receives Connect Request
static void stateIdleAction_connect( void *oldStateData, struct event *event, void *newStateData) {
  //  Set the WiFi Profile for the Connect Request
  wifi_mgmr_msg_t *msg = event->data;
  wifi_mgmr_profile_msg_t *profile_msg = (wifi_mgmr_profile_msg_t*) msg->data;
  profile_msg->ssid_tail[0] = '\0';
  profile_msg->psk_tail[0]  = '\0';

  //  Remember the WiFi Profile in the WiFi Manager
  wifi_mgmr_profile_add(&wifiMgmr, profile_msg, -1);

  //  Connect to the WiFi Profile. TODO: Other security support
  bl_main_connect(
    (const uint8_t *) profile_msg->ssid, profile_msg->ssid_len,
    (const uint8_t *) profile_msg->psk, profile_msg->psk_len,
    (const uint8_t *) profile_msg->pmk, profile_msg->pmk_len,
    (const uint8_t *) profile_msg->mac, (const uint8_t) profile_msg->band, (const uint16_t) profile_msg->freq);
}
```

![stateIdleAction_connect](https://lupyuen.github.io/images/wifi-connect4.png)

Here we set the __WiFi Profile__ and call `bl_main_connect` to connect to the profile.

In __`bl_main_connect`__ we set the __Connection Parameters for the 802.11 WiFi Protocol__: [`bl_main.c`](https://github.com/lupyuen/bl_iot_sdk/blob/master/components/bl602/bl602_wifidrv/bl60x_wifi_driver/bl_main.c#L189-L216)

```c
//  Connect to the WiFi Profile
int bl_main_connect(const uint8_t* ssid, int ssid_len, const uint8_t *psk, int psk_len, const uint8_t *pmk, int pmk_len, const uint8_t *mac, const uint8_t band, const uint16_t freq) {

  //  Connection Parameters for 802.11 WiFi Protocol
  struct cfg80211_connect_params sme;    

  //  Omitted: Set the 802.11 Connection Parameters
  ...
  //  Connect to WiFi Network with the 802.11 Connection Parameters
  bl_cfg80211_connect(&wifi_hw, &sme);
  return 0;
}
```

The Connection Parameters are passed to __`bl_cfg80211_connect`__, defined in [`bl_main.c`](https://github.com/lupyuen/bl_iot_sdk/blob/master/components/bl602/bl602_wifidrv/bl60x_wifi_driver/bl_main.c#L539-L571) ...

```c
//  Connect to WiFi Network with the 802.11 Connection Parameters
int bl_cfg80211_connect(struct bl_hw *bl_hw, struct cfg80211_connect_params *sme) {

  //  Will be populated with the connection result
  struct sm_connect_cfm sm_connect_cfm;

  //  Forward the Connection Parameters to the LMAC
  int error = bl_send_sm_connect_req(bl_hw, sme, &sm_connect_cfm);

  //  Omitted: Check connection result
```

Which calls __`bl_send_sm_connect_req`__ to send the Connection Parameters to the __WiFi Hardware (LMAC)__.

Let's dig in and find out how...

![bl_send_sm_connect_req](https://lupyuen.github.io/images/wifi-connect5.png)

## Send request to LMAC

_What is LMAC?_

__Lower Medium Access Control (LMAC)__ is the firmware that runs __inside the BL602 WiFi Radio Hardware__ and executes the WiFi Radio functions. 

(Like sending and receiving WiFi Packets)

To connect to a WiFi Access Point, we __pass the Connection Parameters to LMAC__ by calling __`bl_send_sm_connect_req`__, defined in [`bl_msg_tx.c`](https://github.com/lupyuen/bl_iot_sdk/blob/master/components/bl602/bl602_wifidrv/bl60x_wifi_driver/bl_msg_tx.c#L722-L804) ...

```c
//  Forward the Connection Parameters to the LMAC
int bl_send_sm_connect_req(struct bl_hw *bl_hw, struct cfg80211_connect_params *sme, struct sm_connect_cfm *cfm) {

  //  Build the SM_CONNECT_REQ message
  struct sm_connect_req *req = bl_msg_zalloc(SM_CONNECT_REQ, TASK_SM, DRV_TASK_ID, sizeof(struct sm_connect_req));

  //  Omitted: Set parameters for the SM_CONNECT_REQ message
  ...
  //  Send the SM_CONNECT_REQ message to LMAC Firmware
  return bl_send_msg(bl_hw, req, 1, SM_CONNECT_CFM, cfm);
}
```

Here we compose an __`SM_CONNECT_REQ`__ message that contains the Connection Parameters.

("`SM`" refers to the LMAC State Machine by RivieraWaves)

Then we call __`bl_send_msg`__ to __send the message to LMAC__: [`bl_msg_tx.c`](https://github.com/lupyuen/bl_iot_sdk/blob/master/components/bl602/bl602_wifidrv/bl60x_wifi_driver/bl_msg_tx.c#L315-L371)

```c
//  Send message to LMAC Firmware
static int bl_send_msg(struct bl_hw *bl_hw, const void *msg_params, int reqcfm, lmac_msg_id_t reqid, void *cfm) {
  //  Omitted: Allocate a buffer for the message
  ...
  //  Omitted: Copy the message to the buffer
  ...
  //  Add the message to the LMAC Message Queue
  int ret = bl_hw->cmd_mgr.queue(&bl_hw->cmd_mgr, cmd);
```

![bl_send_msg](https://lupyuen.github.io/images/wifi-connect6.png)

The code above calls __`ipc_host_msg_push`__ to __add the message to the LMAC Message Queue__: [`ipc_host.c`](https://github.com/lupyuen/bl_iot_sdk/blob/master/components/bl602/bl602_wifidrv/bl60x_wifi_driver/ipc_host.c#L139-L171)

```c
//  Add the message to the LMAC Message Queue.
//  IPC = Interprocess Communication
int ipc_host_msg_push(struct ipc_host_env_tag *env, void *msg_buf, uint16_t len) {
    //  Get the address of the IPC message buffer in Shared RAM
    uint32_t *src = (uint32_t*) ((struct bl_cmd *) msg_buf)->a2e_msg;
    uint32_t *dst = (uint32_t*) &(env->shared->msg_a2e_buf.msg);

    //  Copy the message to the IPC message buffer
    for (int i = 0; i < len; i += 4) { *dst++ = *src++; }
    env->msga2e_hostid = msg_buf;

    //  Trigger an LMAC Interrupt to send the message to EMB
    //  IPC_IRQ_A2E_MSG is 2
    ipc_app2emb_trigger_set(IPC_IRQ_A2E_MSG);
```

![ipc_host_msg_push](https://lupyuen.github.io/images/wifi-connect9.png)

After copying the message to the LMAC Message Queue (in Shared RAM), we call `ipc_app2emb_trigger_set` to __trigger an LMAC Interrupt__.

LMAC (and the BL602 Radio Hardware) will then transmit the proper WiFi Packets to __establish a network connection__ with the WiFi Access Point.

And that's how BL602 connects to a WiFi Access Point!

![ipc_app2emb_trigger_set](https://lupyuen.github.io/images/wifi-connect8.png)

## Trigger LMAC Interrupt

_But how do we trigger an LMAC Interrupt?_

```c
//  Trigger an LMAC Interrupt to send the message to EMB
//  IPC_IRQ_A2E_MSG is 2
ipc_app2emb_trigger_set(IPC_IRQ_A2E_MSG);
```

Let's look inside __`ipc_app2emb_trigger_set`__ and learn how it triggers an __LMAC Interrupt__: [`reg_ipc_app.h`](https://github.com/lupyuen/bl_iot_sdk/blob/master/components/bl602/bl602_wifidrv/bl60x_wifi_driver/reg_ipc_app.h#L41-L69)

```c
//  WiFi Hardware Register Base Address
#define REG_WIFI_REG_BASE          0x44000000

//  IPC Hardware Register Base Address
#define IPC_REG_BASE_ADDR          0x00800000

//  APP2EMB_TRIGGER Register Definition
//  Bits    Field Name           Reset Value
//  -----   ------------------   -----------
//  31:00   APP2EMB_TRIGGER      0x0
#define IPC_APP2EMB_TRIGGER_ADDR   0x12000000
#define IPC_APP2EMB_TRIGGER_OFFSET 0x00000000
#define IPC_APP2EMB_TRIGGER_INDEX  0x00000000
#define IPC_APP2EMB_TRIGGER_RESET  0x00000000

//  Write to IPC Register
#define REG_IPC_APP_WR(env, INDEX, value) \
  (*(volatile u32 *) ((u8 *) env + IPC_REG_BASE_ADDR + 4*(INDEX)) \
    = value)

//  Trigger LMAC Interrupt
static inline void ipc_app2emb_trigger_set(u32 value) {
  //  Write to WiFi IPC Register at address 0x4480 0000
  REG_IPC_APP_WR(
    REG_WIFI_REG_BASE, 
    IPC_APP2EMB_TRIGGER_INDEX, 
    value);
}
```

This code triggers an LMAC Interrupt by writing to the __WiFi Hardware Register__ (for Interprocess Communication) at...

```text
REG_WIFI_REG_BASE + IPC_REG_BASE_ADDR + 4 * IPC_APP2EMB_TRIGGER_INDEX
```

Which gives us address __`0x4480 0000`__

_Wait... Is address `0x4480 0000` documented anywhere?_

Nope it's not documented in the [__BL602 Reference Manual__](https://github.com/bouffalolab/bl_docs/blob/main/BL602_RM/en/BL602_BL604_RM_1.2_en.pdf)...

![Undocumented WiFi Hardware Registers](https://lupyuen.github.io/images/wifi-connect7.png)

In fact the entire region of __WiFi Hardware Registers at `0x4400 0000`__ is undocumented.

We've just uncovered a BL602 WiFi Secret! 🤫

# Decompiled WiFi Demo Firmware

_Are we really Reverse Engineering the BL602 WiFi Driver?_

Not quite. So far we've been reading the __published source code__ for the BL602 WiFi Driver.

_Can we do some serious Reverse Engineering now?_

Most certainly! A big chunk of the BL602 WiFi Driver __doesn't come with any source code.__

(Like the functions for WiFi WPA Authentication)

But [__`BraveHeartFLOSSDev`__](https://github.com/BraveHeartFLOSSDev) did an excellent job decompiling into C (with Ghidra) the BL602 WiFi Demo Firmware...

-   [__BraveHeartFLOSSDev/bl602nutcracker1__](https://github.com/BraveHeartFLOSSDev/bl602nutcracker1)

[(We'll use this fork)](https://github.com/lupyuen/bl602nutcracker1)

We shall now study this Decompiled C Code... And do some serious Reverse Engineering of the BL602 WiFi Driver!

[More about Ghidra](https://ghidra-sre.org/)

[Ghidra configuration for BL602](https://github.com/BraveHeartFLOSSDev/GhidWork)

## Linking to decompiled code

Sadly GitHub won't show our huge Decompiled C Files in the web browser. So __deep-linking to specific lines of code__ will be a problem.

Here's the workaround for deep-linking...

1.  Download the repo of Decompiled C Files...

    ```bash
    git clone --recursive https://github.com/lupyuen/bl602nutcracker1
    ```

1.  When we see a link like this...

    > This is the decompiled code: [`bl602_demo_wifi.c`](https://github.com/lupyuen/bl602nutcracker1/blob/main/bl602_demo_wifi.c#L38512-L38609)

    __Right-click__ (or long press) the link.
    
    Select __`Copy Link Address`__

1.  __Paste the address__ into a text editor.

    We will see this...

    ```text
    https://github.com/lupyuen/bl602nutcracker1/blob/main/bl602_demo_wifi.c#L38512-L38609
    ```

1.  Note that the address ends with...

    ```text
    bl602_demo_wifi.c#L38512-L38609
    ```

1.  This means that we should...

    Open the downloaded source file __`bl602_demo_wifi.c`__ in our code editor (like VSCode)
    
    And jump to line number __`38512`__ (use Ctrl-G)

1.  And we shall see the referenced Decompiled C Code...

![Assertions](https://lupyuen.github.io/images/wifi-assert.png)

# WiFi Firmware Task

The BL602 WiFi Driver operates with two Background Tasks (FreeRTOS)...

-   __WiFi Manager Task__: Manages the WiFi Connection State

-   __WiFi Firmware Task__: Controls the WiFi Firmware

We've covered the WiFi Manager Task. (Remember the State Machine?)

Now we dive into the WiFi Firmware Task. Watch what happens as we...

1.  Start the __WiFi Firmware Task__

1.  __Schedule Kernel Events__ to handle WiFi Packets

1.  Handle the __transmission of WiFi Packets__

![Starting the WiFi Firmware Task](https://lupyuen.github.io/images/wifi-task.png)

## Start Firmware Task

Earlier we saw `cmd_stack_wifi` calling `hal_wifi_start_firmware_task` to start the Firmware Task.

Let's look inside __`hal_wifi_start_firmware_task`__ now: [`hal_wifi.c`](https://github.com/lupyuen/bl_iot_sdk/blob/master/components/hal_drv/bl602_hal/hal_wifi.c#L41-L49)

```c
//  Start WiFi Firmware Task (FreeRTOS)
int hal_wifi_start_firmware_task(void) {
  //  Stack space for the WiFi Firmware Task
  static StackType_t wifi_fw_stack[WIFI_STACK_SIZE];

  //  Task Handle for the WiFi Firmware Task
  static StaticTask_t wifi_fw_task;

  //  Create a FreeRTOS Background Task
  xTaskCreateStatic(
    wifi_main,         //  Task will run this function
    (char *) "fw",     //  Task Name
    WIFI_STACK_SIZE,   //  Task Stack Size
    NULL,              //  Task Parameters
    TASK_PRIORITY_FW,  //  Task Priority
    wifi_fw_stack,     //  Task Stack
    &wifi_fw_task);    //  Task Handle
  return 0;
}
```

This creates a __FreeRTOS Background Task__ that runs the function __`wifi_main`__ forever.

_What's inside `wifi_main`?_

We don't have the source code for `wifi_main`. But thanks to [`BraveHeartFLOSSDev`](https://github.com/BraveHeartFLOSSDev) we have the C code decompiled from the BL602 WiFi Firmware!

Here's __`wifi_main`__ from the Decompiled C Code: [`bl602_demo_wifi.c`](https://github.com/lupyuen/bl602nutcracker1/blob/main/bl602_demo_wifi.c#L32959-L33006)

```c
//  WiFi Firmware Task runs this forever
void wifi_main(void *param) {
  ...
  //  Init the LMAC and UMAC
  rfc_init(40000000);
  mpif_clk_init();
  sysctrl_init();
  intc_init();
  ipc_emb_init();
  bl_init();
  ...
  //  Loop forever handling WiFi Kernel Events
  do {
    ...
    //  Wait for something (?)
    if (ke_env.evt_field == 0) { ipc_emb_wait(); }
    ...
    //  Schedule a WiFi Kernel Event and handle it
    ke_evt_schedule();

    //  Sleep for a while
    iVar1 = bl_sleep();

    //  (Whaaat?)
    coex_wifi_pta_forece_enable((uint) (iVar1 == 0));
  } while( true );
}
```

The actual Decompiled C Code for `wifi_main` looks a lot noisier...

![wifi_main](https://lupyuen.github.io/images/wifi-task2.png)

So we picked the highlights for our Reverse Engineering.

(And we added some annotations too)

__`wifi_main`__ loops forever handling __WiFi Kernel Events__, to transmit and receive WiFi Packets.

("`ke`" refers to the WiFi Kernel, the heart of the WiFi Driver)

`wifi_main` calls __`ke_evt_schedule`__ to handle WiFi Kernel Events.

Let's lookup `ke_evt_schedule` in our Decompiled C Code: [`bl602_demo_wifi.c`](https://github.com/lupyuen/bl602nutcracker1/blob/main/bl602_demo_wifi.c#L28721-L28737)

```c
//  Schedule a WiFi Kernel Event and handle it
void ke_evt_schedule(void) {
  int iVar1;
  evt_ptr_t *peVar2;
  
  while (ke_env.evt_field != 0) {
    iVar1 = __clzsi2(ke_env.evt_field);
    peVar2 = ke_evt_hdlr[iVar1].func;
    if ((0x1a < iVar1) || (peVar2 == (evt_ptr_t *)0x0)) {
      assert_err("(event < KE_EVT_MAX) && ke_evt_hdlr[event].func","module",0xdd);
    }
    (*peVar2)(ke_evt_hdlr[iVar1].param);
  }
  //  This line is probably incorrectly decompiled
  gp = (code *)((int)SFlash_Cache_Hit_Count_Get + 6);
  return;
}
```

This decompiled code does... something something something.

Thankfully [This One Weird Trick](https://en.wikipedia.org/wiki/One_weird_trick_advertisements) will help us understand this cryptic decompiled code... __GitHub Search!__

![Searching for ke_evt_schedule](https://lupyuen.github.io/images/wifi-schedule2.png)

## Schedule Kernel Events

_Is it possible that `ke_evt_schedule` wasn't invented for BL602?_

_Maybe `ke_evt_schedule` was created for something else?_

Bingo! Let's do a __GitHub Search__ for `ke_evt_schedule`!

-   [__GitHub Code Search for `ke_evt_schedule`__](https://github.com/search?o=desc&q=ke_evt_schedule&s=indexed&type=Code)

    (Search results are sorted by recently indexed)

We'll see this __`ke_evt_schedule`__ code from __AliOS Things__ (the embedded OS) and __RivieraWaves__ (explained next chapter): [`ke_event.c`](https://github.com/lupyuen/AliOS-Things/blob/master/platform/mcu/bk7231u/beken/ip/ke/ke_event.c#L203-L231)

```c
//  Event scheduler entry point. This primitive has to be 
//  called in the background loop in order to execute the 
//  event handlers for the event that are set.
void ke_evt_schedule(void) {
  uint32_t field, event;
  field = ke_env.evt_field;
  while (field) { // Compiler is assumed to optimize with loop inversion

    // Find highest priority event set
    event = co_clz(field);

    // Sanity check
    ASSERT_ERR((event < KE_EVT_MAX) && ke_evt_hdlr[event].func);

    // Execute corresponding handler
    (ke_evt_hdlr[event].func)(ke_evt_hdlr[event].param);

    // Update the volatile value
    field = ke_env.evt_field;
  }
}
```

Compare this with our [__decompiled version of `ke_evt_schedule`__](https://github.com/lupyuen/bl602nutcracker1/blob/main/bl602_demo_wifi.c#L28721-L28737)... It's a __perfect match__!

Right down to the __Assertion Check__!

```text
(event < KE_EVT_MAX) && ke_evt_hdlr[event].func
```

Since the two versions of `ke_evt_schedule` are functionally identical, let's read the __AliOS / RivieraWaves version of `ke_evt_schedule`__.

![ke_evt_schedule from AliOS / RivieraWaves](https://lupyuen.github.io/images/wifi-task3.png)

We see that __`ke_evt_schedule`__ handles WiFi Kernel Events by calling the __Event Handlers in `ke_evt_hdlr`__.

Here are the __`ke_evt_hdlr`__ Event Handlers from the AliOS / RivieraWaves code: [`ke_event.c`](https://github.com/lupyuen/AliOS-Things/blob/master/platform/mcu/bk7231u/beken/ip/ke/ke_event.c#L78-L138)

```c
//  Event Handlers called by ke_evt_schedule
static const struct ke_evt_tag ke_evt_hdlr[32] = {
  {&rwnxl_reset_evt,    0},      // [KE_EVT_RESET]	
  {&ke_timer_schedule,  0},      // [KE_EVT_KE_TIMER]   
  {&txl_payload_handle, AC_VO},  // [KE_EVT_IPC_EMB_TXDESC_AC3]

  //  This Event Handler looks interesting                                                      
  {&txl_payload_handle, AC_VI},  // [KE_EVT_IPC_EMB_TXDESC_AC2]
  {&txl_payload_handle, AC_BE},  // [KE_EVT_IPC_EMB_TXDESC_AC1]
  {&txl_payload_handle, AC_BK},  // [KE_EVT_IPC_EMB_TXDESC_AC0]	

  {&ke_task_schedule,   0},      // [KE_EVT_KE_MESSAGE]
  {&mm_hw_idle_evt,     0},      // [KE_EVT_HW_IDLE]
  ...
```

![txl_payload_handle Event Handler](https://lupyuen.github.io/images/wifi-task4.png)

__`txl_payload_handle`__ is the Event Handler that handles the __transmission of WiFi Payloads__.

Let's look inside and learn how it transmits WiFi Packets.

![txl_payload_handle doesn't do much](https://lupyuen.github.io/images/wifi-task5.png)

## Handle Transmit Payload

_What is `txl_payload_handle`?_

Thanks to the AliOS / RivieraWaves source code, we have a meaningful description of the __`txl_payload_handle`__ function: [`txl_cntrl.h`](https://github.com/lupyuen/AliOS-Things/blob/master/platform/mcu/bk7231u/beken/ip/lmac/src/tx/txl/txl_cntrl.h#L377-L386)

```c
//  Perform operations on payloads that have been 
//  transfered from host memory. This primitive is 
//  called by the interrupt controller ISR. It 
//  performs LLC translation and MIC computing if required.
//  LLC = Logical Link Control, MIC = Message Integrity Code
void txl_payload_handle(int access_category);
```

This suggests that `txl_payload_handle` is called to __transmit WiFi Packets__... After the packet payload has been copied from BL602 to the Radio Hardware. (Via the __Shared RAM Buffer__)

Searching our decompiled code for __`txl_payload_handle`__ shows this: [`bl602_demo_wifi.c`](https://github.com/lupyuen/bl602nutcracker1/blob/main/bl602_demo_wifi.c#L20205-L20216)

```c
//  Handle transmit payload
void txl_payload_handle(void) {
  while ((_DAT_44a00024 & 0x1f) != 0) {
    int iVar1 = __clzsi2(_DAT_44a00024 & 0x1f);

    //  Write to WiFi Register at 0x44A0 0020
    _DAT_44a00020 = 1 << (0x1fU - iVar1 & 0x1f);
  }
}
```

It doesn't seem to do much payload processing.  But it writes to the undocumented __WiFi Register at `0x44A0 0020`__.

Which will trigger the __LMAC Firmware to transmit__ the WiFi Packet perhaps?

But hold up! We have something that might explain what's inside `txl_payload_handle`...

![txl_payload_handle_backup](https://lupyuen.github.io/images/wifi-task6.png)

## Another Transmit Payload

Right after `txl_payload_handle` in our decompiled code, there's a function __`txl_payload_handle_backup`__.

Based on the name, `txl_payload_handle_backup` is probably another function that handles payload transmission.

Here are the highlights of the decompiled `txl_payload_handle_backup` function: [`bl602_demo_wifi.c`](https://github.com/lupyuen/bl602nutcracker1/blob/main/bl602_demo_wifi.c#L20222-L20398)

```c
//  Another transmit payload handler.
//  Probably works the same way as txl_payload_handle,
//  but runs on BL602 instead of LMAC Firmware.
void txl_payload_handle_backup(void) {
  ...
  //  Iterate through a list of packet buffers (?)
  while (ptVar4 = ptVar10->list[0].first, ptVar4 == (txl_buffer_tag *)0x0) {
LAB_230059f6:
    uVar3 = uVar3 + 1;
    ptVar10 = (txl_buffer_env_tag *)&ptVar10->buf_idx[0].free_size;
    ptVar11 = (txl_cntrl_env_tag *)(ptVar11->txlist + 1);
  }
```

__`txl_payload_handle_backup`__ starts by iterating through a list of packet buffers to be transmitted. (Probably)

Then it calls some __RXU, TXL and TXU functions from RivieraWaves__...

```c
  //  Loop (until when?)
  do {
    //  Call some RXU, TXL and TXU functions
    rxu_cntrl_monitor_pm((mac_addr *)&ptVar4[1].lenheader);
    ...
    txl_machdr_format((uint32_t)(ptVar4 + 1));
    ...
    txu_cntrl_tkip_mic_append(txdesc,(uint8_t)uVar2);
```

(More about RivieraWaves in the next chapter)

Next we write to some undocumented __WiFi Registers: `0x44B0 8180`, `0x44B0 8198`, `0x44B0 81A4` and `0x44B0 81A8`__ ...

```c
    //  Write to WiFi Registers
    _DAT_44b08180 = 0x800;
    _DAT_44b081a4 = ptVar9;
    ...
    _DAT_44b08180 = 0x1000;
    _DAT_44b081a8 = ptVar9;
    ...
    _DAT_44b08180 = 0x100;
    _DAT_44b08198 = ptVar9;
```

[(They don't appear in this list either)](https://github.com/lupyuen/bl_iot_sdk/blob/master/components/bl602/bl602_wifidrv/bl60x_wifi_driver/reg_access.h)

The function performs some __Assertion Checks__. 

The Assertion Failure Messages may be helpful for deciphering the decompiled code...

```c
    //  Assertion Checks
    line = 0x23c;
    condition = "blmac_tx_ac_2_state_getf() != 2";
    ...
    line = 0x236;
    condition = "blmac_tx_ac_3_state_getf() != 2";
    ...
    line = 0x22f;
    condition = "blmac_tx_bcn_state_getf() != 2";
    ...
    line = 0x242;
    condition = "blmac_tx_ac_1_state_getf() != 2";
    ...
    line = 0x248;
    condition = "blmac_tx_ac_0_state_getf() != 2";
    ...
    assert_rec(condition, "module", line);
```

The function ends by __setting a timer__...

```c
    //  Set a timer
    blmac_abs_timer_set(uVar6, (uint32_t)(puVar8 + _DAT_44b00120));

    //  Continue looping
  } while( true );
}
```

_Is the original source code for `txl_payload_handle_backup` really so long?_

Likely not. The C Compiler optimises the firmware code by inlining some functions.

When we decompile the firmware, the inlined code appears embedded inside the calling functions. 

(That's why we see so much repetition in the decompiled code)

Let's talk about RivieraWaves...

![RivieraWaves in AliOS](https://lupyuen.github.io/images/wifi-schedule3.png)

# CEVA RivieraWaves

When we searched GitHub for the __WiFi Event Scheduler `ke_evt_schedule`__, we found this source code...

-   [__mclown/AliOS-Things__](https://github.com/mclown/AliOS-Things/tree/master/platform/mcu/bk7231u/beken/ip)

[(We'll use this fork)](https://github.com/lupyuen/AliOS-Things/tree/master/platform/mcu/bk7231u/beken/ip)

This appears to be the source code for the __AliOS Things embedded OS__.

[(Ported to the Beken BK7231U WiFi SoC)](http://www.bekencorp.com/en/goods/detail/cid/13.html)

But at the top of the [source file](https://github.com/lupyuen/AliOS-Things/blob/master/platform/mcu/bk7231u/beken/ip/ke/ke_event.c) we see...

```text
Copyright (C) RivieraWaves 2011-2016
```

This means that the WiFi source code originates from __CEVA RivieraWaves, not AliOS__!

![CEVA RivieraWaves](https://lupyuen.github.io/images/wifi-lmac3.jpg)

[(Source)](https://www.ceva-dsp.com/product/rivierawaves-wi-fi-platforms/)

_What is CEVA RivieraWaves?_

RivieraWaves is the __Software / Firmware that implements the 802.11 Wireless Protocol__ on WiFi SoCs (like BL602).

On BL602 there are two layers of RivieraWaves Firmware...

1.  __Upper Medium Access Control (UMAC)__

    Runs on the __BL602 RISC-V CPU.__

    Some of the code we've seen earlier comes from UMAC.
    
    (Like the Kernel Event Scheduler)

1.  __Lower Medium Access Control (LMAC)__

    Runs inside the __BL602 Radio Hardware.__

    We don't have any LMAC code to study since it's hidden inside the Radio Hardware.

    (But we can see the LMAC Interfaces exposed by the WiFi Registers)

[More about WiFi Medium Access Control](https://www.controleng.com/articles/wi-fi-and-the-osi-model/)

[More about RivieraWaves](https://www.ceva-dsp.com/product/rivierawaves-wi-fi-platforms/)

_Is RivieraWaves used elsewhere?_

Yes, RivieraWaves is used in many popular WiFi SoCs.

This article hints at the WiFi SoCs that might be using RivieraWaves (or similar code by CEVA)...

> ![Customers of RivieraWaves](https://lupyuen.github.io/images/wifi-ceva.png)

[(Source)](https://csimarket.com/stocks/markets_glance.php?code=CEVA#:~:text=Included%20among%20our%20licensees%20are,%2C%20RDA%2C%20Renesas%2C%20Rockchip%2C)

## Upper Medium Access Control

Recall that __UMAC (Upper Medium Access Control)__ is the RivieraWaves code that runs on the __BL602 RISC-V CPU__.

When we match the decompiled BL602 WiFi Firmware with the AliOS / RivieraWaves code, we discover the Source Code for the __UMAC Modules (and Common Modules) that are used in BL602__...

1.  [CO Module](https://github.com/lupyuen/AliOS-Things/blob/master/platform/mcu/bk7231u/beken/ip/common) (Common)
1.  [KE Module](https://github.com/lupyuen/AliOS-Things/blob/master/platform/mcu/bk7231u/beken/ip/ke) (Kernel)
1.  [ME Module](https://github.com/lupyuen/AliOS-Things/blob/master/platform/mcu/bk7231u/beken/ip/umac/src/me) (Message?)
1.  [RC Module](https://github.com/lupyuen/AliOS-Things/blob/master/platform/mcu/bk7231u/beken/ip/umac/src/rc) (Rate Control)
1.  [RXU Module](https://github.com/lupyuen/AliOS-Things/blob/master/platform/mcu/bk7231u/beken/ip/umac/src/rxu) (Receive UMAC)
1.  [SCANU Module](https://github.com/lupyuen/AliOS-Things/blob/master/platform/mcu/bk7231u/beken/ip/umac/src/scanu) (Scan SSID UMAC)
1.  [SM Module](https://github.com/lupyuen/AliOS-Things/blob/master/platform/mcu/bk7231u/beken/ip/umac/src/sm) (State Machine)
1.  [TXU Module](https://github.com/lupyuen/AliOS-Things/blob/master/platform/mcu/bk7231u/beken/ip/umac/src/txu) (Transmit UMAC)

These modules are __mostly identical across BL602 and AliOS / RivieraWaves__. (Except RXU, which looks different)

(More about UMAC Matching when we discuss Quantitative Analysis)

![Compare BL602 with RivieraWaves](https://lupyuen.github.io/images/wifi-rivierawaves.png)

## Lower Medium Access Control

Remember that __LMAC (Lower Medium Access Control)__ is the RivieraWaves code that runs inside the __BL602 Radio Hardware__.

By matching the decompiled BL602 WiFi Firmware with the AliOS / RivieraWaves code, we discover the __LMAC Interfaces that are exposed by the BL602 Radio Hardware__...

1.  APM Interface (Missing from AliOS)
1.  CFG Interface (Missing from AliOS)
1.  [CHAN Interface](https://github.com/lupyuen/AliOS-Things/blob/master/platform/mcu/bk7231u/beken/ip/lmac/src/chan) (MAC Channel Mgmt)
1.  [HAL Interface](https://github.com/lupyuen/AliOS-Things/blob/master/platform/mcu/bk7231u/beken/ip/lmac/src/hal) (Hardware Abstraction Layer)
1.  [MM Interface](https://github.com/lupyuen/AliOS-Things/blob/master/platform/mcu/bk7231u/beken/ip/lmac/src/mm) (MAC Mgmt)
1.  [RXL Interface](https://github.com/lupyuen/AliOS-Things/blob/master/platform/mcu/bk7231u/beken/ip/lmac/src/rx/rxl) (Receive LMAC)
1.  [STA Interface](https://github.com/lupyuen/AliOS-Things/blob/master/platform/mcu/bk7231u/beken/ip/lmac/src/sta) (Station Mgmt)
1.  [TXL Interface](https://github.com/lupyuen/AliOS-Things/blob/master/platform/mcu/bk7231u/beken/ip/lmac/src/tx/txl) (Transmit LMAC)

The LMAC Interfaces linked above are for reference only... The __BL602 implementation of LMAC is very different__ from the Beken BK7231U implementation above.

These LMAC Modules seem to be __mostly identical across BL602 and AliOS / RivieraWaves__...

1.  [PS Module](https://github.com/lupyuen/AliOS-Things/blob/master/platform/mcu/bk7231u/beken/ip/lmac/src/ps) (Power Save)
1.  [SCAN Module](https://github.com/lupyuen/AliOS-Things/blob/master/platform/mcu/bk7231u/beken/ip/lmac/src/scan) (Scan SSID)
1.  [TD Module](https://github.com/lupyuen/AliOS-Things/blob/master/platform/mcu/bk7231u/beken/ip/lmac/src/td) (Traffic Detection)
1.  [VIF Module](https://github.com/lupyuen/AliOS-Things/blob/master/platform/mcu/bk7231u/beken/ip/lmac/src/vif) (Virtual Interface)

(More about LMAC Matching when we discuss Quantitative Analysis)

![WiFi Supplicant: Rockchip RK3399 vs BL602](https://lupyuen.github.io/images/wifi-supplicant2.jpg)

# WiFi Supplicant

_What's the WiFi Supplicant?_

[__WiFi Supplicant__](https://en.wikipedia.org/wiki/Wireless_supplicant) is the code that handles WiFi Authentication.

[(Like for WPA and WPA2)](https://en.wikipedia.org/wiki/Wpa_supplicant)

_So WiFi Supplicant comes from RivieraWaves right?_

Nope. Based on the decompiled code, __BL602 implements its own WiFi Supplicant__ with functions like `supplicantInit`, `allocSupplicantData` and `keyMgmtGetKeySize`. [(See this)](https://lupyuen.github.io/images/wifi-supplicant.png)

_Maybe the WiFi Supplicant code came from another project?_

When we [__search GitHub for the function names__](https://github.com/search?o=desc&q=supplicantInit+allocSupplicantData+keyMgmtGetKeySize&s=indexed&type=Code), we discover this matching source code...

-   [__karthirockz/rk3399-kernel__](https://github.com/karthirockz/rk3399-kernel/tree/main/drivers/net/wireless/rockchip_wlan/mvl88w8977/mlan/esa)

[(We'll use this fork)](https://github.com/lupyuen/rk3399-kernel/tree/main/drivers/net/wireless/rockchip_wlan/mvl88w8977/mlan/esa)

That's actually the __WiFi Supplicant for Rockchip RK3399__, based on Linux!

_Are they really the same code?_

We compared the decompiled BL602 WiFi Supplicant code with the Rockchip RK3399 source code... They are __nearly 100% identical__! [(See this)](https://lupyuen.github.io/images/wifi-supplicant2.png)

This is awesome because we have just uncovered the secret origin of (roughly) __2,500 Lines of Code__ from the decompiled BL602 firmware!

(This data comes from the __Quantitative Analysis__, which we'll discuss in a while)

![WiFi Supplicant: Lines of code](https://lupyuen.github.io/images/wifi-rockchip.jpg)

# WiFi Physical Layer

_What's the WiFi Physical Layer?_

__WiFi Physical layer__ is the wireless protocol that controls the airwaves and dictates how WiFi Packets should be transmitted and received.

(It operates underneath the Medium Access Control Layer)

[More about WiFi Physical Layer](https://www.controleng.com/articles/wi-fi-and-the-osi-model/)

_Lemme guess... BL602 doesn't use RivieraWaves for the WiFi Physical Layer?_

Nope we don't think the BL602 Physical Layer comes from RivieraWaves.

The origin of __BL602's Physical Layer is a little murky__...

_How so?_

Here's a snippet of __BL602 Physical Layer__ from the decompiled code: [`bl602_demo_wifi.c`](https://github.com/lupyuen/bl602nutcracker1/blob/main/bl602_demo_wifi.c#L33527-L33614)

```c
//  From BL602 Decompiled Code: Init Physical Layer
void phy_init(phy_cfg_tag *config) {
  mdm_reset();
  ...
  mdm_txcbwmax_setf((byte)(_DAT_44c00000 >> 0x18) & 3);
  _Var2 = phy_vht_supported();
  agc_config();
  ...
  // Init transmitter rate power control
  trpc_init();

  // Init phy adaptive features
  pa_init();

  phy_tcal_reset();
  phy_tcal_start();
}
```

When we [__search GitHub for `phy_init` and `phy_hw_set_channel`__](https://github.com/search?q=phy_init+phy_hw_set_channel&type=code) (another BL602 function), we get one meaningful result...

-   [__jixinintelligence/bl602-604__](https://github.com/jixinintelligence/bl602-604)

[(We'll use this fork)](https://github.com/lupyuen/bl602-604)

Which implements `phy_init` like so: [`phy_bl602.c`](https://github.com/lupyuen/bl602-604/blob/master/components/bl602/bl602_wifi/plf/refip/src/driver/phy/bl602_phy_rf/phy_bl602.c#L474-L492) ...

```c
//  From GitHub Search: Init Physical Layer
void phy_init(const struct phy_cfg_tag *config) {
  const struct phy_bl602_cfg_tag *cfg = (const struct phy_bl602_cfg_tag *)&config->parameters;
  phy_hw_init(cfg);
  phy_env->cfg               = *cfg;
  phy_env->band              = PHY_BAND_2G4;
  phy_env->chnl_type         = PHY_CHNL_BW_OTHER;
  phy_env->chnl_prim20_freq  = PHY_UNUSED;
  phy_env->chnl_center1_freq = PHY_UNUSED;
  phy_env->chnl_center2_freq = PHY_UNUSED;

  // Init transmitter rate power control
  trpc_init();

  // Init phy adaptive features
  pa_init();
}
```

![BL602 Physical Layer](https://lupyuen.github.io/images/wifi-phy.png)

Comparing the BL602 decompiled code with the GitHub Search Result... The __BL602 code seems to be doing a lot more__?

(Where are the calls to `mdm_reset`, `phy_tcal_reset` and `phy_tcal_start`?)

Thus __we don't have a 100% match__ for the BL602 Physical Layer. (Maybe 50%)

Nonetheless this is a helpful discovery for our Reverse Engineering!

![Extracting the function names from the decompiled firmware](https://lupyuen.github.io/images/wifi-quantify.png)

# Quantitative Analysis

_How many Lines of Decompiled Code do we actually need to decipher?_

_How much of the BL602 WiFi Source Code is already available elsewhere?_

To answer these questions, let's do a __Quantitative Analysis of the Decompiled BL602 Firmware Code__.

(Yep that's the fancy term for __data crunching with a spreadsheet__)

We shall...

1.  __Extract the function names__ from the decompiled BL602 WiFi Demo Firmware

1.  __Load the decompiled function names__ into a spreadsheet for analysis

1.  __Classify the decompiled function names__ by module

1.  __Match the decompiled function code__ with the source code we've discovered through GitHub Search

1.  __Count the number of lines of decompiled code__ that don't have any matching source code

## Extract the decompiled functions

Our __BL602 WiFi Demo Firmware [`bl602_demo_wifi`](https://github.com/lupyuen/bl_iot_sdk/blob/master/customer_app/bl602_demo_wifi)__ has been decompiled into one giant C file...

-   [__Decompiled WiFi Demo Firmware `bl602_demo_wifi.c`__](https://github.com/lupyuen/bl602nutcracker1/blob/main/bl602_demo_wifi.c)

We run this command to __extract the Function Names__ and their Line Numbers (for counting the Lines of Code)...

```bash
# Extract the function names (and line numbers)
# from the decompiled firmware. The line must
# begin with an underscore or a letter,
# without indentation.
grep --line-number \
    "^[_a-zA-Z]" \
    bl602_demo_wifi.c \
    | grep -v LAB_ \
    >bl602_demo_wifi.txt
```

This produces [__`bl602_demo_wifi.txt`__](https://github.com/lupyuen/bl602nutcracker1/blob/main/bl602_demo_wifi.txt), a long list of Decompiled Function Names and their Line Numbers. [(Here's a snippet)](https://lupyuen.github.io/images/wifi-quantify.png)

(Plus Function Parameters and Type Definitions... We'll scrub them away soon)

_But this list includes EVERYTHING... Including the non-WiFi functions no?_

Yes. But it's fun to comb through __Every Single Function in the Decompiled Firmware__ (128,000 Lines of Code)... Just to see what makes it tick.

_Why not just decompile and analyse the BL602 WiFi Library: [`libbl602_wifi.a`](https://github.com/pine64/bl602-re/tree/master/blobs) ?_

The [__BL602 WiFi Library `libbl602_wifi.a`__](https://github.com/pine64/bl602-re/tree/master/blobs) might contain some extra WiFi Functions that won't get linked into the WiFi Firmware.

Hence we're decompiling and analysing the __actual WiFi Functions called by the WiFi Firmware__.

(BTW: Our counting of Lines of Code will include Blank Lines and Comment Lines)

![Loading decompiled function names into a spreadsheet](https://lupyuen.github.io/images/wifi-quantify2.png)

## Load functions into spreadsheet

We load [__`bl602_demo_wifi.txt`__](https://github.com/lupyuen/bl602nutcracker1/blob/main/bl602_demo_wifi.txt) (the list of Decompiled Function Names and Line Numbers) into a spreadsheet for analysis. (See pic above)

Here's our __spreadsheet for Quantitative Analysis__ in various formats...

-   [Google Sheets](https://docs.google.com/spreadsheets/d/1C_XmkH-ZSXz9-V2HsYBv7K1KRx3RF3-zsoJRLh1GwxI/edit?usp=sharing)

-   [LibreOffice / OpenOffice Format](https://github.com/lupyuen/bl602nutcracker1/blob/main/bl602_demo_wifi.ods)

-   [Excel Format](https://github.com/lupyuen/bl602nutcracker1/blob/main/bl602_demo_wifi.xlsx)

-   [CSV Format (raw data only)](https://github.com/lupyuen/bl602nutcracker1/blob/main/bl602_demo_wifi.csv)

We scrub the data to remove the Type Definitions, Function Return Types and the Function Parameters.

Based on the Line Numbers, we compute the __Lines of Code__ for each function (including Blank Lines and Comment Lines).

And apply __Conditional Formatting__ to highlight the Decompiled Functions with the most Lines of Code. (Which are also the __Most Complex Functions__)

These are the functions we should pay more attention during the analysis.

![Classify the decompiled functions](https://lupyuen.github.io/images/wifi-quantify3.png)

## Classify the decompiled functions

Next we __classify each Decompiled Function by Module__.

The pic above shows that we've classified the "`rxl_`" functions as "`???RivieraWaves RXL`". (RXL means Receive LMAC)

We use "`???`" to mark the Modules that we couldn't find any source code.

_Doing this for all 3,000 Decompiled Functions sounds tedious...?_

Fortunately the Decompiled Functions belonging to a Module are __clustered together__. So it's easy to copy and fill the Module Name for a batch of functions.

Remember our red highlighting for Complex Functions? It's OK to __skip the classification of the Less Complex Functions__ (if we're not sure how to classify them).

In our spreadsheet we've __classified over 97,000 Decompiled Lines of Code__. That's __86%__ of all Decompiled Lines of Code. Good enough for our analysis!

![Matching the decompiled function code](https://lupyuen.github.io/images/wifi-quantify5.png)

## Match the decompiled functions

Remember the source code we've discovered earlier through __GitHub Search__?

(For RivieraWaves, WiFi Supplicant and Physical Layer)

Now we dive into the Discovered Source Code and see __how closely they match the Decompiled Functions__.

_How do we record our findings?_

Inside our spreadsheet is a column that records the __Source Code URL__ (from GitHub Search) that we've matched with our Decompiled Functions. (See pic above)

We've also added a comment that says __how closely they match__. ("BL602 version is different")

If the Discovered Source Code doesn't match the Decompiled Function, we __flag the Module Name__ with "`???`".

_Do we need to match every Decompiled Function?_

To simplify the matching, we picked one or two of the __Most Complex Functions from each Module__.

(Yep the red highlighting really helps!)

Thus our matching is not 100% thorough and accurate... But it's reasonably accurate.

![Counting the decompiled lines of code in BL602 WiFi Firmware](https://lupyuen.github.io/images/wifi-title.jpg)

## Count the lines of code

Finally we add a __Pivot Table to count the Lines of Code__ that are matched (or unmatched) with GitHub Search.

In the second tab of our spreadsheet, we see the Pivot Table that summarises the __results of our Quantitative Analysis__...

1.  Lines of Code to be __Reverse Engineered: 10,500__

    Not found on GitHub Search: __LMAC Interface__, and some parts of WiFi Supplicant.

    ![Decompiled lines of code to be reverse engineered](https://lupyuen.github.io/images/wifi-loc1.png)

1.  Lines of Code for __Partial Reverse Engineering: 3,500__

    We found partial matches for the __Physical Layer__ on GitHub Search.

    ![Decompiled lines of code for partial reverse enginnering](https://lupyuen.github.io/images/wifi-loc2.png)

    (We'll talk about BL602 HAL and Standard Driver in the next chapter)

1.  Lines of Code __Already Found Elsewhere: 11,300__ (Wow!)

    Found on GitHub Search: __UMAC and most of WiFi Supplicant__

    ![Decompiled lines of code already found elsewhere](https://lupyuen.github.io/images/wifi-loc3.png)

1.  We also have __7,500__ Lines of Code from parts of the __BL602 WiFi Driver__ whose source code may be found in the __BL602 IoT SDK__. [(See this)](https://github.com/lupyuen/bl_iot_sdk/tree/master/components/bl602/bl602_wifidrv/bl60x_wifi_driver)

    Includes the __WiFi Manager__ that we've seen earlier.

    ![Lines of code for BL602 WiFi Driver](https://lupyuen.github.io/images/wifi-loc8.png)

Conclusion: We have __plenty of source code__ to guide us for the __Reverse Engineering of BL602 WiFi__!

# Other Modules

WiFi Functions make up __29%__ of the total Lines of Code in our Decompiled WiFi Firmware.

_What's inside the other 71% of the Decompiled Code?_

Let's run through the __Non-WiFi Functions in our Decompiled Firmware__...

(Complex modules are highlighted in red)

![Decompiled lines of code](https://lupyuen.github.io/images/wifi-loc7.png)

-   [__AliOS__](https://github.com/lupyuen/bl_iot_sdk/tree/master/components/fs): Embedded framework for multitasking and device drivers

-   [__AWS IoT, AWS MQTT__](https://github.com/lupyuen/bl_iot_sdk/tree/master/components/3rdparty/aws-iot): Demo Firmware talks to AWS Cloud for IoT and MQTT (Message Queue) Services

-   [__BL602 Hardware Abstraction Layer (HAL)__](https://github.com/lupyuen/bl_iot_sdk/tree/master/components/hal_drv/bl602_hal): Functions for Bootloader, DMA, GPIO, Flash Memory, Interrupts, Real Time Clock, Security (Encryption), UART, ...

-   [__BL602 Standard Driver__](https://github.com/lupyuen/bl_iot_sdk/tree/master/components/bl602/bl602_std/bl602_std/StdDriver): Called by the BL602 Hardware Abstraction Layer to access the BL602 Hardware Registers

-   [__C Standard Library__](https://github.com/lupyuen/bl_iot_sdk/tree/master/toolchain/riscv/Linux/lib/gcc/riscv64-unknown-elf/8.3.0/rv32imf/ilp32f): Because our firmware is compiled with the GCC Compiler

-   [__EasyFlash__](https://github.com/lupyuen/bl_iot_sdk/blob/master/components/stage/easyflash4/inc/easyflash.h): Embedded database

-   [__FreeRTOS__](https://github.com/lupyuen/bl_iot_sdk/tree/master/components/bl602/freertos_riscv): Embedded OS that runs underneath AliOS

-   [__Lightweight IP (LWIP)__](https://github.com/lupyuen/bl_iot_sdk/tree/master/components/network): For IP, UDP, TCP and HTTP Networking

-   [__Mbed TLS__](https://github.com/lupyuen/bl_iot_sdk/tree/master/components/security/mbedtls): Implements Transport Layer Security, needed by AWS IoT and MQTT

Source code is available for most of the Non-WiFi Functions.

(Just click the links above)

![GitHub Code Search](https://lupyuen.github.io/images/wifi-schedule.png)

# GitHub Search Is Our Best Friend!

Today we've learnt a valuable lesson... __GitHub Search is our Best Friend for Reverse Engineering__!

Here's what we have discovered through GitHub Search...

1.  Source Code for __UMAC and LMAC__

    [GitHub Code Search for `ke_evt_schedule`](https://github.com/search?o=desc&q=ke_evt_schedule&s=indexed&type=Code)

1.  Source Code for __WiFi Supplicant__

    [GitHub Code Search for `supplicantInit`, `allocSupplicantData` and `keyMgmtGetKeySize`](https://github.com/search?o=desc&q=supplicantInit+allocSupplicantData+keyMgmtGetKeySize&s=indexed&type=Code)

1.  Source Code for __Physical Layer__

    [GitHub Code Search for `phy_init` and `phy_hw_set_channel`](https://github.com/search?q=phy_init+phy_hw_set_channel&type=code)

Remember to check GitHub Search when doing any Reverse Engineering! 👍 

# What's Next

This has been a thrilling journey... Many Thanks to the contributors of the [__Pine64 BL602 Reverse Engineering Project__](https://github.com/pine64/bl602-re) for inspiring this article!

Today we've done some Reverse Engineering for the purpose of __Education__... Just to understand how BL602 sends and receives WiFi Packets.

And we now understand how to sniff around the Decompiled WiFi Firmware to __uncover every WiFi Hardware Register__.

I hope someone will continue the Reverse Engineering work... Maybe create an __Open Source WiFi Driver for BL602__!

[(Perhaps we should revive the __Pine64 BL602 Nutcracker Project__)](https://github.com/pine64/bl602-re)

In the next article we shall check out the __BL706 Audio Video Board__...

-   [__"RISC-V BL706 Audio Video Board"__](https://lupyuen.github.io/articles/bl706)

Stay tuned for more articles on BL602, BL604 and BL706!

-   [Sponsor me a coffee](https://lupyuen.github.io/articles/sponsor)

-   [Discuss this article on Reddit](https://www.reddit.com/r/RISCV/comments/ofj34x/reverse_engineering_wifi_on_riscv_bl602/?utm_source=share&utm_medium=web2x&context=3)

-   [Read "The RISC-V BL602 Book"](https://lupyuen.github.io/articles/book)

-   [Check out my articles](https://lupyuen.github.io)

-   [RSS Feed](https://lupyuen.github.io/rss.xml)

_Got a question, comment or suggestion? Create an Issue or submit a Pull Request here..._

[`lupyuen.github.io/src/wifi.md`](https://github.com/lupyuen/lupyuen.github.io/blob/master/src/wifi.md)

![BL706 Audio Video Board](https://lupyuen.github.io/images/bl706.jpg)

_BL706 Audio Video Board_

# Notes

1.  This article is the expanded version of [this Twitter Thread](https://twitter.com/MisterTechBlog/status/1407971263088193540)

1.  According to [madushan1000 on Twitter](https://twitter.com/madushan1000/status/1409392882612637696), the __BL602 WiFi RTL__ may be found here...

    -   [fengmaoqiao/my_logic_code](https://github.com/fengmaoqiao/my_logic_code)

    -   [fengmaoqiao/workplace](https://github.com/fengmaoqiao/workplace)

    -   [More tips on BL602 WiFi and Bluetooth LE](https://twitter.com/madushan1000/status/1412694106816585728?s=19)

1.  More about __BL602 RF IP and Hardware Registers__:

    -   [Hardware Notes: RF IP](https://github.com/pine64/bl602-docs/tree/main/hardware_notes#rf-ip)
    
    -   [Hardware Notes: MDM Registers](https://github.com/pine64/bl602-docs/blob/main/hardware_notes/registers/phy/mdm.md)

    -   [Hardware Notes: AGC Registers](https://github.com/pine64/bl602-docs/blob/main/hardware_notes/registers/phy/agc.md)

1.  There's an interesting discussion about the __licensing of the WiFi Supplicant__, which looks identical to the Linux version on Rockchip RK3399...

    ![Licensing of the WiFi Supplicant](https://lupyuen.github.io/images/wifi-licence.png)

    [(Source)](https://news.ycombinator.com/item?id=27763031)

1.  ESP32 uses CEVA's Bluetooth IP but not CEVA's WiFi IP, according to [SpritesMods on Twitter](https://twitter.com/SpritesMods/status/1412308410226286598?s=19)
