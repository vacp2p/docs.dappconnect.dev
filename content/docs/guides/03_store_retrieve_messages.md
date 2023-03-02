---
title: Retrieve Messages Using Waku Store
date: 2021-12-09T14:00:00+01:00
weight: 30
---

# Retrieve Messages Using Waku Store

DApps running on a phone or in a browser are often offline:
The browser could be closed or mobile app in the background.

[Waku Relay](https://rfc.vac.dev/spec/11/) is a gossip protocol.
As a user, it means that your peers forward you messages they just received.
If you cannot be reached by your peers, then messages are not relayed;
relay peers do **not** save messages for later.

However, [Waku Store](https://rfc.vac.dev/spec/13/) peers do save messages they relay,
allowing you to retrieve them at a later time.
The Waku Store protocol is best-effort and does not guarantee data availability.
Waku Relay should still be preferred when online;
Waku Store can be used after resuming connectivity:
For example, when the dApp starts.

In this guide, we'll review how you can use Waku Store to retrieve messages.

Before starting, you need to choose a _Content Topic_ for your dApp.
Check out the [how to choose a content topic guide](/docs/guides/01_choose_content_topic/) to learn more about content topics.

For this guide, we are using a single content topic: `/store-guide/1/news/proto`.

# Installation

You can install [js-waku](https://npmjs.com/package/js-waku) using your favorite package manager:

```shell
npm install js-waku
```

# Create Waku Instance

In order to interact with the Waku network, you first need a Waku instance:

```js
import { Waku } from "js-waku";

const wakuNode = await Waku.create({ bootstrap: { default: true } });
```

Passing the `bootstrap` option will connect your node to predefined Waku nodes.
If you want to bootstrap to your own nodes, you can pass an array of multiaddresses instead:

```js
import { Waku } from "js-waku";

const wakuNode = await Waku.create({
  bootstrap: {
    peers: [
      "/dns4/node-01.ac-cn-hongkong-c.wakuv2.test.statusim.net/tcp/443/wss/p2p/16Uiu2HAkvWiyFsgRhuJEb9JfjYxEkoHLgnUQmr1N5mKWnYjxYRVm",
      "/dns4/node-01.do-ams3.wakuv2.test.statusim.net/tcp/443/wss/p2p/16Uiu2HAmPLe7Mzm8TsYUubgCAW1aJoeFScxrLj8ppHFivPo97bUZ",
    ],
  },
});
```

# Wait to be connected

When using the `bootstrap` option, it may take some times to connect to other peers.
To ensure that you have store peers available to retrieve historical messages from,
use the following function:

```js
await waku.waitForRemotePeer();
```

The returned Promise will resolve once you are connected to a Waku Store peer.

# Use Protobuf

Waku v2 protocols use [protobuf](https://developers.google.com/protocol-buffers/) [by default](https://rfc.vac.dev/spec/10/).

Let's review how you can use protobuf to send structured data.

First, define a data structure.
For this guide, we will use a simple news article that contains a date of publication, title and body:

```js
{
  date: Date;
  title: string;
  body: string;
}
```

To encode and decode protobuf payloads, you can use the [protons](https://www.npmjs.com/package/protons) package.

## Install Protobuf Library

First, install protons:

```shell
npm install protons
```

## Protobuf Definition

Then specify the data structure:

```js
import protons from "protons";

const proto = protons(`
message ArticleMessage {
  uint64 date = 1;
  string title = 2;
  string body = 3;
}
`);
```

You can learn about protobuf message definitions here:
[Protocol Buffers Language Guide](https://developers.google.com/protocol-buffers/docs/proto).

## Decode Messages

To decode the messages retrieved from a Waku Store node,
you need to extract the protobuf payload and decode it using `protons`.

```js
const decodeWakuMessage = (wakuMessage) => {
  // No need to attempt to decode a message if the payload is absent
  if (!wakuMessage.payload) return;

  const { date, title, body } = proto.SimpleChatMessage.decode(
    wakuMessage.payload
  );

  // In protobuf, fields are optional so best to check
  if (!date || !title || !body) return;

  const publishDate = new Date();
  publishDate.setTime(date);

  return { publishDate, title, body };
};
```

## Retrieve messages

You now have all the building blocks to retrieve and decode messages for a store node.

Store node responses are paginated.
The `WakuStore.queryHistory` API automatically query all the pages in a sequential manner.
To process messages as soon as they received (page by page), use the `callback` option:

```js
const ContentTopic = "/store-guide/1/news/proto";

const callback = (retrievedMessages) => {
  const articles = retrievedMessages
    .map(decodeWakuMessage) // Decode messages
    .filter(Boolean); // Filter out undefined values

  console.log(`${articles.length} articles have been retrieved`);
};

waku.store.queryHistory([ContentTopic], { callback }).catch((e) => {
  // Catch any potential error
  console.log("Failed to retrieve messages from store", e);
});
```

Note that `WakuStore.queryHistory` select an available store node for you.
However, it can only select a connected node, which is why the bootstrapping is necessary.
It will throw an error if no store node is available.

## Filter messages by send time

By default, Waku Store nodes store messages for 30 days.
Depending on your use case, you may not need to retrieve 30 days worth of messages.

[Waku Message](https://rfc.vac.dev/spec/14/) defines an optional unencrypted `timestamp` field.
The timestamp is set by the sender.
By default, js-waku sets the timestamp of outgoing message to the current time.

You can filter messages that include a timestamp within given bounds with the `timeFilter` option.

Retrieve messages up to a week old:

```js
// [..] `ContentTopic` and `callback` definitions

const startTime = new Date();
// 7 days/week, 24 hours/day, 60min/hour, 60secs/min, 100ms/sec
startTime.setTime(startTime.getTime() - 7 * 24 * 60 * 60 * 1000);

waku.store
  .queryHistory([ContentTopic], {
    callback,
    timeFilter: { startTime, endTime: new Date() },
  })
  .catch((e) => {
    console.log("Failed to retrieve messages from store", e);
  });
```

## End result

You can see a similar example implemented in ReactJS in the [Minimal ReactJS Waku Store App](https://github.com/status-im/js-waku/tree/main/examples/store-reactjs-chat).
