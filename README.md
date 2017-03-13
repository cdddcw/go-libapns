go-libapns
==========

APNS library for go

The idea here is to be a simple low level library that will handle establishing a connection and sending push notifications via Apple's apns service with thought towards throughput and performance.

Handles the latest Apple push notification guidelines at https://developer.apple.com/library/content/documentation/NetworkingInternet/Conceptual/RemoteNotificationsPG/APNSOverview.html

Specifically will implement the binary framed format by batching push notifications. Each batch will be flushed either every 10ms or when a frame is full. A frame is full when the framed format cannot fit anymore data into a tcp packet (65535 bytes). Due to this framing, when finished with the apns connection, one should call Disconnect() to flush any remaining messages out the door.

##Godoc
Located here [![GoDoc](https://godoc.org/github.com/joekarl/go-libapns?status.svg)](https://godoc.org/github.com/joekarl/go-libapns)

##Installation

```bash
> go get github.com/joekarl/go-libapns
```

##Basic Usage
```go
package main

import (
    apns "github.com/joekarl/go-libapns"
    "io/ioutil"
)

func main() {
  certPem, err := ioutil.ReadFile("../certs/PushTestCert.pem")
  if err != nil {
    panic(err)
  }
  keyPem, err := ioutil.ReadFile("../certs/PushTestKey-noenc.pem")
  if err != nil {
    panic(err)
  }

  apnsConnection, _ := apns.NewAPNSConnection(&APNSConfig{
      CertificateBytes: certPem,
      KeyBytes: keyPem,
  })

  payload := &apns.Payload {
      Token: "2ed202ac08ea9...cf8d55910df290567037dcc4",
      AlertText: "This is a push notification!",
  }

  apnsConnection.SendChannel <- payload
  apnsConnection.Disconnect()
}
```
**Note** This example doesn't take into account essential error handling. See below for error handling details

**Payload.Badge Need to Know** Apple specifies that one should set the badge key to 0 to clear the badge number. This unfortunately has the side effect of causing the go JSON serializer to omit the badge field. Luckily Apple uses negative badge numbers to clear the badge as well. So for our purposes, a badge > 0 will set the badge number, a badge < 0 will clear the badge number, and a badge == 0 will leave the badge number as is.

##Creating an APNS connection
Creating a connection consists of a couple of steps. They are:

* Creating a tcp socket to Apple's servers
* Initiating a TLS session using your Pem certs

There are two ways to do this using go-libapns. They are:

* NewConnection(*APNSConfig)
* SocketAPNSConnection(net.Conn, *APNSConfig)

The `NewConnection` method will validate your config, create the tcp connection, initiate a TLS session, and return a new APNSConnection. You probably will always use this way of creating a connection.

If you are on a platform that needs to create a custom socket (like Google App Engine), you can use the `SocketAPNSConnection` method. This takes a `net.Conn` (should be a tcpSocket), validates your config, initializes a TLS session, and returns a new APNSConnection.

##Pem Certs
You should provide your apns certificate as separated cert/key pem files. Currently go doesn't support password protected pem files (https://github.com/golang/go/issues/6722) so you'll need remove the password from your key pem.

####Separate pem files from p12
```sh
openssl pkcs12 -clcerts -nokeys -out cert.pem -in cert.p12

openssl pkcs12 -nocerts -out key.pem -in key.p12
```

####Remove password from pem file
```sh
openssl rsa -in key.pem -out key-noenc.pem
```

##Error Handling
As per Apple's guidelines, when a connection is closed due to error, the id of the message which caused the error will be transmitted back over the connection. In this case, multiple push notifications may have followed the bad message. These push notifications will be supplied on a channel **as well as any other unsent messages** and will be then available to re-process. Also when writing to the send channel, you should wrap the send with a select and case both the send and connection close channels. This will allow you to correctly handle the async nature of Apple's error handling scheme. See this gist (https://gist.github.com/joekarl/86d9bdb8f9af044710b7) for a full featured example of how to integrate go-libapns with proper shutdown handling and looped connection handling.

##Persistent Connection
go-libapns will use a persistant tcp connection (supplied by the user) to connect to Apple's APNS gateway. This allows for the greatest throughput to Apple's servers. On close or error, this connection will be killed and all unsent push notifications will be supplied for re-process. **Note** Unlike most other APNS libraries, go-libapns will NOT attempt to re-transmit your unsent payloads. Because it is trivial to write this retry logic, go-libapns leaves that to the user to implement as not everyone needs or wants this behavior (i.e. you may want to put the messages that need resent into a queue or store them for later).

##Feedback Service
Apple specifies that you should connect to the feedback service gateway regularly to keep track of devices that no longer have your application installed. go-libapns provides a simple interface to the feedback service. Simply create a `APNSFeedbackServiceConfig` object and then call `ConnectToFeedbackService`. This will return a list of device tokens that you should keep track of and not send push notifications to again (specifically this will return a List of `*FeedbackResponse`)

##Push Notification Length
Apple places a strict limit on push notification length (currently at 2048 bytes). go-libapns will attempt to fit your push notification into that size limit by first applying all of your supplied custom fields and applying as much of your alert text as possible. This truncation is not without cost as it takes almost twice the time to fix a message that is too long. So if possible, try to find a sweet spot that won't cause truncation to occur. If unable to truncate the message, go-libapns will close it's connection to the APNS gateway (you've been warned). This limit is configurable in the APNSConfig object.

_Note: Prior to iOS 8, the limit was 256 bytes. APNS will accept and deliver up to 2048 bytes to devices
running iOS 8 as well as those running on older versions of iOS._

##TCP Framing
Most APNS libraries rely on the OS Nagling to buffer data into the socket. go-libapns does not rely on Nagling but does do what it can to optimize the number of bytes sent per TCP frame. The two relevant config options that control this behavior are:

* MaxOutboundTCPFrameSize - (default TCP_FRAME_MAX) Max number of bytes to send per TCP frame
* FramingTimeout - (default 10ms) Max time between TCP flushes

TCP_NODELAY can be turned on with this setup by setting the FramingTimeout to anything less than 0 (like -1). In practice you want this buffering to occur, so best to leave defaults. If you're concerned about a (max) 10ms delay between your push notifications being sent onto the socket be aware that this is much much much shorter than the default linux Nagle timeout of 1 second.

##What's with using channels for writing to the connection?
Basically, this makes it easier to synchronize error handling and socket errors. Not sure if this is the best idea, but definitely works.

##APNSConfig
The only required fields are the CertificateBytes and KeyBytes.
The other fields all have sane defaults

```go
InFlightPayloadBufferSize       int                     //number of payloads to keep for error purposes, defaults to 10000
FramingTimeout                  int                     //number of milliseconds between frame flushes, defaults to 10ms
MaxPayloadSize                  int                     //max number of bytes allowed in payload, defaults to 2048
CertificateBytes                []byte                  //bytes for cert.pem : required
KeyBytes                        []byte                  //bytes for key.pem : required
GatewayHost                     string                  //apple gateway, defaults to "gateway.push.apple.com"
GatewayPort                     string                  //apple gateway port, defaults to "2195"
MaxOutboundTCPFrameSize         int                     //max number of bytes to frame data to, defaults to TCP_FRAME_MAX
                                                        //generally best to NOT set this and use the default
SocketTimeout                   int                     //number of seconds to wait before bailing on a socket connection, defaults to no timeout
TlsTimeout                      int                     //number of seconds to wait before bailing on a tls handshake, defaults to 5 sec
```

#License
The MIT License (MIT)

Copyright (c) 2014 Karl Kirch

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
