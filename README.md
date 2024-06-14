# 🍎🎧 Apple Music authorization workflow

> ### ⚠️ Warnings
>***This documentation is for educational purposes only. Do not attempt to violate the Apple Media Services Terms and
Conditions and the Apple Developer Program License Agreement.***
> 
> This documentation is made by understing how the library work, it was not tested with various cases. If you have any problems, feel free to open an issue.

## 🔍 Overview

This is a repo that describes how MusicKit JS authorization works.

## ❗️ Required

To use the Apple Music authoriztion workflow, you need :

- A developper token for Apple Music. You can learn
  more [here](https://developer.apple.com/documentation/applemusicapi/generating_developer_tokens)

## 🔧 Steps

### Understanding the authorization url

Here's an exemple of an authorization URL :
`https://authorize.music.apple.com/woa?a=eyJ0aGlyZFBhcnR5SWNvblVSTCI6Imh0...&referrer=https://github.com/appleMusicLogin&app=music&p=subscribe`

The base URL is `https://authorize.music.apple.com/woa` with 4 parameters

| Name     | Value                                 | Decription                                                                       |
|----------|---------------------------------------|----------------------------------------------------------------------------------|
| a        | `eyJ0aGlyZFBhcnR5SWNvblVSTCI6Imh0...` | Base64 encoded ThirdPartyInfo() (i will explain it later)                        |
| referrer | `https://github.com/appleMusicLogin`  | URL of the window base URL                                                       |
| app      | `music`                               | Probably the app who need the authorization, but in our case it's always `music` |
| p        | `subscribe`                           | ? but always the same                                                            |

### Generate the Third Party Info

Third party info is a JSON generated by MusicKit to send data to the authorization page.
It contains :

- **thirdPartyIconURL :** The icon URL of the app
- **thirdPartyName :** The app name used in the authorization page
- **thirdPartyToken :** The developper token

Our JSON look like this :

```json
{
  "thirdPartyIconURL": "https://...",
  "thirdPartyName": "YOUR APP NAME",
  "thirdPartyToken": "ey..."
}
```

Then we need to stringify this JSON. I recommend you to write these as a function, we will call it later.
<br><br>**Javascript exemple :**

```javascript
const devToken = 'ey...'
let iconUrl = 'https://...'
let appName = 'YOUR APP NAME'

function getThirdPartyInfo() {
    return JSON.stringify({
        thirdPartyIconURL: iconUrl,
        thirdPartyName: appName,
        thirdPartyToken: devToken
    })
}
```

### Generate the authorization URL

The first things we need is to get third party info and encode it as base 64.

```javascript
let thirdPartyInfo = getThirdPartyInfo()
let b64thirdPartyInfo = btoa(thirdPartyInfo)
```

Then we have to generate the url search params.

```javascript
let params = {
    a: b64thirdPartyInfo,
    referrer: window.location.href,
    app: 'music',
    p: 'subscribe',
}
```

And then generate the url.

```javascript
const base_url = 'https://authorize.music.apple.com/woa?'
let params = new URLSearchParams(params).toString()
return base_url + params
```

### Open an authentication window

To test your URL, you can paste it in your browser but, you can see the referrer is never used. The Authorization flow *
*need to be opened from a parent to get the token**.

You can open a new window with [window.open()](https://developer.mozilla.org/fr/docs/Web/API/Window/open).
Save the new window to a variable, next we have to set an event listener.

```javascript
//The size used is the default from MusicKit
let authWindow = window.open(url, 'apple-music-service-view', 'height=650,menubar=no,resizable=no,scrollbars=no,status=no,toolbar=no,width=650')
```

### Add event listener

Next we have to add a listener on message. If you don't, you can't see the authorization form from Apple Music.

```javascript
window.addEventListener('message', (event) => { //Add the event listener
    if (event.source === authWindow) { //Check if the message is from our auth window
        console.log(event)//log the event
    }
})
```

Each message from the authorization window contains data. It contains id, jsonrpc, method, and sometimes params.

Here's the avaible method from the MusicKit v3 :

```javascript
{
    methods: {
        authorize(e, n, p) {
            validateToken(e) ? d({restricted: n && "1" === n, userToken: e, cid: p}) : y(0);
        },
        close(){},
        decline() {
            y(1); //y is the returned value
        },
        switchUserId() {
            y(0);
        },
        thirdPartyInfo: () => n._thirdPartyInfo(n.developerToken, _object_spread$J({}, n.deeplinkParameters, e)), 
        unavailable() {
            y(-1);
        }
    }
}
```

#### thirdPartyInfo
So the first event appear is the `thirdPartyInfo` method.
These methods only return the string of our `getThirdPartyInfo()`.
The response is made like this :
```json
{"id": 0, "jsonrpc": "2.0", "result": "{...}"}
```
The `id` and `jsonrpc` are the same as the data from message, and result is the return of `getThirdPartyInfo()`.
```javascript
if (msg.data.method === 'thirdPartyInfo') {
  authWindow.postMessage({id: msg.data.id, jsonrpc: msg.data.jsonrpc, result: getThirdPartyInfo()}, '*')
}
```
#### unavailable
This event is called when the authorization is unavailable. You don't need to do anything, the window will close itself.

### switchUserId
This event is unkown ! Please contribute to this documentation if you know what it does.

### decline
This event is called when the authorization is declined. You don't need to do anything, the window will close itself.

### close
This event is called when the window is closed. You don't need to respond anything.

### authorize
This event is called when the authorization is accepted. You can get the user token from the parameters of event.

| index | description                                             |
|-------|---------------------------------------------------------|
| 0     | Returned token from authorization                       |
| 1     | named `restricted` by MusicKit, if value is 1 it's true |
| 2     | named `cip` by MusicKit but it's unkown                 |

You can get the user token like this :

```javascript
if (msg.data.method === 'authorize') {
  let userToken = msg.data.params[0]
  console.log(userToken)
}
```

## 👀 Learn more about MusicKit JS

MusicKit docs is avaible at [this link](https://js-cdn.music.apple.com/musickit/v1/index.html). If you plan to use advanced features, you can add it to your project with this tag.
```html
<script src="musickit.js" data-web-components async></script>
```

The code is minified, but you can use tool like [unminify](https://unminify.com/) to unminify it.

## 📄 Contributing

If you have any questions or want to contribute to this documentation, feel free to open an issue or a pull request.
