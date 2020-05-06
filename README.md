# LuWS - A Luup WebSocket Client

LuWS is a WebSocket client implementation for Luup systems (Vera and openLuup). It implements the WebSocket protocol as described in RFC6455, except for 64-bit frame length. It supports encrypted (SSL/TLS) connections.

LuWS supports the SockProxy plugin, which allows it to respond immediately, asynchronous to available receive data. This makes full duplex communications more efficient, as polling for receivable data isn't necessary. Polling for data introduces the possibility that a message may not be received and handled with any better response time than the poll interval -- that is, if the poll interval is 15 seconds, it will often be nearly that long before an available datagram is processed, unless asynchronous receive assisted by SockProxy is used.

## LuWS' To-Do List

The following are known TBDs, which includes several known non-compliances with RFC6455:

* SECURITY: The `Sec-WebSocket-Accept` response header is not verified in the current implementation, although correcting this is a high-priority task and expected to be completed shortly.
* SECURITY? According to RFC6455 section 5.1, we MUST error and close when receiving a frame/fragment from a server that is masked; we don't do this currently, we unmask it and roll on.
* 64-bit frame lengths are not supported (receive or transmit). There are no plans to fix this currently. Messages of virtually any size can be sent and received, as long as no fragment (a message is comprised of one or more fragments) exceeds 65535 bytes in length.
* An LTN12 *source* can be optionally used for sending data, but there is no use of, for example, *sinks* on the receive side. I'm still contemplating how LTN12 could be useful elsewhere.

## Installing LuWS

LuWS is a drop-in for your Luup plugin/subsystem. It is recommended that you change the filename to `L_yourpluginname_LuWS.lua`, and modify the `module` statement at the head of the file in the same way (omitting the `.lua` suffix there) to avoid potential version conflicts with other plugins that may want to use LuWS as well.

## Using LuWS -- Basics

This section will cover basic LuWS integration -- **don't skip it!** If you are going to support asynchronous receive using SockProxy, we'll cover the additional details of that in the next section. Most of this section applies to all use cases, so please read on...

To load LuWS, just use a `require` statement as you would any other module:

`luws = require "L_MyPluginName_LuWS"`

To open a session, `luws.wsopen()` is called with the endpoint URL. The function will return a blind table -- a table you should not play with, but keep to pass around to other functions where needed. If an error occurs during the connection attempt, `wsopen()` will return two values, *false* and an error message, so always receive two values on your call:

```
local luws = require "L_MyPluginName_LuWS"

local conn, err = luws.wsopen( "wss://www.websocket.org/echo", message_handler, options )
if not conn then
	luup.log("The connection failed: " .. err)
else
	luup.log("Successful connection!")
end
```

The `message_handler` is a function reference (not a name in a string) or closure that will handle *messages* received from the endpoint. Messages are complete datagrams after any necessary reassemby resulting from fragmentation (a message is comprised of one or more *fragments*). The message handler is also called for certain "control" situations, such as an unexpected disconnect of the endpoint or other error in communication. See *Your Message Handler* below for more details.

The `options` argument is a table containing connection options, further described below.

To send a message, you call `wssend( conn, opcode, data )`. The `conn` parameter is the table handed back by the previous successful `wsopen()` (all functions take this argument, so I won't repeat this description on future calls). The opcode is the WebSocket message type (per the RFC, 1 = text data, 2 = binary data). The data can be either a string or a LTN12 *source* (see the LTN12 module documentation).

To receive a reply, you call `wsreceive( conn )`. You should call this function periodically, not just after sending data, so that any unprompted message received from the endpoint gets handled. This is receive polling (we'll talk about asynchronous receive later), and you should do this polling as often as makes sense for your application (every 15 seconds is sane for apps that need to respond quickly to receive messages -- it doesn't waste much CPU if no data is available). But it can be much longer, and you may apply other heuristics, depending on the nature of your endpoint. For example, if your endpoint only sends replies to things you send, there's no sense trying to receive data when you haven't sent anything. That's all up to you to determine; the protocol and LuWS do not enforce any rules in this respect.

Finally, any time you need to discard the current session, make sure you call `wsclose()` to tear down the session and free its resources. You must also always do this when an error occurs before recovering by opening a new session, as well as any exit of your plugin. If you don't do this, you will leak resources and eventually cause a Luup reload, or in some cases, a system crash/restart.

### Your Message Handler

If your `wsreceive()` call actually picked up a message from the endpoint, LuWS will call the message handler function you specified to `wsopen()`. Your message handler declaration should look like this:

`function message_handler( conn, opcode, data, ... )`

The `opcode` is the opcode (numeric, per the RFC) of the received message, and the `data` is the accompanying data (as a string) of the received message. Fragmented messages are reassembled by LuWS before being passed to your handler, so there is no need to worry about that -- the message you receive is complete. The `...` is any additional arguments that you asked be passed to the handler (by setting `handler_args` in `wsopen()`'s `options` -- see *Options* below).

If the `opcode` received by your handler is *false* (boolean), then an error has occurred in receiving data. **You must always test for this.** In this case, the `data` will be the error message. The most common errors are "message timeout" and "receiver error: closed"; the former indicates that a message was not received for longer than the message timeout setting (see *Options* below), and the latter indicates that the TCP connection has been broken for some reason. Typically, you will just want to log the message, `wsclose()` the connection (to be sure all of its resources are released), and open a new one (discard the old `conn` from the previous `wsopen()`).

### Options

There are a few options you can define when calling `wsopen()` to control the behavior of LuWS' handling of connections and data. Generally, you won't need to supply any options, the defaults are usually acceptable.

* `handler_args`: a Lua table containing additional arguments that should be passed to your message handler when it is called, default: none;
* `receive_timeout`: if no messages are received from the endpoint in this many seconds, the connection is assumed to have died somehow and will cause an error notification to the message handler (0=default, no timeout is enforced).
* `receive_chunk_size`: maximum size of a socket read; it is usually not necessary to modify this, but some older Vera systems with little RAM may benefit from a size smaller than the default 2048.
* `max_payload_size`: maximum size of a message that can be received and passed to the handler; longer messages generate an error notification to the handler. The default is 64K bytes. Note that this is the total size of the message, not the maximum size of a fragment (several fragments assemble into one message). Fragments are always limited to 65535 bytes in the current implementation, but the `max_payload_size` can be set much larger.
* `ssl_protocol`: LuaSec `protocol` option value for SSL/TLS connections, default `all` (systems using older versions of LuaSec may not be able to use `all` and should use `tlsv1_2`);
* `ssl_verify`: LuaSec `verify` option value for SSL/TLS connections, default `none` (**Nota Bene!** this turns off peer certificate validation and is thus less secure, but is necessary for connecting to systems with self-signed certificates. If you are connecting to a system with valid certificates, it is strongly recommended that you set this to `peer`);
* `ssl_mode`: LuaSec `mode` option value for SSL/TLS connections, default `client`;
* `ssl_options`: LuaSec `options` value for SSL/TLS connections, table/array, default `{ "all" }`;
* `use_masking`: turn masking on (*true*) or off (*false*) for all frames sent to the endpoint. RFC6455 requires that all frames sent by a client be masked, so this is *true* by default. Some trivial WebSocket server implementations, however, don't do it, and some don't care (if your app sends a lot of data frequently, and your endpoint doesn't care, turning masking off saves a little CPU time). This setting has no effect on received frames;
* `control_handler`: normally LuWS handles control frames and does not even notify the app that they have been received; a function or closure given at this key will be called when control messages are received. See the *Reference* section for additional details.
* `connect`: connect function for creating a TCP socket; a default function is provided and this function only needs to be provided if somehow the caller needs to control how the socket is created.

The full details of the SSL option values can be found in the LuaSec documentation.

## Using LuWS with SockProxy

To use SockProxy's asynchronous receive capabilities with LuWS, you need to modify your program/plugin as follows:

1. Provide an override `connect` function (in `options` for `wsopen()`) that attempts to open the connection to SockProxy first;
2. Provide an action declaration in your plugin's service for SockProxy to use to notify you that pending receive data is waiting;
3. Provide an implementation of that action that calls `wsreceive()`.

That's it. Although there are more details, that's really all that's needed to get asynchronous receive with LuWS (and that's pretty much the same as it is for all SockProxy-integrated apps).

For step one, here's a replacement `connect` function:

```
-- Attempt proxy connection first, fallback to regular
local usingProxy = false -- declare global so other parts of our plugin know if proxy in use or not
function connect_sockproxy( ip, port, options )
	local sock = socket.tcp()
	if not sock then
		return nil, "Can't get socket for connection"
	end
	-- Try SockProxy plugin first
	sock:settimeout( 5 )
	local st,se = sock:connect( "127.0.0.1", 2504 )
	if st then
		local ans,ae = sock:receive("*l")
		if ans and ans:match("^OK TOGGLEDBITS%-SOCKPROXY") then
			sock:send( string.format("CONN %s:%d NTFY=%d/%s/%s RTIM=0 PACE=0\n",
				ip, port, NOTIFY_DEVICE, NOTIFY_SERVICE, NOTIFY_ACTION ) )
			ans,ae = sock:receive("*l")
			if ans and ans:match("^OK CONN") then
				-- Successful proxy connection! Return socket to LuWS
				usingProxy = true
				return sock
			end
		end
		-- Proxy negotiation failed; fallback to plain connect.
		sock:shutdown("both")
	end
	-- No good. Close socket, make a new one.
	luup.log("SockProxy connection attempt failed, using plain socket")
	sock:settimeout( 15 )
	local r, e = sock:connect( ip, port )
	if r then
		usingProxy = false
		return sock
	end
	sock:close()
	return nil, string.format("Connection to %s:%s failed: %s", ip, port, tostring(e))
end
```

The only thing you need to modify in the above code is the `NOTIFY_` fields. The `NOTIFY_DEVICE` should be replaced with the device number of your plugin device; the `NOTIFY_SERVICE` should be replaced with the service ID your receive notification action is declared in (usually your plugin's primary/own service); and `NOTIFY_ACTION` is the name of the action (I always just use `HandleReceive`).

Now, to use this override connection opener, all we need to do is pass it to `wsopen()` in the `options` argument:

```
luws = require("L_MyPluginName_LuWS")
...
local conn, err = luws.wsopen( url, message_handler, { connect=connect_sockproxy } )
...
```

Step two, declaring the action that SockProxy should use to notify your plugin when data is waiting, is simple. Add this block to the `<actionList>` section of your plugin's primary/own service (`S_.xml`) file:

```
	<action>
		<name>HandleReceive</name>
		<argumentList>
			<argument>
				<name>Pid</name>
				<direction>in</direction>
			</argument>
		</argumentList>
	</action>
```

Step three, add the implementation for this new action:

```
	<action>
		<serviceId>...your plugin's service Id...</serviceId>
		<name>HandleReceive</name>
		<job>
			return LoboPlugin.actionHandleReceive( lul_device, lul_settings )
		</job>
	</action>
```

The above form is what I use, but your preferred style may be different. When I write a plugin, I load the plugin's code as a module in my startup function (not using the `<files>` section of the `I_.xml` implementation file) and store it as a global variable (that's the `LoboPlugin` variable/table in the above example). In that module, I create a public (not `local`) function called `actionHandleReceive`, and all it needs to do is call `wsreceive()`:

```
function actionHandleReceive( dev, params )
	luws.wsreceive( conn ) -- luws and conn are assumed to be global here
	return 4,0 -- required, return successful job completion
end
```

Again, this is my style. Do what suits yours and your plugin. The objective of the action is to simply call `wsreceive()` with the correct `conn`... however that gets done, you're winning.

**NOTE:** The notification handler example above has been implemented as a job (as opposed to just using `<run>`), and this is recommended. It creates more flexibility for Luup in scheduling when the receive data is processed, and usually contributes little to latency. There is no reason you can't do it as a `<run>` action, if you wish; just make sure you change the return value of `actionHandleReceive` to a boolean, since that action type requires it.

**One more detail...** In our custom `connect_sockproxy` function, we set the global variable `usingProxy` to *true* or *false* based on whether or not we were successful connecting through SockProxy for our current session. How you use that, or *if* you use that, is up to you, but here's how I use it:

* If `usingProxy` is *false*, I create a timer/delay task to periodically call `wsreceive()` at an interval frequently enough for the app's needs (this is the same polling you would do when not using SockProxy);
* If `usingProxy` is *true*, I still create a timer/delay task, but I use a much longer delay period (typically 60 or 120 seconds), just as a backup to the receive notification, in case somehow a notification gets missed (I've never seen it happen, but I foresee it's a possibility so why not guard for it?).

By doing this, it's possible for your app to behave well on systems that use SockProxy and those that don't, with no code changes. It will just work better (more responsive, less CPU) where SockProxy is running.

## Reference

### `conn, errmsg = wsopen( url, message_handler [, options ] )`

Opens a connection to the specified URL. Protocols `ws` and `wss` are accepted (only). The `message_handler` is a function reference or closure to receive and handle messages from the endpoint. The optional `options` table contains settings modifications from default for the session (see *Options* above).

This function returns one or two values. If the first value is *false*, a connection error occurred and the second value is the error message. Otherwise, the first (and only) value returned is the connection handle (a blind table).

### `status = wssend( conn, opcode, bytedata )`

Sends a WebSocket datagram using opcode `opcode` and payload data from the string `bytedata`. The `conn` is a connection handle previously returned by `wsopen()`.

The return status boolean indicates success or failure. If *false*, the send failed and the connection should be closed and a new session started.

### `more, nb = wsreceive( conn )`

Receive and process data from the endpoint. This function should be called periodically, and/or when using SockProxy, whenever a notification is received that data is waiting.

The return value `more` indicated that additional data may be waiting; in this case, an additional call to `wsreceive()` should be made as soon as possible. The recommended practice for Luup is to use a delay timer of 0, which effectively yields to the operating system but calls you back as quickly as possible. When an endpoint is transmitting frequently, this avoids the possibility that the receive loop causes a deadlock or large pause in system operation.

If `more` is `nil`, a receive error occurred and `nb` contains the error message; the connection should be closed and a new session started.

### `wsclose( conn )`

Terminates the connection with the endpoint, closes and releases the underlying socket, and releases all other resources associated with the session. The `conn` may then be discarded (e.g. set it to `nil`) for garbage collection.

This function should be called whenever there is an error, prior to opening a replacement session. This is vital for freeing up limited system resources. Failure to do so will lead to a system crash.

### `message_handler( conn, opcode, data, ... )`

This is not a function provided by the library, but rather the prototype of a function that you must provide for the LuWS as the second argument to `wsopen()`. It will be called by LuWS whenever a full message has been received, or when an error occurs while attempting to receive on the session.

When a message is received, the `opcode` will be a number corresponding to the WebSocket opcode for the datagram. When an error occurs, `opcode` will be *false* (boolean), and `data` will contain the error message, which should be logged; the connection should then be closed and a new replacement session started.

The `...` is a placeholder for any arguments you additionally requested be passed to the handler by providing them in the `handler_args` key of `options` for `wsopen()`.

### `control_handler( conn, opcode, data, ... )`

This is an optional callback function that you can provide if you want to intercept WebSocket control messages. Normally, your app is not notified when they are received, and LuWS handles the control messages itself (e.g. it sends a *pong* when it receives a *ping*). If you want to know when control messages are received, or preempt LuWS default control message handling, supply a function or closure with this prototype as `options.control_handler` to `wsopen()`. Your function will be called before LuWS default handling takes place. If you wish to prevent LuWS default handling, return boolean *false*. Any other value (or no return value at all) will allow the default handling to be performed.

LuWS default handling for control messages is as follows:

* If a *close* message (opcode 0x08) is received, and the session is as yet not known to be closing, it is marked as pending close and a *close* message is sent in reply. The message handler is also called with `opcode`=(boolean)*false* and `data`=(string)"receiver error: closed" to notify the app that the endpoint has requested that the connection be closed. It is up to the message handler/application to call `wsclose()` to finalize the closure and release of the session.
* In response to a received *ping* control message (opcode 0x09), LuWS will send a *pong* (opcode 0x0a);
* All other opcodes are ignored.

## License

LuWS and all associated materials: Copyright (C) 2020 Patrick H. Rigney, All Rights Reserved.

LuWS is offered under the MIT License.

## References

[RFC6455 - The WebSocket Protocol](https://tools.ietf.org/html/rfc6455)
