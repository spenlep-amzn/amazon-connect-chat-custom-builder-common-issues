# ChatJS Issue - End-customer chat loses connection, Agent messages fail to send

## Description

During an ongoing chat between Agent and Customer, the customer (browser client) loses websocket/network connection. The Agent successfully sends a message (recorded in the CTR/Chat transcript records), but it fails to appear in the customer chat UI.

## Explanation

If the **end-customer chat UI** loses websocket/network connection, it must:

1. Re-establish the websocket connection (to receive future incoming messages agin)
2. Make a [chatSession.getTranscript](https://github.com/amazon-connect/amazon-connect-chatjs?tab=readme-ov-file#chatsessiongettranscript) (API: [ GetTranscript](https://docs.aws.amazon.com/connect-participant/latest/APIReference/API_GetTranscript.html)) request (to retrieve all missing messages that were sent while end-customer was disconnected)

If the agent sends a message while the end-customer chat UI is disconnected, it is successfully stored in the Connect backend (CCP is working as expected, messages are all recorded in transcript), but the client's device was unable to receive it. When the client reconnects to the websocket connection, there is a gap in messages. Future incoming messages will appear again from the websocket, but the gap messages are still missing unless the code explicitly makes a [GetTranscript](https://docs.aws.amazon.com/connect-participant/latest/APIReference/API_GetTranscript.html) call.

## Solution

The [chatSession.onConnectionEstablished](https://github.com/amazon-connect/amazon-connect-chatjs?tab=readme-ov-file#chatsessiononconnectionestablished) event handler may help, which is triggered when websocket re-connects. ChatJS has built-in heartbeat and retry logic for the websocket connection. Since ChatJS is not storing the transcript though, **Custom chat UI** code must manually fetch the transcript again

> CX could refer to our open source implementation, although it has additional logic: https://github.com/amazon-connect/amazon-connect-chat-interface/blob/c88f854073fe6dd45546585c3bfa363d3659d73f/src/components/Chat/ChatSession.js#L408

```js
import "amazon-connect-chatjs";

const chatSession = connect.ChatSession.create({
  chatDetails: {
    ContactId: "abc",
    ParticipantId: "cde",
    ParticipantToken: "efg",
  },
  type: "CUSTOMER",
  options: { region: "us-west-2" },
});

chatSession.onConnectionEstablished(() => {
  chatSession.getTranscript({
    scanDirection: "BACKWARD",
    sortOrder: "ASCENDING",
    maxResults: 15,
    // nextToken?: nextToken - OPTIONAL, for pagination
  })
    .then((response) => {
      const { initialContactId, nextToken, transcript } = response.data;
      // ...
    })
    .catch(() => {})
});
```

```js
function loadLatestTranscript(args) {
    // Documentation: https://github.com/amazon-connect/amazon-connect-chatjs?tab=readme-ov-file#chatsessiongettranscript
    return chatSession.getTranscript({
        scanDirection: "BACKWARD",
        sortOrder: "ASCENDING",
        maxResults: 15,
        // nextToken?: nextToken - OPTIONAL, for pagination
      })
      .then((response) => {
        const { initialContactId, nextToken, transcript } = response.data;
        
        const exampleMessageObj = transcript[0];
        const {
          DisplayName,
          ParticipantId,
          ParticipantRole, // CUSTOMER, AGENT, SUPERVISOR, SYSTEM
          Content,
          ContentType,
          Id,
          Type,
          AbsoluteTime, // sentTime = new Date(item.AbsoluteTime).getTime() / 1000
          MessageMetadata, // { Receipts: [{ RecipientParticipantId: "asdf" }] }
          Attachments,
          RelatedContactid,
        } = exampleMessageObj;

        return transcript // TODO - store the new transcript somewhere
      })
      .catch((err) => {
        console.log("CustomerUI", "ChatSession", "transcript fetch error: ", err);
      });
}
```

## Further Explanation

Related event listeners:

- [onConnectionLost](TODO): fired when ongoing websocket connection is closed (network goes offline)
- [onConnectionBroken](TODO): agent/customer has ended a chat
- [onConnectionEstablished](TODO): websocket has successfully connected or re-connected
- [onConnectionEnded](TODO): fail to connect to websocket when initializing
