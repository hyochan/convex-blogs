# Building Real-Time AI Chat with Convex Components

**Imagine building a ChatGPT-like streaming experience without wrestling with WebSockets or SSE protocols.** With Convex Components, this becomes surprisingly simple. Let me show you how **Convex Components** revolutionize the way we build real-time features.

If you’re a **developer**, you might already have a sense of what this means. The concept is similar to the component architecture used in frontend frameworks like React, Vue, and Angular, and it also resembles modular units found in backend frameworks like Spring or NestJS.

Convex goes beyond being just another server framework—it **abstracts the entire client-server structure into modular, reusable components**.

In other words, even backend logic is provided as **declarative, reusable components**, which is one of Convex’s most powerful features.

<img src="https://patient-pigeon-633.convex.cloud/api/storage/f17b82bc-8639-461b-b302-d97ad7d17e9c" alt="1" style="height: 360px; width: auto; max-width: 100%;">

> Source: [Convex Components](https://www.convex.dev/components)

From the official site, you can browse through various Convex Components.

For this tutorial, we’ll focus on `@convex-dev/persistent-text-streaming`. Unlike traditional APIs that return the full response at once, this component enables **real-time streaming output**, similar to how ChatGPT streams its responses.

Streaming provides a **faster and more interactive user experience**, as users can start seeing partial responses immediately instead of waiting for the entire message to complete.

Implementing this manually is non-trivial—doing so would usually require protocols like SSE or WebSockets, adding complexity. With Convex’s built-in component, you can **skip the low-level implementation and quickly enable streaming functionality** in your app.

<img src="https://patient-pigeon-633.convex.cloud/api/storage/1c56d398-736b-4fd4-a08d-7eadf4ab5176" alt="2" style="height: 480px; width: auto; max-width: 100%;">

> Source: [Persistent Text Streaming Component](https://www.convex.dev/components/persistent-text-streaming)
>
> The component interface shows how simple it is to implement streaming - just install, configure, and use the provided hooks.

To demonstrate this in action, we'll use [Convex Chef](https://chef.convex.dev/) to quickly spin up an AI chat feature. The complete code for this tutorial is available at [https://github.com/hyochan/ai-chat](https://github.com/hyochan/ai-chat).

<div style="display: flex; width: 100%; gap: 0;">
  <div style="flex: 1;">
    <img src="https://patient-pigeon-633.convex.cloud/api/storage/65103061-ee68-4939-853b-fcce740cf7ca" style="width: 100%; height: 280px;">
  </div>
  <div style="flex: 1;">
    <img src="https://patient-pigeon-633.convex.cloud/api/storage/3d08e5a2-9c64-4bbd-abc0-e1b897467a6d" style="width: 100%; height: 280px;">
  </div>
</div>

After entering a prompt and waiting briefly, a fully functional React application is automatically generated. Notice how the generated app includes a chat interface ready for our streaming enhancement:

<div style="display: flex; width: 100%; gap: 0;">
  <div style="flex: 1;">
    <img src="https://patient-pigeon-633.convex.cloud/api/storage/1ad7b3c9-dfbd-46b3-8135-aa799d337de3" style="height: 280px; width: auto; max-width: 100%;">
  </div>
  <div style="flex: 1;">
    <img src="https://patient-pigeon-633.convex.cloud/api/storage/2b6c1008-85aa-4a50-ba2d-746e4f0107e5" style="height: 280px; width: auto; max-width: 100%;">
  </div>
</div>

By clicking **"Sign in anonymously"**, you can start chatting with AI. The interface shows messages in a traditional request-response pattern:

Currently, streaming is not yet enabled, so responses are delivered in full rather than progressively. Let’s integrate `@convex-dev/persistent-text-streaming` to enable real-time streaming functionality.

---

## 1. Setting up Persistent Text Streaming

### Install the Package

```bash
npm install @convex-dev/persistent-text-streaming
```

### Register the Component in Convex

```ts
import { defineApp } from "convex/server";
import persistentTextStreaming from "@convex-dev/persistent-text-streaming/convex.config";

const app = defineApp();
app.use(persistentTextStreaming);
export default app;
```

As described in the [Convex documentation](https://docs.convex.dev/api/modules/server#defineapp), `defineApp()` is a beta API that lets you define your Convex app by attaching reusable components. This allows you to modularly add functionalities like authentication, payments, or chat without manually setting up `defineSchema` and `defineFunctions`. Since it’s experimental, use it cautiously in production environments.

### Set Up an HTTP Route

```ts
import { httpRouter } from "convex/server";
import { streamChat } from "./chat";

const http = httpRouter();

http.route({
  path: "/chat-stream",
  method: "POST",
  handler: streamChat,
});

export default http;
```

The `streamChat` handler accepts POST requests to `/chat-stream` and enables real-time text streaming. This endpoint will be used to stream AI responses back to the client.

---

## 2. Backend: Implementing Real-Time Streaming

### Basic Streaming Pattern

Here’s a simple example that streams text one character at a time:

```ts
import { PersistentTextStreaming } from "@convex-dev/persistent-text-streaming";
import { StreamId } from "@convex-dev/persistent-text-streaming";
import { components } from "./_generated/api";
import { httpAction } from "./_generated/server";

const persistentTextStreaming = new PersistentTextStreaming(
  components.persistentTextStreaming
);

export const streamChat = httpAction(async (ctx, request) => {
  const body = (await request.json()) as {
    streamId: string;
    conversationId: string;
    userMessage: string;
  };

  const generateChat = async (ctx, request, streamId, chunkAppender) => {
    try {
      const message = "Hello! How can I help you today?";

      // Stream the response character by character
      for (let i = 0; i < message.length; i++) {
        await chunkAppender(message[i]);
        await new Promise((resolve) => setTimeout(resolve, 50));
      }
    } catch (error) {
      console.error("Chat generation error:", error);
      await chunkAppender("Sorry, an error occurred.");
    }
  };

  const response = await persistentTextStreaming.stream(
    ctx,
    request,
    body.streamId as StreamId,
    generateChat
  );

  // Set CORS headers
  response.headers.set("Access-Control-Allow-Origin", "*");
  response.headers.set("Vary", "Origin");
  return response;
});
```

- `PersistentTextStreaming` initializes the streaming component.
- `generateChat` function receives a `chunkAppender` to send text incrementally.
- `persistentTextStreaming.stream()` handles the HTTP streaming response.
- CORS headers are set for cross-origin requests.

### Complete Streaming Implementation

Here's the complete implementation that handles message history and streaming:

```ts
export const streamChat = httpAction(async (ctx, request) => {
  const body = (await request.json()) as {
    streamId: string;
    conversationId: string;
    userMessage: string;
  };

  const generateChat = async (
    ctx: any,
    request: any,
    streamId: StreamId,
    chunkAppender: any
  ) => {
    try {
      // Get the message that we're streaming to
      const message = await ctx.runQuery(api.chat.getMessageByStreamId, {
        streamId,
      });
      if (!message) {
        await chunkAppender("Error: Message not found");
        return;
      }

      // Get conversation history
      const allMessages = await ctx.runQuery(api.chat.getMessagesInternal, {
        conversationId: message.conversationId,
      });

      // Get the user's latest message
      const userMessages = allMessages.filter((m: any) => m.role === "user");
      const latestUserMessage = userMessages[userMessages.length - 1];

      if (!latestUserMessage) {
        await chunkAppender("Hello! How can I help you today?");
        return;
      }

      // Generate AI response (this is where you'd integrate with OpenAI, Claude, etc.)
      const userContent = latestUserMessage.content;
      const response = `I received your message: "${userContent}". How can I help you further?`;

      // Stream the response character by character
      for (let i = 0; i < response.length; i++) {
        await chunkAppender(response[i]);
        await new Promise((resolve) => setTimeout(resolve, 50));
      }

      // Mark the message as complete
      await ctx.runMutation(api.chat.markStreamComplete, {
        messageId: message._id,
        finalContent: response,
      });
    } catch (error) {
      console.error("Chat generation error:", error);
      const errorMessage =
        "Sorry, an error occurred while generating the response.";
      await chunkAppender(errorMessage);
    }
  };

  const response = await persistentTextStreaming.stream(
    ctx,
    request,
    body.streamId as StreamId,
    generateChat
  );

  // Set CORS headers
  response.headers.set("Access-Control-Allow-Origin", "*");
  response.headers.set("Vary", "Origin");
  return response;
});
```

Key features:

- Retrieves the streaming message using `streamId`
- Fetches conversation history for context
- Streams response character by character for visual effect
- Marks the message as complete when done
- Handles errors gracefully

## 3. Frontend: Using the `useStream` Hook

Once the backend is set up with the `/chat-stream` endpoint, we can connect it to the client. The `@convex-dev/persistent-text-streaming/react` package provides a `useStream` hook that automatically handles real-time data updates in your UI.

### Basic Usage

Here's a simple example of how to display streaming text in a React component:

```tsx
import { useStream } from "@convex-dev/persistent-text-streaming/react";
import { StreamId } from "@convex-dev/persistent-text-streaming";
import { api } from "../../convex/_generated/api";

interface Message {
  _id: string;
  conversationId: string;
  role: "user" | "assistant";
  content: string;
  streamId?: string;
  isStreaming?: boolean;
}

function StreamingMessage({ message }: { message: Message }) {
  // Convex site URL for HTTP actions - convert .cloud to .site
  const convexApiUrl = import.meta.env.VITE_CONVEX_URL;
  const convexSiteUrl =
    convexApiUrl?.replace(".convex.cloud", ".convex.site") ||
    window.location.origin;

  // For newly created streaming messages, this component should drive the stream
  const isDriven = message.isStreaming === true;

  const { text, status } = useStream(
    api.chat.getStreamBody,
    new URL(`${convexSiteUrl}/chat-stream`),
    isDriven, // Drive the stream if the message is actively streaming
    message.streamId as StreamId
  );

  // Use streamed text if available and streaming, otherwise use message content
  const displayText = status === "streaming" && text ? text : message.content;
  const isActive = status === "streaming" || message.isStreaming;

  return (
    <div className="whitespace-pre-wrap break-words">
      {displayText}
      {isActive && (
        <span className="inline-block w-2 h-5 bg-current opacity-75 animate-pulse ml-1" />
      )}
    </div>
  );
}
```

- **api.chat.getStreamBody** → The Convex query function that retrieves stream content
- **URL** → Points to the Convex backend HTTP route. Note: Convex serves HTTP endpoints on `.convex.site` domain, not `.convex.cloud`
- **isDriven** → Controls streaming behavior based on `message.isStreaming`:
  - `true`: Automatically starts streaming when the message is actively streaming
  - `false`: Stream remains paused when the message is not streaming
- **streamId** → A unique ID created by the backend when generating a streaming response
- **displayText** → Shows streamed text during streaming, falls back to message content otherwise
- **isActive** → Visual indicator shown when either streaming is active or message is marked as streaming

### Manual Start (Driven Option)

You can set `isDriven` to `false` to start streaming manually, such as when a button is clicked:

```tsx
const isDriven = false; // Manual control

const { text, status } = useStream(
  api.chat.getStreamBody,
  new URL(`${convexSiteUrl}/chat-stream`),
  isDriven,
  streamId as StreamId
);

// Since isDriven is false, streaming won't start automatically
// You'll need to trigger it based on your app's logic
```

- **Automatic mode (isDriven: true):** Starts immediately when data is available.
- **Manual mode (isDriven: false):** Stream remains paused until conditions change.

## 4. Linking Stream Creation and Consumption

To take full advantage of `PersistentTextStreaming`, the backend needs to issue a stream ID when creating a message, and the frontend uses that ID to subscribe to real-time responses.

### 1. Assigning a Stream ID in the Backend

When a user sends a message, you can store it and create an empty assistant message with a unique `streamId`:

```ts
const persistentTextStreaming = new PersistentTextStreaming(
  components.persistentTextStreaming
);

export const sendMessage = mutation({
  args: { conversationId: v.id("conversations"), content: v.string() },
  handler: async (ctx, args) => {
    const userId = await getAuthUserId(ctx);
    if (!userId) throw new Error("Not authenticated");

    const conversation = await ctx.db.get(args.conversationId);
    if (!conversation || conversation.userId !== userId) {
      throw new Error("Conversation not found");
    }

    // Insert user message
    const userMessageId = await ctx.db.insert("messages", {
      conversationId: args.conversationId,
      role: "user",
      content: args.content,
    });

    // Create a stream for the AI response
    const streamId = await persistentTextStreaming.createStream(ctx);

    // Create the AI message with streaming enabled
    const aiMessageId = await ctx.db.insert("messages", {
      conversationId: args.conversationId,
      role: "assistant",
      content: "", // Start with empty content
      streamId: streamId,
      isStreaming: true,
    });

    // Update conversation timestamp
    await ctx.db.patch(args.conversationId, {
      lastMessageAt: Date.now(),
    });

    return {
      userMessageId,
      aiMessageId,
      streamingMessageId: aiMessageId,
      streamId: streamId,
    };
  },
});
```

The `streamId` is saved alongside the assistant message, and the frontend can use it with `useStream` to subscribe to live updates.

### 2. Consuming Streams in React

On the frontend, query the messages and display those with a `streamId` using a streaming component:

```tsx
import React from "react";
import { useQuery } from "convex/react";
import { api } from "../../convex/_generated/api";
import StreamingMessage from "./StreamingMessage";

const MessageList: React.FC<{ conversationId: string }> = ({
  conversationId,
}) => {
  const messages = useQuery(api.chat.getMessages, { conversationId });

  return (
    <div className="message-list">
      {messages?.map((message) => (
        <div key={message._id} className={`message ${message.role}`}>
          {message.role === "assistant" && message.streamId ? (
            <StreamingMessage message={message} />
          ) : (
            <div className="whitespace-pre-wrap break-words">
              {message.content}
            </div>
          )}
        </div>
      ))}
    </div>
  );
};
```

Messages with a `streamId` are displayed using the `StreamingMessage` component. Once streaming completes, the final text is saved and rendered as part of the conversation. Regular messages are shown as usual.

With this structure:

- The backend issues a `streamId` and creates a placeholder assistant message.
- A streaming action progressively writes the AI response.
- The frontend subscribes to updates using the `streamId`.

This pattern enables a ChatGPT-style, real-time chat experience built on Convex.

With this pattern in place, you can create a fully interactive, real-time AI chat experience in Convex without handling SSE or WebSocket logic manually.

<img src="https://patient-pigeon-633.convex.cloud/api/storage/6eb78245-ba80-43c4-8e27-63321d3fa8e2" alt="7" style="height: 480px; width: auto; max-width: 100%;">

> The final result: Real-time streaming responses that create an engaging, ChatGPT-like experience for your users.

---

## Conclusion

Convex Components transform complex real-time features into simple, reusable modules. With `@convex-dev/persistent-text-streaming`, we've built a production-ready streaming chat interface without dealing with low-level protocols.

**Key takeaways:**

- Convex Components abstract away infrastructure complexity
- Real-time streaming is as simple as installing a package and using a React hook
- The pattern of backend stream creation + frontend consumption scales to any streaming use case

**What's next?**

- Explore other [Convex Components](https://www.convex.dev/components) for features like auth, payments, and more
- Check out the [Convex documentation](https://docs.convex.dev) for advanced patterns
- Join the [Convex Discord](https://convex.dev/community) to share your streaming implementations
