# TikTok-Livestream-Chat-Connector
A Node.js module to receive and decode livestream events such as comments and gifts in realtime from [TikTok LIVE](https://www.tiktok.com/live) by connecting to TikTok's internal WebCast push service. The package includes a wrapper that connects to the WebCast service using just the username (`uniqueId`). This allows you to connect to your own live chat as well as the live chat of other streamers. No credentials are required. Besides [Chat Comments](#chat), other events such as [Members Joining](#member), [Gifts](#gift), [Viewers](#roomuser), [Follows](#social), [Shares](#social), [Questions](#questionnew), [Likes](#like) and [Battles](#linkmicbattle) can be tracked.

Looking for a Python implementation of this library? Check out [TikTokLive](https://github.com/isaackogan/TikTokLive) by [@isaackogan](https://github.com/isaackogan)

**NOTE:** This is not an official API. It's a reverse engineering project.

### Demo: [https://tiktok-chat-reader.zerody.one/](https://tiktok-chat-reader.zerody.one/)

#### Overview
- [Getting started](#getting-started)
- [Params and options](#params-and-options)
- [Methods](#methods)
- [Events](#events)
- [Examples](#examples)
- [Contributing](#contributing)

## Getting started

1. Install the package via NPM
```
npm i tiktok-livestream-chat-connector
```

2. Create your first chat connection

```javascript
const { WebcastPushConnection } = require('tiktok-livestream-chat-connector');

// Username of someone who is currently live
let tiktokUsername = "officialgeilegisela";

// Create a new wrapper object and pass the username
let tiktokChatConnection = new WebcastPushConnection(tiktokUsername);

// Connect to the chat (await can be used as well)
tiktokChatConnection.connect().then(state => {
    console.info(`Connected to roomId ${state.roomId}`);
}).catch(err => {
    console.error('Failed to connect', err);
})

// Define the events that you want to handle
// In this case we listen to chat messages (comments)
tiktokChatConnection.on('chat', data => {
    console.log(`${data.uniqueId} (userId:${data.userId}) writes: ${data.comment}`);
})

// And here we receive gifts sent to the streamer
tiktokChatConnection.on('gift', data => {
    console.log(`${data.uniqueId} (userId:${data.userId}) sends ${data.giftId}`);
})

// ...and more events described in the documentation below
```

## Params and options

To create a new `WebcastPushConnection` object the following parameters are required.

`WebcastPushConnection(uniqueId, [options])`

| Param Name | Required | Description |
| ---------- | -------- | ----------- |
| uniqueId   | Yes | The unique username of the broadcaster. You can find this name in the URL.<br>Example: `https://www.tiktok.com/@officialgeilegisela/live` => `officialgeilegisela` |
| options  | No | Here you can set the following optional connection properties. If you do not specify a value, the default value will be used.<br><br>`processInitialData` (default: `true`) <br> Define if you want to process the initital data which includes old messages of the last seconds.<br><br>`fetchRoomInfoOnConnect` (default: `true`) <br> Define if you want to fetch all room information on [`connect()`](#methods). If this option is enabled, the connection to offline rooms will be prevented. If enabled, the connect result contains the room info via the `roomInfo` attribute. You can also manually retrieve the room info (even in an unconnected state) using the [`getRoomInfo()`](#methods) function.<br><br>`enableExtendedGiftInfo` (default: `false`) <br> Define if you want to receive extended information about gifts like gift name, cost and images. This information will be provided at the [gift event](#gift). <br><br>`enableWebsocketUpgrade` (default: `true`) <br> Define if you want to use a WebSocket connection instead of request polling if TikTok offers it. <br><br>`requestPollingIntervalMs` (default: `1000`) <br> Request polling interval if WebSocket is not used.<br><br>`clientParams` (default: `{}`) <br> Custom client params for Webcast API.<br><br>`requestHeaders` (default: `{}`) <br> Custom request headers passed to [axios](https://github.com/axios/axios).<br><br>`websocketHeaders` (default: `{}`) <br> Custom websocket headers passed to [websocket.client](https://github.com/theturtle32/WebSocket-Node). <br><br>`requestOptions` (default: `{}`) <br> Custom request options passed to [axios](https://github.com/axios/axios). Here you can specify an `httpsAgent` to use a proxy and a `timeout` value. See [Example](#connect-via-proxy). <br><br>`websocketOptions` (default: `{}`) <br> Custom websocket options passed to [websocket.client](https://github.com/theturtle32/WebSocket-Node). Here you can specify an `agent` to use a proxy and a `timeout` value. See [Example](#connect-via-proxy). |

Example Options:
```javascript
let tiktokChatConnection = new WebcastPushConnection(tiktokUsername, {
    processInitialData: false,
    enableExtendedGiftInfo: true,
    enableWebsocketUpgrade: true,
    requestPollingIntervalMs: 2000,
    clientParams: {
        "app_language": "en-US",
        "device_platform": "web"
    },
    requestHeaders: {
        "headerName": "headerValue"
    },
    websocketHeaders: {
        "headerName": "headerValue"
    },
    requestOptions: {
        timeout: 10000
    },
    websocketOptions: {
        timeout: 10000
    }
});
```

## Methods
A `WebcastPushConnection` object contains the following methods.

| Method Name | Description |
| ----------- | ----------- |
| connect     | Connects to the live stream chat.<br>Returns a `Promise` which will be resolved when the connection is successfully established. |
| disconnect  | Disconnects the connection. |
| getState    | Gets the current connection state including the cached room info (see below). |
| getRoomInfo | Gets the current room info from TikTok API including streamer info, room status and statistics.<br>Returns a `Promise` which will be resolved when the API request is done.<br>*<b>Note: </b>You can call this function even if you're not connected.*<br>[Example](#retrieve-room-info) |
| getAvailableGifts | Gets a list of all available gifts including gift name, image url, diamont cost and a lot of other information.<br>Returns a `Promise` that will be resolved when all available gifts has been retrieved from the API.<br>*<b>Note: </b>You can call this function even if you're not connected.*<br>[Example](#retrieve-available-gifts) |

## Events

A `WebcastPushConnection` object has the following events which can be handled via `.on(eventName, eventHandler)`

Control Events:
- [`connected`](#connected)
- [`disconnected`](#disconnected)
- [`streamEnd`](#streamend)
- [`rawData`](#rawdata)
- [`websocketConnected`](#websocketconnected)
- [`error`](#error)

Message Events:
- [`member`](#member)
- [`chat`](#chat)
- [`gift`](#gift)
- [`roomUser`](#roomuser)
- [`like`](#like)
- [`social`](#social)
- [`questionNew`](#questionnew)
- [`linkMicBattle`](#linkmicbattle)
- [`linkMicArmies`](#linkmicarmies)
- [`liveIntro`](#liveintro)

### Control Events

#### `connected`
Triggered when the connection gets successfully established.

```javascript
tiktokChatConnection.on('connected', state => {
    console.log('Hurray! Connected!', state);
})
```

#### `disconnected`
Triggered when the connection gets disconnected. In that case you can call `connect()` again to have a reconnect logic. Note that you should wait a little bit before attempting a reconnect to to avoid being rate-limited.

```javascript
tiktokChatConnection.on('disconnected', () => {
    console.log('Disconnected :(');
})
```


#### `streamEnd`
Triggered when the live stream gets terminated by the host. Will also trigger the [`disconnected`](#disconnected) event.

```javascript
tiktokChatConnection.on('streamEnd', () => {
    console.log('Stream ended');
})
```

#### `rawData`
Triggered every time a protobuf encoded webcast message arrives. You can deserialize the binary object depending on the use case with <a href="https://www.npmjs.com/package/protobufjs">protobufjs</a>.

```javascript
tiktokChatConnection.on('rawData', (messageTypeName, binary) => {
    console.log(messageTypeName, binary);
})
```

#### `websocketConnected`
Will be triggered as soon as a websocket connection is established. The websocket client object is passed.

```javascript
tiktokChatConnection.on('websocketConnected', websocketClient => {
    console.log("Websocket:", websocketClient.connection);
})
```

#### `error`
General error event. You should handle this.

```javascript
tiktokChatConnection.on('error', err => {
    console.error('Error!', err);
})
```


### Message Events

#### `member`
Triggered every time a new viewer joins the live stream.

```javascript
tiktokChatConnection.on('member', data => {
    console.log(`${data.uniqueId} joins the stream!`);
})
```


Data structure:
```javascript
{
  userId: '6776663624629974021',
  uniqueId: 'zerodytester',
  nickname: 'Zerody One',
  profilePictureUrl: 'https://p16-sign-va.tiktokcdn.com/...',
  followRole: 1, // 0 = none; 1 = follower; 2 = friends
  userBadges: [] // e.g. Moderator badge
}
```

#### `chat`
Triggered every time a new chat comment arrives.

```javascript
tiktokChatConnection.on('chat', data => {
    console.log(`${data.uniqueId} writes: ${data.comment}`);
})
```

Data structure:
```javascript
{
    comment: 'how are you?',
    userId: '6776663624629974121',
    uniqueId: 'zerodytester',
    nickname: 'Zerody One',
    profilePictureUrl: 'https://p16-sign-va.tiktokcdn.com/...',
    followRole: 2, // 0 = none; 1 = follower; 2 = friends
    userBadges: [
        {
            type: 'pm_mt_moderator_im', 
            name: 'Moderator'
        }
    ]
}
```

#### `gift`
Triggered every time a gift arrives. You will receive additional information via the `extendedGiftInfo` attribute when you enable the [`enableExtendedGiftInfo`](#params-and-options) option. 

> **NOTE:** Users have the capability to send gifts in a streak. This increases the `data.gift.repeat_count` value until the user terminates the streak. During this time new gift events are triggered again and again with an increased `data.gift.repeat_count` value. It should be noted that after the end of the streak, another gift event is triggered, which signals the end of the streak via `data.gift.repeat_end`:`1`. This applies only to gifts with `data.gift.gift_type`:`1`. This means that even if the user sends a `gift_type`:`1` gift only once, you will receive the event twice. Once with `repeat_end`:`0` and once with `repeat_end`:`1`. Therefore, the event should be handled as follows:


```javascript
tiktokChatConnection.on('gift', data => {
    if (data.gift.gift_type === 1 && data.gift.repeat_end === 0) {
        // Streak in progress => show only temporary
        console.log(`${data.uniqueId} is sending gift ${data.giftId} x${data.gift.repeat_count}`);
    } else {
        // Streak ended or non-streakable gift => process the gift with final repeat_count
        console.log(`${data.uniqueId} has sent gift ${data.giftId} x${data.gift.repeat_count}`);
    }
})
```


Data structure:
```javascript
{
  userId: '6776663624629974121',
  uniqueId: 'zerodytester',
  nickname: 'Zerody One',
  profilePictureUrl: 'https://p16-sign-va.tiktokcdn.com/...',
  gift: {
    gift_id: 5729,
    gift_type: 2,
    repeat_count: 1,
    repeat_end: 0,
    to_user_id: 6929592145315251000
  },
  giftId: 5729,
  // Extended info is present if you have enabled the 'enableExtendedGiftInfo' option
  extendedGiftInfo: {
    describe: 'sent Rose',
    diamond_count: 1,
    duration: 1000,
    icon: {
      avg_color: '#A3897C',
      is_animated: false,
      url_list: [
        // Icon URLs...
      ]
    },
    id: 5655,
    image: {
      avg_color: '#FFEBEB',
      is_animated: false,
      url_list: [
        // Image URLs...
      ]
    },
    is_broadcast_gift: false,
    is_displayed_on_panel: true,
    is_effect_befview: false,
    item_type: 1,
    name: 'Rose',
    type: 1
  }
}
```

#### `roomUser`
Triggered every time a statistic message arrives. This message currently contains only the viewer count.

```javascript
tiktokChatConnection.on('roomUser', data => {
    console.log(`Viewer Count: ${data.viewerCount}`);
})
```

#### `like`
Triggered when a viewer sends likes to the streamer. For streams with many viewers, this event is not always triggered by TikTok.

```javascript
tiktokChatConnection.on('like', data => {
    console.log(`${data.uniqueId} sent ${data.likeCount} likes, total likes: ${data.totalLikeCount}`);
})
```

Data structure:
```javascript
{
  likeCount: 5, // likes given by the user (taps on screen)
  totalLikeCount: 83033, // likes that this stream has received in total
  userId: '6776663624629974121',
  uniqueId: 'zerodytester',
  nickname: 'Zerody One',
  profilePictureUrl: 'https://p16-sign-sg.tiktokcdn.com/...',
  displayType: 'pm_mt_msg_viewer',
  label: '{0:user} sent likes to the host'
}
```

#### `social`
Triggered every time someone shares the stream or follows the host.

```javascript
tiktokChatConnection.on('social', data => {
    console.log('social event data:', data);
})
```

Data structure:
```javascript
{
  userId: '6776663624629974121',
  uniqueId: 'zerodytester',
  nickname: 'Zerody One',
  profilePictureUrl: 'https://p16-sign-va.tiktokcdn.com/...',
  displayType: 'pm_main_follow_message_viewer_2', // or 'pm_mt_guidance_share' for sharing
  label: '{0:user} followed the host'
}
```

#### `questionNew`
Triggered every time someone asks a new question via the question feature.

```javascript
tiktokChatConnection.on('questionNew', data => {
    console.log(`${data.uniqueId} asks ${data.questionText}`);
})
```

Data structure:
```javascript
{
  questionText: 'Do you know why TikTok has such a complicated API?',
  userId: '6776663624629974121',
  uniqueId: 'zerodytester',
  nickname: 'Zerody One',
  profilePictureUrl: 'https://p16-sign-va.tiktokcdn.com/...'
}
```

#### `linkMicBattle`
Triggered every time a battle starts.

```javascript
tiktokChatConnection.on('linkMicBattle', (data) => {
    console.log(`New Battle: ${data.battleUsers[0].uniqueId} VS ${data.battleUsers[1].uniqueId}`);
})
```

Data structure:
```javascript
{
    battleUsers: [
        {
            userId: '6761609734837650437', // Host
            uniqueId: 'haje_bahjat',
            nickname: '𝙃𝙖𝙟𝙚_𝙗𝙖𝙝𝙟𝙖𝙩',
            profilePictureUrl: 'https://p77-sign-sg.tiktokcdn.com/...'
        },
        {
            userId: '6994367558246597637', // Guest
            uniqueId: 'aborayanelzidicomedy',
            nickname: 'ابو ريان الايزيدي الكوميدي',
            profilePictureUrl: 'https://p16-sign-va.tiktokcdn.com/....'
        }
    ]
}
```

#### `linkMicArmies`
Triggered every time a battle participant receives points. Contains the current status of the battle and the army that suported the group.

```javascript
tiktokChatConnection.on('linkMicArmies', (data) => {
    console.log('linkMicArmies', data);
})
```

Data structure:
```javascript
{
    "battleStatus": 1, // 1 = running; 2 = final state
    "battleArmies": [
        {
            "hostUserId": "6761609734837650437", // Streamer Host ID
            "points": 17058,
            "participants": [ // Top 3 supporters
                {
                    "userId": "6809941952760742917",
                    "nickname": "Abdulaziz Slivaney"
                },
                {
                    "userId": "7062835930043139078",
                    "nickname": "Dilschad Amedi"
                },
                {
                    "userId": "6773657511493977093",
                    "nickname": "Kahin Guli"
                }
            ]
        },
        {
            "hostUserId": "6994367558246597637", // Streamer Guest ID
            "points": 6585,
            "participants": [
                {
                    "userId": "7060878425477792773",
                    "nickname": "ADAM"
                },
                {
                    "userId": "7048005772659328006",
                    "nickname": "كلو"
                },
                {
                    "userId": "6818014975869699078",
                    "nickname": "Karwan###"
                }
            ]
        }
    ]
}
```

#### `liveIntro`
Triggered when a live intro message appears.

```javascript
tiktokChatConnection.on('liveIntro', (msg) => {
    console.log(msg);
})
```

## Examples
### Retrieve Room Info
````javascript
let tiktokChatConnection = new WebcastPushConnection('@username');

tiktokChatConnection.getRoomInfo().then(roomInfo => {
    console.log(roomInfo);
    console.log(`Stream started timestamp: ${roomInfo.create_time}, Streamer bio: ${roomInfo.owner.bio_description}`);
    console.log(`HLS URL: ${roomInfo.stream_url.hls_pull_url}`); // Can be played or recorded with e.g. VLC
}).catch(err => {
    console.error(err);
})
````

### Retrieve Available Gifts
````javascript
let tiktokChatConnection = new WebcastPushConnection('@username');

tiktokChatConnection.getAvailableGifts().then(giftList => {
    console.log(giftList);
    giftList.forEach(gift => {
        console.log(`id: ${gift.id}, name: ${gift.name}, cost: ${gift.diamond_count}`)
    });
}).catch(err => {
    console.error(err);
})
````

### Connect via Proxy
[proxy-agent](https://www.npmjs.com/package/proxy-agent) supports `http`, `https`, `socks4` and `socks5` proxies:
````
npm i proxy-agent
````
You can specify if you want to use a proxy for https requests, websockets or both:
````javascript
const { WebcastPushConnection } = require('tiktok-livestream-chat-connector');
const ProxyAgent = require('proxy-agent');

let tiktokChatConnection = new WebcastPushConnection('@username', {
    requestOptions: {
        httpsAgent: new ProxyAgent('https://username:password@host:port'),
        timeout: 10000 // 10 seconds
    },
    websocketOptions: {
        agent: new ProxyAgent('https://username:password@host:port'),
        timeout: 10000 // 10 seconds
    }
});

// Connect as usual
````


## Contributing
Your improvements are welcome! Feel free to open an <a href="https://github.com/zerodytrash/TikTok-Livestream-Chat-Connector/issues">issue</a> or <a href="https://github.com/zerodytrash/TikTok-Livestream-Chat-Connector/pulls">pull request</a>.
