# Building Custom Handlers

To implement a new application you need to create a `lorawan_application_xxx.erl`
module implementing the `lorawan_application` behaviour and register it in the
[`sys.config`](lorawan_server.config):
```erlang
{lorawan_server, [
    {plugins, [
        {<<"my-app">>, lorawan_application_xxx},
        ...
    ]}
```

## lorawan_application Behaviour

Your module needs to export `init/1`, `handle_join/3` and `handle_rx/4` functions.

### init(App)

The `init/1` will be called upon server initialization. It shall return either
`ok` or a tuple `{ok, PathsList}` to define application specific URIs (e.g.
REST API or WebSockets). For more details see
[Cowboy Routing](https://github.com/ninenines/cowboy/blob/master/doc/src/guide/routing.asciidoc).

### handle_join(DevAddr, App, AppID)

The `handle_join/3` will be called when a new node joins the network. The function
shall return either `ok` or `{error, error_description}`.

### handle_rx(DevAddr, App, AppID, RxData)
The `handle_rx/4` will be called upon reception of a LoRaWAN frame:
  * *DevAddr* is the 4-byte device address
  * *App* is the application name defined in the `sys.config`
  * *AppID* is an opaque value assigned to the device
  * *RxData* is the #rxdata{} record with:
    * *port* number
    * *data* binary
    * *last_lost* flag indicating that the last confirmed response got lost
    * *shall_reply* flag indicating the MAC has to reply to the device even if
      the application sends no data

The *last_lost* flag allows the application to decide to send new data instead of
retransmitting the old data.

The *shall_reply* flag allows the application to send data when a downlink frame
needs to be transmitted anyway due to a MAC layer decision.
This flag is set when:
  * Confirmed uplink was received, which needs to be acknowledged
  * ADR ACK was requested by the device
  * MAC layer needs to send a command back to the device

The function may return:
  * *ok* to not send any response
  * *retransmit* to re-send the last frame (when *#rxdata.last_lost* was *true*)
  * *{send, #txdata{}}* to send a response back, where the #txdata{} record may include:
    * *port* number
    * *data* binary
    * *confirmed* flag to indicate a confirmed response (default is *false*)
    * *pending* flag to indicates the application has more data to send (default is *false*)
  * *{error, error_description}* to record a failure and send nothing

For example:
```erlang
handle_rx(DevAddr, <<"my-app">>, AppID, #rxdata{last_lost=true}) ->
    retransmit;
handle_rx(DevAddr, <<"my-app">>, AppID, #rxdata{port=PortIn, data= <<"DataIn">>}) ->
    %% application logic
    %% ...
    {send, #txdata{port=PortOut, data= <<"DataOut">>}}.
```