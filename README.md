# Red5 WHEP Player

This repository is a working example of utilizing the [Red5 HTML SDK](https://github.com/red5pro/red5pro-webrtc-sdk) to create a bare-bones video player solution that can be used as-is or easily embedded into an `iframe` for playback of live streams.

* [Introduction](#introduction)
* [Set Up](#set-up)
* [Full Example](#full-example)
* [Example Page](#example-page)
* [iframe Usage](#iframe-usage)
* [Autoplay & Browser Restrictions](#autoplay-and-browser-restrictions)
* [Extra Credit - Player UI]()

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

const { WHEPClient } = red5

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
import { WHEPClient } from 'red5pro-webrtc-sdk'
```

# Example Page

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

# iframe Usage

This repository provides a basic example [index.html](index.html) of a Red5 "player" that can be further used to embed into an `iframe` element on your own web-based pages and applications.

The [iframe-form.html](iframe-form.html) page provides an easy form to fill out that will generate a URL for a Red5 Player that can be provided to Users with access to a web browser, or even used as the source for an `iframe` on your own web page!

> Visit the form at the following page: [https://red5pro.github.io/red5pro-whep-player/iframe-form.html](https://red5pro.github.io/red5pro-whep-player/iframe-form.html)

# Autoplay and Browser Restrictions

> This section intends to address playback issues you may see regarding audio.

Due to `autoplay` policy restrictions in browsers, it is required that any User visiting a page with a `video` or `audio` element declared with the `autoplay` attribute must arrive at that page through User Impetus - i.e., click on a link that directs the User to the page, or an element on the page that allows the `video` or `audio` element to begin playback.

This restriction means that any copy-and-paste of links into the URL bar of a browser will impede auto-playback, as will any page refreshes on which the autoplay is requested.

> You can read more about the policy in the following documentation: [https://developer.chrome.com/blog/autoplay/#mei](https://developer.chrome.com/blog/autoplay).

Because of such restrictions, the safest way to ensure auto playback of a live stream is to also include the `muted` attribute along `autoplay` on the Media Element... or allow the Red5 HTML SDK to handle such logic internally! Yes, it can do that.

## Red5 SDK Handling & Details

Using the `muteOnAutoplayRestriction` initialization configuration for the `WHEPClient`, you can define how you would like the Red5 HTML SDK to handle exceptions in playback requests with `autoplay` declared on your media element(s).

By setting `muteOnAutoplayRestriction` to `true` - which is the default - you are requesting the Red5 HTML SDK to handle any exceptions in the initial unmuted autoplay request. If an exception is thrown in the initial autoplay request, the Red5 Pro SDK will attempt to mute the media element and request playback again.

If the subsequent - and muted - playback is successful, a `Subscribe.Autoplay.Muted` event will be notified on the `WHEPClient`, allowing you - as the developer - to handle such a case as meets your webapp requirements; for example, displaying a call-out UI element notifying the end-user to unmute the audio.

If the subsequent request to playback as muted throws an exception, or if a failure happens at any point within the autoplay routine, the Red5 HTML SDK will dispatch a `Subscribe.Autoplay.Failure` event notification. Typically, this will result in not only audio being muted, but the video or audio stream is not auto-played at all in the media element. In such a scenario, the end-user will have to explicitly click the *play* button of the media element to begin playback. As a developer, you can respond to such an event to notify the end-user to take appropriate action.

Alternatively, setting `muteOnAutoplayRestriction` to `false` will let the Red5 HTML SDK know to not take any further action if the initial autoplay request throws an exception. If an exception is thrown, the `Subscribe.Autoplay.Failure` event notification will bubble from the `WHEPClient`.

## The Example

The following example demonstrates how to incorporate the `muteOnAutoplayRestriction` initialization configuration property into your webapp.

Take for example, that you have the following `video` element declared in your page:

```html
<video id="red5pro-subscriber" autoplay controls playsinline></video>
```

To tell the Red5 Pro HTML SDK that you would like it to handle autoplay and possible muting internally:

```js
const { WHEPClient } = red5

...

const handleAutoplayMuted = (event: SubscriberEvent) => {
  // Notify user to unmute audio, e.g., callout UI to click unmute.
}

const handleAutoplayFailure = (event: SubscriberEvent) => {
  // Notify user to click the Play button.
}

...

const config = getConfiguration()

subscriber = new WHEPClient()
subscriber.on(red5prosdk.SubscriberEventTypes.AUTO_PLAYBACK_MUTED, handleAutoplayMuted)
subscriber.on(red5prosdk.SubscriberEventTypes.AUTO_PLAYBACK_FAILURE, handleAutoplayFailure)

await subscriber.init(config)
await subscriber.subscribe()

...
```

After initialization of the `subscriber` and prior to a request to start subscribing to stream, two event handlers are defined to handle the `Subscribe.Autoplay.Muted` and `Subscribe.Autoplay.Failure` events (defined on the `SubscriberEventTypes` object as `AUTO_PLAYBACK_MUTED` and `AUTO_PLAYBACK_FAILURE`, respectively). We left out any sordid details on how to alert end-users of such notifications - so let your imaginations run wild!

# Extra Credit - Player UI

Because the Red5 HTML SDK is at its core a library to ease in negotiation and connection with the Red5 Server to stream live media, it does not provide any Custom UI and allows for the underlying generated `MediaStream` to be assigned to an HTML `video` (or `audio`) element directly.

This _does not_ mean that you have to keep the default browser chrome of the `video` player, however. In fact, because of its simplicity, it opens up the possibility of using whatever UI customization you prefer!

## Player UI & Styles

When thinking about displaying your own cutom Player UI for a `video` element that is playing back a live `MediaStream`, you would most likely want to think about state updates based on properties and events coming from the HTML `video` element itself. To understand what those events are and think about state (e.g., time display, audio mute, etc.), we invite you to refer to the [MDN documentation for video state and events](https://developer.mozilla.org/en-US/docs/Web/API/HTMLMediaElement).

While we encourage you to create your own Player UI and understand more about the underlying architecture of the HTML `video` and `audio` elements, there are many smart engineers out there who have also studied such things and made it easy for you to apply Player themes.

In fact, there is a site where you can browse themes that will match your webapp look-and-feel and easily apply to the `video` playback:

[https://player.style/?ref=producthunt&media=video&media=audio&framework=html](https://player.style/?ref=producthunt&media=video&media=audio&framework=html)

> The `index-styled.html` page included in this repo is an example of applying the [Sutro](https://player.style/themes/sutro) theme from the site above to the `video` element of a live playback!

Below is an example of applying a simple player theme on the `video` element with `MediaStream` live playback:

![Player UI](images/bbb.png)

```html
<script type="module" src="https://cdn.jsdelivr.net/npm/player.style/sutro/+esm"></script>
...
<media-theme-sutro style="width:100%">
    <video id="red5pro-subscriber" 
            class="red5pro-subscriber" 
            autoplay playsinline
            slot="media"
            crossorigin="anonymous"></video>
</media-theme-sutro>
```

### Note about live video and Playback UI

Due to the nature of live playback, there is no inherent concept/logic of scrubbing for `video` and `audio` elements in the browser. As such, if using any Player theme libraries out of the box, their progress bars will have a scrubber stuck at the beginning time and will not respond to user interaction.

We invite you to look at the default player UI chrome for each browser when streaming live video and notice the difference - not only between vendor implementations, but also live versus VOD UI elements.

As such, you may have to apply additional logic when it comes to UI display when using a 3rd-party Player Theme library.