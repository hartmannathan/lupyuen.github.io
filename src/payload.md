# CBOR Payload Formatter for The Things Network

📝 _18 Oct 2021_

Suppose we have an __IoT Sensor Device__ (like [__PineDio Stack BL604__](https://lupyuen.github.io/articles/pinedio2)) connected to __The Things Network__ (via LoRaWAN)...

-   [__"The Things Network on PineDio Stack BL604 RISC-V Board"__](https://lupyuen.github.io/articles/ttn)

And our device __transmits Sensor Data__ to The Things Network in __CBOR Format__ (because it requires fewer bytes than JSON)...

-   [__"Encode Sensor Data with CBOR on BL602"__](https://lupyuen.github.io/articles/cbor)

> ![IoT Sensor Device transmits CBOR Sensor Data to The Things Network](https://lupyuen.github.io/images/grafana-flow3.jpg)

_How shall we process the CBOR Sensor Data transmitted by our device?_

We could let __each Application fetch and decode__ the CBOR Sensor Data from The Things Network...

![Each Application fetches and decodes the CBOR Sensor Data from The Things Network](https://lupyuen.github.io/images/payload-flow3.jpg)

Each Application would __decode CBOR into JSON__ before processing, like we've done for Grafana and Roblox...

-   [__"Grafana Data Source for The Things Network"__](https://lupyuen.github.io/articles/grafana)

-   [__"IoT Digital Twin with Roblox and The Things Network"__](https://lupyuen.github.io/articles/roblox)

(Assuming that JSON is supported natively by each Application)

_Erm this solution doesn't scale well if we have many Applications..._

Exactly! For every Application that we add (like Prometheus), we would need to __decode the CBOR Sensor Data into JSON again and again__.

(And some Applications might need code changes to support CBOR)

_What's the right solution then?_

The proper solution is to configure a __Payload Formatter__ at The Things Network that will __decode our CBOR Sensor Data into JSON once__...

![CBOR Payload Formatter for The Things Network](https://lupyuen.github.io/images/payload-title.jpg)

And __distribute the Decoded Sensor Data__ (as JSON) to all Applications.

Read on to learn how we created a __CBOR Payload Formatter__ for The Things Network.

![Payload Formatter](https://lupyuen.github.io/images/payload-formatter.jpg)

[(Source)](https://www.thethingsindustries.com/docs/integrations/payload-formatters/javascript/)

# What's a Payload Formatter?

A __Payload Formatter is JavaScript Code__ that runs on the __servers at The Things Network__ to decode our LoRaWAN Message Payload. (Which contains our CBOR-Encoded Sensor Data)

Inside the Payload Formatter we need to provide a JavaScript Function named __decodeUplink__ that will decode our LoRaWAN Message Payload...

```javascript
//  Decode the Payload in the Uplink Message
function decodeUplink(input) {
  //  `input.fPort` contains the LoRaWAN Port Number (like 2)
  //  `input.bytes` contains the LoRaWAN Message Payload bytes like...
  //  [ 0xa2, 0x61, 0x74, 0x19, 0x12, 0x3d, 0x61, 0x6c, 0x19, 0x0f, 0xa0 ]

  //  TODO: Data and warnings to be returned to The Things Network
  var data     = { "t": 4669, "l": 4000 };  
  var warnings = [];

  //  Return the decoded data and warnings
  return {
    data:     data,
    warnings: warnings
  };
}
```

In the above example we return the __Decoded Data__ as...

```json
{
  "t": 4669,
  "l": 4000
}
```

(Let's pretend that's the Sensor Data for our Temperature Sensor and Light Sensor)

_Can we run any kind of JavaScript in a Payload Formatter?_

Nope, here are the Rules for Squid Game _(oops)_ __Payload Formatter__...

1.  Only [__JavaScript (ECMAScript) 5.1__](https://www.ecma-international.org/ecma-262/5.1/) is supported

    (a.k.a "Plain Old JavaScript")

1.  Which means we don't allow __let__, __const__ and __Arrow Functions__

    (Like `x => {...}`)

1.  __JavaScript Modules__ are not supported

    (Like __require__ and __import__)

1.  __Input / Output__ are not supported

    (__console.log__ will fail!)

[(More about Payload Formatters)](https://www.thethingsindustries.com/docs/integrations/payload-formatters/javascript/)

[(Helium has a similar Payload Decoder)](https://docs.helium.com/use-the-network/console/functions/)

![decodeUplink Function](https://lupyuen.github.io/images/payload-formatter2.jpg)

[(Source)](https://www.thethingsindustries.com/docs/integrations/payload-formatters/javascript/#decode-uplink-example-the-things-node)

# CBOR Payload Formatter

Let's study the JavaScript for our __CBOR Payload Formatter__: [`cbor.js`](https://github.com/lupyuen/cbor-the-things-network/blob/master/cbor.js#L410-L441)

```javascript
//  The Things Network Payload Formatter for CBOR
//  Based on https://github.com/paroga/cbor-js/blob/master/cbor.js
(function(global, undefined) { 
  function decode(...) { ... }
  ...
})(this);
```

The script begins by including the entire contents of this __JavaScript Decoder for CBOR__...

-   [__paroga/cbor-js__](https://github.com/paroga/cbor-js/blob/master/cbor.js)

> ![CBOR Payload Formatter](https://lupyuen.github.io/images/payload-code3.png)

This defines the CBOR Decoder Function __CBOR.decode__, which we'll call in a while.

(Yep, this decoder is all Plain Old JavaScript)

Next we define the __decodeUplink__ function that will be called by The Things Network...

```javascript
//  Decode the CBOR Payload in the Uplink Message
function decodeUplink(input) {
  //  Data and warnings to be returned to The Things Network
  var data     = {};  
  var warnings = [];
```

Soon we shall compose the __Decoded Data and Decoder Warnings__ that will be returned to The Things Network.

__input.bytes__ contains a byte array of CBOR-Encoded Sensor Data...

```javascript
[ 0xa2, 0x61, 0x74, 0x19, 0x12, 0x3d, 0x61, 0x6c, 0x19, 0x0f, 0xa0 ]
```

We convert it to an __ArrayBuffer__...

```javascript
  //  Catch any exceptions and return them as warnings.
  try {
    //  Convert payload bytes to ArrayBuffer.
    //  `input.bytes` contains CBOR bytes like...
    //  [ 0xa2, 0x61, 0x74, 0x19, 0x12, 0x3d, 0x61, 0x6c, 0x19, 0x0f, 0xa0 ]
    var array = new Uint8Array(input.bytes);
    var buf   = array.buffer;
```

And we call our __CBOR Decoder__ to decode the ArrayBuffer...

```javascript
    //  Decode the ArrayBuffer
    data = CBOR.decode(buf);

    //  `data` contains Key-Value Pairs like...
    //  { "l": 4000, "t": 4669 }
```

In case of errors, we catch them and __return as warnings__...

```javascript
  } catch (error) {
    //  Catch any exceptions and return them as warnings.
    //  The Things Network will drop the message if we return errors.
    warnings.push(error);
  }
```

Finally we return the __Decoded Data and Decoder Warnings__ to The Things Network...

```javascript
  //  Return the decoded data and decoder warnings
  return {
    data:     data,
    warnings: warnings
  };
}
```

The __Decoded Data__ will look like this...

```json
{
  "t": 4669,
  "l": 4000
}
```

And that's how we decode CBOR Sensor Data in our Payload Formatter!

![decodeUplink Function](https://lupyuen.github.io/images/payload-code4.png)

# Configure Payload Formatter

Now we __configure The Things Network__ with our Payload Formatter...

1.  Log on to The Things Network Console

1.  Click __Applications → (Your Application) → Payload Formatters → Uplink__

    ![Configure Payload Formatter](https://lupyuen.github.io/images/payload-config2.png)

1.  Set the __Formatter Type__ to __JavaScript__

1.  Copy and paste the contents of [__cbor.js__](https://github.com/lupyuen/cbor-the-things-network/blob/master/cbor.js)
 into the __Formatter Parameter__ box

1.  Click __Save Changes__

Let's test our CBOR Payload Formatter!

![PineDio Stack BL604 RISC-V Board (foreground) talking to The Things Network via RAKWireless RAK7248 LoRaWAN Gateway (background)](https://lupyuen.github.io/images/ttn-title.jpg)

# Run Payload Formatter

To __test our CBOR Payload Formatter__, we need a LoRaWAN Device that will transmit CBOR Payloads to The Things Network.

Today we shall use [__PineDio Stack BL604__](https://lupyuen.github.io/articles/pinedio2) (pic above)

1.  Follow the instructions below to __build, flash and run__ the LoRaWAN Firmware for PineDio Stack...

    [__"Build and Run LoRaWAN Firmware"__](https://lupyuen.github.io/articles/tsen#appendix-build-and-run-lorawan-firmware)

1.  Enter the command to __transmit Temperature Sensor Data__ to The Things Network, encoded with CBOR...

    [__"Run the LoRaWAN Firmware"__](https://lupyuen.github.io/articles/tsen#run-the-lorawan-firmware)

1.  Log on to The Things Network Console

1.  Click __Applications → (Your Application) → Live Data__

1.  Our __Decoded Sensor Data__ should appear in the Live Data Table...

    ```json
    { "l": 4000, "t": 4669 }
    ```

    ![Decoded Sensor Data in the Live Data Table](https://lupyuen.github.io/images/payload-ttn3.png)

1.  Click on a message in the Live Data Table. 

    We should see the __decoded_payload__ field with our Decoded Sensor Data...

    ```json
    "decoded_payload": {
      "l": 4000,
      "t": 4656
    }
    ```

    (Our decoder warnings will also appear here)

1.  __decoded_payload__ will be visible to Applications that are connected to The Things Network via...

    -   [__MQTT__](https://github.com/lupyuen/cbor-the-things-network#mqtt-log)

    -   [__Storage API__](https://github.com/lupyuen/cbor-the-things-network#storage-log) (over HTTPS)

    -   And [__other supported protocols__](https://www.thethingsnetwork.org/docs/applications-and-integrations/)

    ![decoded_payload Field](https://lupyuen.github.io/images/payload-ttn4.png)

    [(Source)](https://github.com/lupyuen/cbor-the-things-network#mqtt-log)

Congratulations, we have successfully decoded our CBOR Message Payload in The Things Network... Ready to be consumed by multiple Applications!

(Like __Prometheus__, the open source Time Series Database for IoT Sensor Data)

![Storing The Things Network Sensor Data with Prometheus](https://lupyuen.github.io/images/grafana-flow2.jpg)

[(Source)](https://lupyuen.github.io/articles/prometheus)

# What's Next

Now that we can decode CBOR Sensor Data in The Things Network, it becomes a lot easier to __ingest the Sensor Data into Prometheus__ (the open source Time Series Database).

In the next article we shall build an __IoT Monitoring System__ that stores the __Sensor Data with Prometheus__ and visualises the data in a __Grafana Dashboard__...

-   [__"Monitor IoT Devices in The Things Network with Prometheus and Grafana"__](https://lupyuen.github.io/articles/prometheus)

![Monitoring Devices on The Things Network with Prometheus and Grafana](https://lupyuen.github.io/images/prometheus-grafana4.png)

Many Thanks to my [__GitHub Sponsors__](https://lupyuen.github.io/articles/sponsor) for supporting my work! This article wouldn't have been possible without your support.

-   [Sponsor me a coffee](https://lupyuen.github.io/articles/sponsor)

-   [Discuss this article on Reddit](https://www.reddit.com/r/TheThingsNetwork/comments/qafzu4/cbor_payload_formatter_for_the_things_network/?utm_source=share&utm_medium=web2x&context=3)

-   [Read "The RISC-V BL602 / BL604 Book"](https://lupyuen.github.io/articles/book)

-   [Check out my articles](https://lupyuen.github.io)

-   [RSS Feed](https://lupyuen.github.io/rss.xml)

_Got a question, comment or suggestion? Create an Issue or submit a Pull Request here..._

[`lupyuen.github.io/src/payload.md`](https://github.com/lupyuen/lupyuen.github.io/blob/master/src/payload.md)

# Notes

1.  This article is the expanded version of [this Twitter Thread](https://twitter.com/MisterTechBlog/status/1448846003608567809)

1.  __Be careful when decoding CBOR__ with our Payload Formatter!

    Some Data Types in CBOR don't exist in JSON, and might throw errors.
    
    [(See this mapping of CBOR to JSON types)](https://json.nlohmann.me/features/binary_formats/cbor/#deserialization)

    When we're encoding Sensor Data with CBOR, use Data Types that are supported in JSON: Integers, Floats, Strings and Maps.

1.  Here's a tip for __debugging Payload Formatters__...

    We can't use __console.log__ for logging Debug Messages, but we can return Debug Messages as __Decoder Warnings__...

    ```javascript
    //  Decode the payload in the Uplink Message
    function decodeUplink(input) {
      //  Data and warnings to be returned to The Things Network
      var data     = {};  
      var warnings = [];

      //  Omitted: Decode the payload
      ...

      //  For Debugging: Show the contents of
      //  `input.bytes` in the decoder warnings
      warnings.push(input.bytes);

      //  Return the decoded data and decoder warnings
      return {
        data:     data,
        warnings: warnings
      };
    }
    ```

    The Decoder Warnings (and our Debug Messages) will be shown when we click on a message in __Applications → (Your Application) → Live Data__.
