# Connectors

Connectors define transport of data fields to/from external servers. Each
connector is linked with one Handler and specifies:
 * Communication protocol
 * Target endpoint, i.e server address and message topics
 * Encoding of the data fields

## Configuration

To create a new connector you set:
 * *Connector Name*
 * *Application* that references a specific backend *Handler*
 * *Format* of the message payload, which can be:
   * *JSON* to encode data fields as [JSON](http://www.json.org) structures like `{"NameOne":ValueOne, "NameTwo":ValueTwo}`.
   * *Raw Data* to send just the content of the *data* field, without ant port numbers nor flags.
   * *Web Form* to encode fields in a query strings like `NameOne=ValueOne&NameTwo=ValueTwo`.
 * *URI* defines the target host, which can be
   * `ws:` for Web Sockets
   * `http://host:port` for HTTP POST or `https://host:port` for HTTP/SSL
   * `mqtt://host:port` for MQTT or `mqtts://host:port` for MQTT/SSL
   * `amqp://host:port` for AMQP or `amqp://host:port` for AMQP/SSL
 * *Published Uplinks*, which is a server pattern for constructing the publication
   topic for uplink messages, e.g. `out/{devaddr}`. This can be used to include
   the actual DevEUI, DevAddr or other data field in the message topic.
 * *Published Events*, which is a server pattern for constructing the publication
   topic for event messages.
 * *Subscribe*, which is a topic to be subscribed. It may include broker specific
   wilcards, e.g. `in/#`. The MQTT broker will then send messages with a matching
   topic to this connector.
 * *Received Topic*, which is a template for parsing the topic of received
   messages, e.g. `in/{devaddr}`. This can be used to obtain a DevEUI, DevAddr or
   a device group that shall receive a given downlink.
 * *Enabled* flag that allows you to temporarily disable an existing connector.

On the Authentication tab:
 * *Client ID* is the MQTT parameter
 * *Auth* identifies the type of authentication:
   * *Username+Password* for common servers
   * *Shared Access Signature* for Microsoft servers
 * *Name* and *Password/Key* for plain authentication
 * *User Certificate* and *Private Key* if SSL authentication is needed

To include node-specific attributes the Published and Received Topic may include
data field names in curly brackets, e.g. `{deveui}` or `{port}`.

The Received Topic may also include the `#` (hash), which matches zero or more
characters, including any '/' delimiters. It can be used to ignore the leading
or training characters in the topic.

If the Connector is *Enabled* and a *Subscribe* topic is defined the server will
automatically connect to the backend server and subscribe this topic.

You can generate a self-signed *User Certificate* (`cert.pem`) and corresponding
*Private Key* (`key.pem`) by:
```
openssl req -x509 -newkey rsa:4096 -keyout privkey.pem -out cert.pem -days 365
openssl rsa -in privkey.pem -out key.pem
```

Please read the [Integration Guide](Integration.md) for detailed information on
how to connect to a specific IoT Platform like AWS IoT, IBM Watson IoT, MathWorks
ThingSpeak, Azure IoT Hub or Adafruit IO.


## Web Sockets

To create a web socket connector you set:
 * *URI* to `ws:` (with no hostname)
 * *Publish Uplinks* to a URL pattern starting with a slash, e.g. '/ws/uplink/{devaddr}'
 * *Publish Events* to another URL pattern, e.g. '/ws/events/{devaddr}'

To connect to the WebSocket, then open URL to the path you defined, i.e.
`ws://server:8080/ws/uplink/<DevAddr>` or `ws://server:8080/ws/events/<DevAddr>`.

The URL is matched against the pattern to select the target device(s). For example,
`ws://127.0.0.1:8080/ws/events/11223344` connects to events from the DevAddr *11223344*.

Multiple parallel connections may be established to one URL.
When the device sends a frame, all connected clients will receive the application data.
Any of the clients may then send a response back. If multiple clients send data to
the device the frames will be enqueued and sent one by one. The enqueued *Downlinks*
can be viewed via the [Node Administration](Nodes.md) page.

### Keep-alive

By default, the WebSocket connection will be closed if the client (e.g. your web browser)
sends for 1 hour no data back to the server. This is to avoid stale connections.

To keep the connection open for a longer time:
 * You can adjust the `{websocket_timeout, 360000}` configuration parameter to a higher
   value (in milliseconds).
 * You can even set `{websocket_timeout, infinity}` to disable the session expiration.
 * Or the client (browser) needs to keep sending **ping** frames.

The **ping** frames may not be enabled by default. To enable **ping** frames in Firefox,
go to **about:config** and set **network.websocket.timeout.ping.request** to (for example)
120 (seconds).

### Demo page

Demo client is available at [`http://127.0.0.1:8080/admin/ws.html`](../priv/admin/ws.html).
Enter the desired URL path (e.g. `/ws/events/11223344`) and the desired format
(Raw or JSON) and then establish a WebSocket connection.
The page will display data received from the device and allow you to send data back.

In the **Raw** mode all information must be entered as a string of hexadecimal digits,
without any spaces.
Each byte is represented by exactly 2 digits. For example, "4849" represents ASCII string "01".


## Generic MQTT Server

You can integrate with generic MQTT server (message broker), e.g. the
[RabbitMQ](https://www.rabbitmq.com/mqtt.html) or
[Mosquitto](https://mosquitto.org).

First of all, make sure you understand the
[terminology and principles of messaging](http://www.rabbitmq.com/tutorials/tutorial-one-php.html)
and that the MQTT protocol [is enabled](https://www.rabbitmq.com/mqtt.html)
in your broker.

Open the lorawan-server web-administration and create a Backend Connector:
 * *URI* defines the target host either as `mqtt://host:port` or `mqtts://host:port`
 * *Publish Uplinks* is a pattern for constructing the message topic
   of uplinks, e.g. `out/{devaddr}`.
 * *Subscribe* is a downlink topic to be subscribed by the lorawan-server,
   e.g. `in/#`.
 * *Received Topic* is a template for parsing the topic of received downlink
   messages, e.g. `in/{devaddr}`.

On the Authentication tab:
 * *Auth* shall be set to *Username+Password*, even when the *Name* and
   *Password/Key* are empty.

In order to consume the uplink messages sent by your devices you have to subscribe
at the message broker for the *Published Topic*, e.g. for `out/#` (or even just `#`) by:
```bash
mosquitto_sub -h 127.0.0.1 -p 1883 -t 'out/#' -u 'user' -P 'pass'
```

When using RabbitMQ, create a queue and then bind it to the `amq.topic` exchange
using the `out.#` (or `#`) binding key. Note that while MQTT uses slashes ("/") for
topic segment separators, RabbitMQ uses dots. RabbitMQ internally translates the two,
so for example the MQTT topic `cities/london` becomes in RabbitMQ `cities.london`.

To send a downlink message to one of your devices do e.g.
```bash
mosquitto_pub -h 127.0.0.1 -p 1883 -t 'in/00112233' -m '{"data":"00"}' -u 'user' -P 'pass'
```