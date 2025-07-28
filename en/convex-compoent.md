# Building Real-Time AI Chat with Convex Components

Today, I’d like to share one of the main reasons you should consider using Convex: **Convex Components**.

If you’re a **developer**, you might already have a sense of what this means. The concept is similar to the component architecture used in frontend frameworks like React, Vue, and Angular, and it also resembles modular units found in backend frameworks like Spring or NestJS.

Convex goes beyond being just another server framework—it **abstracts the entire client-server structure into modular, reusable components**.

In other words, even backend logic is provided as **declarative, reusable components**, which is one of Convex’s most powerful features.

<img src="https://patient-pigeon-633.convex.cloud/api/storage/f17b82bc-8639-461b-b302-d97ad7d17e9c" alt="1" style="height: 480px; width: auto; max-width: 100%;">

> Source: [Convex Components](https://www.convex.dev/components)

From the official site, you can browse through various Convex Components.

For this tutorial, we’ll focus on `@convex-dev/persistent-text-streaming`. Unlike traditional APIs that return the full response at once, this component enables **real-time streaming output**, similar to how ChatGPT streams its responses.

Streaming provides a **faster and more interactive user experience**, as users can start seeing partial responses immediately instead of waiting for the entire message to complete.

Implementing this manually is non-trivial—doing so would usually require protocols like SSE or WebSockets, adding complexity. With Convex’s built-in component, you can **skip the low-level implementation and quickly enable streaming functionality** in your app.

<img src="https://patient-pigeon-633.convex.cloud/api/storage/1c56d398-736b-4fd4-a08d-7eadf4ab5176" alt="2" style="height: 480px; width: auto; max-width: 100%;">

> Source: [Persistent Text Streaming Component](https://www.convex.dev/components/persistent-text-streaming)

To demonstrate this in action, we’ll use [Convex Chef](https://chef.convex.dev/) to quickly spin up an AI chat feature:

<div style="display: flex; width: 100%; gap: 0;">
  <div style="flex: 1;">
    <img src="https://patient-pigeon-633.convex.cloud/api/storage/65103061-ee68-4939-853b-fcce740cf7ca" style="width: 100%; height: auto;">
  </div>
  <div style="flex: 1;">
    <img src="https://patient-pigeon-633.convex.cloud/api/storage/3d08e5a2-9c64-4bbd-abc0-e1b897467a6d" style="width: 100%; height: auto;">
  </div>
</div>

After entering a prompt and waiting briefly, a fully functional React application is automatically generated:

<div style="display: flex; width: 100%; gap: 0;">
  <div style="flex: 1;">
    <img src="https://patient-pigeon-633.convex.cloud/api/storage/1ad7b3c9-dfbd-46b3-8135-aa799d337de3" style="height: 480px; width: auto; max-width: 100%;">
  </div>
  <div style="flex: 1;">
    <img src="https://patient-pigeon-633.convex.cloud/api/storage/2b6c1008-85aa-4a50-ba2d-746e4f0107e5" style="height: 480px; width: auto; max-width: 100%;">
  </div>
</div>

By clicking **“Sign in anonymously”** and entering your ChatGPT API key as instructed, you can start chatting with AI.

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

const app = defineApp();

app.use({
  persistentTextStreaming: {
    package: "@convex-dev/persistent-text-streaming",
  },
});

export default app;
```

As described in the [Convex documentation](https://docs.convex.dev/api/modules/server#defineapp), `defineApp()` is a beta API that lets you define your Convex app by attaching reusable components. This allows you to modularly add functionalities like authentication, payments, or chat without manually setting up `defineSchema` and `defineFunctions`. Since it’s experimental, use it cautiously in production environments.

### Set Up an HTTP Route

```ts
import { components } from "./_generated/api";
import { httpRouter } from "convex/server";

const http = httpRouter();

http.route({
  path: "/chat-stream",
  method: "POST",
  handler: components.persistentTextStreaming.lib.httpHandler,
});

export default http;
```

The `httpHandler` accepts POST requests to `/chat-stream` and enables real-time text streaming. This endpoint will be used to stream AI responses back to the client.

---

## 2. Backend: Implementing Real-Time Streaming

### Basic Streaming Pattern

Here’s a simple example that streams text one character at a time:

```ts
import { PersistentTextStreaming } from "@convex-dev/persistent-text-streaming/server";
import { components } from "./_generated/api";
import { action } from "./_generated/server";
import { v } from "convex/values";

export const streamChat = action({
  args: { streamId: v.string() },
  handler: async (ctx, args) => {
    const pts = new PersistentTextStreaming(
      ctx,
      components.persistentTextStreaming
    );
    const stream = pts.startStream(args.streamId);

    try {
      const message = "Hello! How can I help you today?";
      for (let i = 0; i < message.length; i++) {
        await stream.write(message.substring(0, i + 1));
        await new Promise((r) => setTimeout(r, 50));
      }
      await stream.end();
    } catch (error) {
      await stream.write("An error occurred.");
      await stream.end();
      throw error;
    }
  },
});
```

- `PersistentTextStreaming` initializes the streaming session.
- `startStream()` begins the stream for a given `streamId`.
- `write()` sends text incrementally to the client.
- `end()` closes the stream safely.

---

### Integrating with OpenAI

In many production scenarios, you’ll want to call the OpenAI API and stream the model’s response directly to the client:

```ts
export const streamChatWithOpenAI = action({
  args: { streamId: v.string() },
  handler: async (ctx, args) => {
    const pts = new PersistentTextStreaming(
      ctx,
      components.persistentTextStreaming
    );
    const stream = pts.startStream(args.streamId);

    try {
      const response = await fetch(
        "https://api.openai.com/v1/chat/completions",
        {
          method: "POST",
          headers: {
            Authorization: `Bearer ${process.env.OPENAI_API_KEY}`,
            "Content-Type": "application/json",
          },
          body: JSON.stringify({
            model: "gpt-3.5-turbo",
            messages: [{ role: "user", content: "Hello" }],
            stream: true,
          }),
        }
      );

      if (!response.body) throw new Error("No response body");

      const reader = response.body.getReader();
      const decoder = new TextDecoder();
      let accumulatedText = "";

      while (true) {
        const { done, value } = await reader.read();
        if (done) break;

        const chunk = decoder.decode(value);
        const lines = chunk.split("\n").filter((line) => line.trim());

        for (const line of lines) {
          if (line.startsWith("data: ")) {
            const data = line.slice(6);
            if (data === "[DONE]") break;

            try {
              const parsed = JSON.parse(data);
              const content = parsed.choices?.[0]?.delta?.content;
              if (content) {
                accumulatedText += content;
                await stream.write(accumulatedText);
              }
            } catch (e) {}
          }
        }
      }

      await stream.end();
    } catch (error) {
      await stream.write("An error occurred.");
      await stream.end();
      throw error;
    }
  },
});
```

- The `stream: true` option tells OpenAI to return tokens incrementally.
- `reader.read()` processes data chunks as they arrive.
- `stream.write()` sends partial responses to the client in real time.

---

### Streaming by Word or Sentence

You can also stream responses by word or sentence:

```ts
const words = response.split(" ");
for (let i = 0; i < words.length; i++) {
  await stream.write(words.slice(0, i + 1).join(" "));
  await new Promise((r) => setTimeout(r, 100));
}
```

- `split(" ")` separates the response into words.
- `join(" ")` streams progressively constructed sentences.
- For sentence-based streaming, use `split(/[.!?]/)`.

## 3. Frontend: Using the `useStream` Hook

Once the backend is set up with the `/chat-stream` endpoint, we can connect it to the client. The `@convex-dev/persistent-text-streaming/react` package provides a `useStream` hook that automatically handles real-time data updates in your UI.

### Basic Usage

Here’s a simple example of how to display streaming text in a React component:

```tsx
import React from "react";
import { useStream } from "@convex-dev/persistent-text-streaming/react";

interface StreamingMessageProps {
  streamId: string;
  fallbackContent?: string;
}

const StreamingMessage: React.FC<StreamingMessageProps> = ({
  streamId,
  fallbackContent = "",
}) => {
  const convexSiteUrl = process.env.REACT_APP_CONVEX_URL?.replace(
    ".convex.cloud",
    ".convex.site"
  );

  const { text, error, isLoading } = useStream({
    endpoint: `${convexSiteUrl}/chat-stream`,
    streamId,
    driven: true, // Automatically starts streaming
  });

  if (error) {
    return <span className="error">An error occurred while streaming.</span>;
  }

  return (
    <span>
      {text || fallbackContent}
      {isLoading && <span className="cursor">▊</span>}
    </span>
  );
};

export default StreamingMessage;
```

- **endpoint** → Points to the Convex backend HTTP route (`.convex.cloud` → `.convex.site` adjustment is required).
- **streamId** → A unique ID created by the backend when generating a streaming response.
- **driven: true** → Automatically starts streaming when a message is available.

---

### Manual Start (Driven Option)

You can set `driven` to `false` to start streaming manually, such as when a button is clicked:

```tsx
const { text, startStreaming } = useStream({
  endpoint: `${convexSiteUrl}/chat-stream`,
  streamId,
  driven: false,
});

<button onClick={startStreaming}>Start Streaming</button>;
```

- **Automatic mode:** Starts immediately when data is available.
- **Manual mode:** Requires explicit user action to begin streaming.

---

### Advanced Usage with Retry Logic

In production, you may want to handle network errors gracefully and automatically retry if streaming fails:

```tsx
import React, { useEffect, useState } from "react";
import { useStream } from "@convex-dev/persistent-text-streaming/react";

const AdvancedStreamingMessage: React.FC<{ streamId: string }> = ({
  streamId,
}) => {
  const [isRetrying, setIsRetrying] = useState(false);
  const convexSiteUrl = process.env.REACT_APP_CONVEX_URL?.replace(
    ".convex.cloud",
    ".convex.site"
  );

  const { text, error, isLoading, startStreaming } = useStream({
    endpoint: `${convexSiteUrl}/chat-stream`,
    streamId,
    driven: true,
  });

  // Automatically retry on error
  useEffect(() => {
    if (error && !isRetrying) {
      setIsRetrying(true);
      setTimeout(() => {
        startStreaming();
        setIsRetrying(false);
      }, 2000);
    }
  }, [error, isRetrying, startStreaming]);

  if (error && !isRetrying) {
    return (
      <div className="error-container">
        <span>Streaming failed.</span>
        <button onClick={() => startStreaming()}>Retry</button>
      </div>
    );
  }

  return (
    <div className="streaming-container">
      {text && (
        <div className="streaming-text">
          {text}
          {isLoading && <span className="typing-indicator">●●●</span>}
        </div>
      )}
      {isRetrying && <div className="retry-indicator">Retrying...</div>}
    </div>
  );
};
```

- Automatically retries after a 2-second delay if an error occurs.
- Provides a “Retry” button for manual attempts.
- Displays a typing indicator during streaming and a retry indicator while reconnecting.

---

With this frontend setup, you can seamlessly connect your Convex backend streaming responses to a React UI, creating an experience similar to ChatGPT’s real-time conversation flow.

## 4. Linking Stream Creation and Consumption

To take full advantage of `PersistentTextStreaming`, the backend needs to issue a stream ID when creating a message, and the frontend uses that ID to subscribe to real-time responses.

---

### 1. Assigning a Stream ID in the Backend

When a user sends a message, you can store it and create an empty assistant message with a unique `streamId`:

```ts
export const sendMessage = mutation({
  args: { conversationId: v.id("conversations"), content: v.string() },
  handler: async (ctx, args) => {
    const userId = await ctx.auth.getUserIdentity();
    if (!userId) throw new Error("Not authenticated");

    // Store the user's message
    await ctx.db.insert("messages", {
      conversationId: args.conversationId,
      role: "user",
      content: args.content,
      userId: userId.subject,
      createdAt: Date.now(),
    });

    // Create an assistant message with a stream ID
    const streamId = crypto.randomUUID();
    const assistantMessageId = await ctx.db.insert("messages", {
      conversationId: args.conversationId,
      role: "assistant",
      content: "",
      streamId,
      userId: userId.subject,
      createdAt: Date.now(),
    });

    return { assistantMessageId, streamId };
  },
});
```

The `streamId` is saved alongside the assistant message, and the frontend can use it with `useStream` to subscribe to live updates.

---

### 2. Consuming Streams in React

On the frontend, query the messages and display those with a `streamId` using a streaming component:

```tsx
import React from "react";
import { useQuery } from "convex/react";
import StreamingMessage from "./StreamingMessage";

const MessageList: React.FC<{ conversationId: string }> = ({
  conversationId,
}) => {
  const messages = useQuery(api.chat.getMessages, { conversationId });

  return (
    <div className="message-list">
      {messages?.map((message) => (
        <div key={message._id} className={`message ${message.role}`}>
          {message.streamId ? (
            <StreamingMessage
              streamId={message.streamId}
              fallbackContent={message.content}
            />
          ) : (
            <span>{message.content}</span>
          )}
        </div>
      ))}
    </div>
  );
};
```

Messages with a `streamId` are displayed using the `StreamingMessage` component. Once streaming completes, the final text is saved and rendered as part of the conversation. Regular messages are shown as usual.

---

With this structure:

- The backend issues a `streamId` and creates a placeholder assistant message.
- A streaming action progressively writes the AI response.
- The frontend subscribes to updates using the `streamId`.

This pattern enables a ChatGPT-style, real-time chat experience built on Convex.

---

## 5. Complete Streaming Message Component

Below is a complete React component that subscribes to streamed messages and triggers a callback once the streaming process finishes:

```tsx
import React, { useEffect, useRef } from "react";
import { useStream } from "@convex-dev/persistent-text-streaming/react";

interface CompleteStreamingMessageProps {
  streamId: string;
  onComplete?: (finalText: string) => void;
}

const CompleteStreamingMessage: React.FC<CompleteStreamingMessageProps> = ({
  streamId,
  onComplete,
}) => {
  const prevTextRef = useRef<string>("");

  const convexSiteUrl = process.env.REACT_APP_CONVEX_URL?.replace(
    ".convex.cloud",
    ".convex.site"
  );

  const { text, error, isLoading } = useStream({
    endpoint: `${convexSiteUrl}/chat-stream`,
    streamId,
    driven: true,
  });

  // Detect when streaming is finished
  useEffect(() => {
    if (!isLoading && text && text !== prevTextRef.current) {
      prevTextRef.current = text;
      onComplete?.(text);
    }
  }, [isLoading, text, onComplete]);

  if (error) {
    return <span className="error">An error occurred while streaming.</span>;
  }

  return (
    <span className="streaming-message">
      {text}
      {isLoading && (
        <span className="typing-indicator" aria-label="AI is responding">
          <span>●</span>
          <span>●</span>
          <span>●</span>
        </span>
      )}
    </span>
  );
};

export default CompleteStreamingMessage;
```

With this component in place, you can create a fully interactive, real-time AI chat experience in Convex without handling SSE or WebSocket logic manually.

<img src="https://patient-pigeon-633.convex.cloud/api/storage/6eb78245-ba80-43c4-8e27-63321d3fa8e2" alt="7" style="height: 480px; width: auto; max-width: 100%;">
