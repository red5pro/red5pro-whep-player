# Red5 WHEP Player

This repository is a working example of utilizing the [Red5 HTML SDK](https://github.com/red5pro/red5pro-webrtc-sdk) to create a bare-bones video player solution that can be used as-is or easily embedded into an `iframe` for playback of live streams.

# Introduction

It is a common misconception that our browser-based Red5 HTML SDK provides a "player" that can be added to a web page. Our SDK is designed to ease the communication process with the Red5 Server in negotiation and establishing a connection for live streaming. As a result of this negotiation, a `MediaStream` is available which can be assigned as a source object to an HTML `video` element and will enable playback - for both a publisher (`WHIPClient`) and subscriber (`WHEPClient`).

> The Red5 HTML SDK also handles ease in assigning this `MediaStream` to the desired `video` element for playback, as well.

Due to the nature of browsers providing their own implementation of the HTML `video` element - along with their own consistent designs - and their adherence to a `MediaStream` being allowed a as a source object to that element, our SDK team decided that to allowing for the browser to present its own controls and design for playback was the correct approach.

But fear not those who are looking for easy ways to include a "player" from Red5 on their page! This article will provide a bare-bones example of live streaming playback on the web using the Red5 HTML SDK and query parameters for configuration.

# Set Up

The Red5 HTML SDK requires some initialization configuration properties in order to begin the negotiation process with the Red5 Server to generate a `MediaStream` to be consumed and played back. Most properties have defaults that, in most cases, will not need to be modified - however, there are two properties that will need to be provided based on your Red5 deployment and stream:

* `endpoint` - The WHEP-based endpoint URL related to your Red5 Server Deployment.
* `streamName` - The name of the live stream to consume and playback.

## Standalone & Stream Manager

It should be noted that the `endpoint` structure of the init configuration property differs slightly based on your type of deployment.

For a Standalone Red5 Server (e.g., single deployment, self-managed and non-clustering), the `WHEP` URL endpoint will have the following structure:

```text
const streamName = 'my-stream'
https://my-deployment.red5/live/whep/endpoint/${streamName}
```

While the structure for the URL endpoint when using a Stream Manager deployment (such as that provided and managed by [Red5 Cloud](https://cloud.red5.net/)) will be the following:

```text
const streamName = 'my-stream'
https://my-deployment.cloud.red5/as/v1/proxy/whep/live/${streamName}
```

> The reason for the structure difference is to signal to the endpoint, under a Stream Manager deployment, to use the `proxy` service. The discussion of CORS and browser security is too large for this document, but the `proxy` service is required to properly connect WHIP/WHEP clients to the desired nodes in a Stream Manager environment (i.e., cluster).

## WHEPClient

To consume a live stream for playback, our Red5 HTML SDK offers a WHEP-based client. The `WHEPClient` - under the hood - is based on the [WebRTC-HTTP egress](https://www.ietf.org/archive/id/draft-ietf-wish-whep-03.html)(WHEP) protocol providing the ability to negotation and establish a connection using HTTP/S requests. This removes the requirement for a WebSocket, which historically has been used for the role of negotiation and connection.

The following demonstrates how to quickly set up and start a live stream subscription for playback in the browser:

```js
const config = getConfiguration()
subscriber = new WHEPClient()
subscriber.on('*', onSubscriberEvent)
await subscriber.init(config)
await subscriber.subscribe()
```

By default - if the server connection is successfully established - the `MediaStream` delivered to the client will be assigned to a `video` element with the `id` attribure of `red5pro-subscriber`:

```html
<video id="red5pro-subscriber" autoplay playsinline controls></video>
```

> Note: It is recommended to have the attributes for the `video` element as defined above for auto-playback of the live stream once established. Because of browser security restrictions, some muting of audio may occur. The Red5 HTML SDK will properly handle such exceptions, but you may need additional development for proper UX. Please see the [Autoplay and Browser Restrictions](#autoplay-and-browser-restrictions) section for more information.

# Full Example

The following provides a full example of the HTML and JavaScript pieces required for a basic web-based player of a live stream delivered from Red5.

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <meta name="robots" content="noindex, nofollow" />
    <title>Red5 Player</title>
    <script src="//webrtchacks.github.io/adapter/adapter-latest.js"></script>
    <script src="//cdn.jsdelivr.net/npm/red5pro-webrtc-sdk@latest/red5pro-sdk.min.js"></script>
  </head>
  <body>
    <div class="app">
        <div class="video-container subscribe-container">
            <video id="red5pro-subscriber" class="red5pro-subscriber" autoplay playsinline controls></video>
        </div>
    </div>
    <script>
        // See next code section.
    </script>
  </body>
</html>
```

```js
;((window, red5) => {

const { WhepClient } = red5

let subscriber
const host = 'my-deployment.cloud.red5'
const streamName = 'my-stream'

const getConfiguration = (standalone = false) => {
    if (standalone) {
        return {
            endpoint: `https://${host}/live/whep/endpoint/${streamName}`,
            streamName,
            connectionParams: configureAuth({})
        }
    }
    // Else, Stream Manager 2.0 integration.
    return {
        endpoint: `https://${host}/as/v1/proxy/whep/live/${streamName}`,
        streamName,
        connectionParams: configureAuth({
            nodeGroup
        })
    }
}

const onSubscriberEvent = event => {
    console.log('[Subscriber]', type)
}

const subscribe = async () => {
    try {
        const config = getConfiguration()
        subscriber = new WHEPClient()
        subscriber.on('*', onSubscriberEvent)
        await subscriber.init(config)
        await subscriber.subscribe()
    } catch (e) {
        await unsubscribe()
    }
}

const unsubscribe = async () => {
    if (!subscriber) {
        return
    }
    try {
        subscriber.off('*', onSubscriberEvent)
        await subscriber.unsubscribe()
        subscriber = undefined
    } catch (e) {
        // noop.
    }
}

window.addEventListener('beforeunload', async () => {
    await unsubscribe()
})
window.addEventListener('pagehide', async () => {
    await unsubscribe()
})

start()

})(window, window.red5prosdk)
```

## Note of Browser vs NPM install

The above example assumes you have included the Red5 HTML SDK dependency as a browser script from a CDN (such as the one available from [cdn.jsdelivr.net](https://cdn.jsdelivr.net/npm/red5pro-webrtc-sdk@latest/red5pro-sdk.min.js)).

If you install the SDK through NPM (e.g., `npm install --save red5pro-webrtc-sdk`) and use a packager - such as `Vite` - you most likely will access the SDK like the following:

```js
import { WHEPClient } from 'red5pro-webrtc-sdk
```

# Example Page & iframe

The [index.html](index.html) page provided in this repository has the above example included in it, along with the ability to define init configuration parameters using query params to have a Red5 "player"  - allowing you to use this page along with param configuration in your own projects without any further development!

## Query Params

When loading the `index.html` page in a web browser, the URL can be appended with key-value pairs known commonly as "query params". These parameters can be access by the page, and in the context of the example, used to assign configuration properties for initialization of the `WHEPClient` for playback.

The following query params are available, and listed from most relevant to your needs in descending order:

* `host` - The domain name endpoint at which the Red5 Server is deployed.
* `stream_name` - The name of the live stream to consume and playback.
* `standalone` - Whether your deployment is a standalone of not (Stream Manager, Cloud, etc.).
* `node_group` - The optional name of the target Node Group if targeting a Stream Manager deployment.
* `retry` - Whether to continually retry playback if subscription is stopped.
* `retry_delay` - The delay - in milliseconds - to wait until a retry.
* `object_fit` - The CSS property to assign to the `video` element. [See documentation](https://developer.mozilla.org/en-US/docs/Web/CSS/Reference/Properties/object-fit).
* `app` - The application context the stream resides on. Most likely the default: `live`.
* `rta_user` - If using Round-Trip Authentication, the username associated.
* `rta_pass` - If using Round-Trip Authentication, the password associated.
* `rta_token` - If using Round-Trip Authentication, the token associated.
* `api_version` - The API version of the Stream Manager deployment. Default: `v1`.

## Example URL with Params

The following is an example of the URL you can place into the URL bar of your favorite web browser to load the Red5 Player example with configuration parameters related to your deployment:

```txt
https://localhost:8001/index.html?host=my-deployment.red5&stream_name=my-stream&standalone=true&retry=true
```

> The above assumes that you are hosting the `index.html` under a basic web server on `localhost`. An example using python would be: `python -m http.server 8001`

## iframe Usage

This repository provides a basic example [index.html](index.html) of a Red5 "player" that can be further used to embed into an `iframe` element on your own web-based pages and applications.

# Autoplay and Browser Restrictions

