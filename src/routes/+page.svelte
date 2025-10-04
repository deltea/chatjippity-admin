<script lang="ts">
  import { onMount } from "svelte";
  import { io, Socket } from "socket.io-client";
  import { PUBLIC_DEV } from "$env/static/public";

  interface Message {
    bot: boolean;
    text: string;
  }

  let socket: Socket;
  let connecting = $state(true);
  let reply = $state("");
  let roomId = "";
  let partner = $state("");
  let waiting = $state(true);
  let messages: Message[] = $state([]);
  let isReasoning = $state(false);

  onMount(() => {
    socket = io(PUBLIC_DEV === "true" ? "http://localhost:3000" : "wss://chat-jippity-server.onrender.com", { transports: ["websocket"] });

    socket.on("connect", () => {
      console.log("connected to server with id: ", socket.id);

      setTimeout(() => {
        console.log("finding match...");
        socket.emit("find-match", { type: "bot" });
      }, 1000);

      socket.on("match-found", ({ roomId: room, partnerId }) => {
        console.log("room id: ", room);
        console.log("partner id: ", partnerId);
        roomId = room;
        partner = partnerId;

        connecting = false;
      });

      socket.on("receive-message", ({ message, reasoning }) => {
        console.log(reasoning);
        console.log("message received from human: ", message);
        isReasoning = reasoning;
        messages.push({ bot: false, text: message });
        waiting = false;
      });

      socket.on("partner-left", () => {
        console.log("partner left");
        socket.emit("find-match", { type: "bot" });
        connecting = true;
        reset();
      });

      socket.on("disconnect", () => {
        console.log("disconnected from server");
      });

      socket.on("connect_error", (err) => {
        // the reason of the error, for example "xhr poll error"
        console.log(err.message);
      });
    });
  });

  function send() {
    if (reply.length > 0) {
      socket.emit("message-reply", { roomId, reply });
      messages.push({ bot: true, text: reply });
      waiting = true;
      reply = "";
    }
  }

  function reset() {
    roomId = "";
    reply = "";
    messages = [];
  }
</script>

<main class="h-screen glow font-mono flex flex-col justify-center items-center crt">
  <pre class="mb-8">
_____ _           _     ___ _             _ _
/  __ \ |         | |   |_  (_)           (_) |
| /  \/ |__   __ _| |_    | |_ _ __  _ __  _| |_ _   _
| |   | '_ \ / _` | __|   | | | '_ \| '_ \| | __| | | |
| \__/\ | | | (_| | |_/\__/ / | |_) | |_) | | |_| |_| |
\____/_| |_|\__,_|\__\____/|_| .__/| .__/|_|\__|\__, |
                              | |   | |           __/ |
                              |_|   |_|          |___/
  </pre>

  {#if connecting}
    <p class="animate-pulse">CONNECTING TO HUMAN...</p>
  {:else}
    <div class="space-y-8 text-center">
      <h1>SUCCESSFULLY CONNECTED TO HUMAN {partner}</h1>

      {#if messages.length > 0}
        <div class="space-y-2 w-[40rem] h-[30rem] overflow-y-auto">
          {#each messages as message}
            <div class="flex gap-3">
              <span class="font-bold">{message.bot ? "ME" : "HUMAN"}:</span>
              <p>{message.text}</p>
            </div>
          {/each}
        </div>

        {#if waiting}
          <p class="animate-pulse">WAITING FOR PROMPT...</p>
        {:else}
          <form on:submit|preventDefault={send}>
            <input
              type="text"
              bind:value={reply}
              class="outline-none w-[20rem]"
              placeholder="reply here..."
              minlength={isReasoning ? 50 : 0}
            />

            <input type="submit" hidden />
          </form>

          <p class="flex flex-col gap-1 {isReasoning ? "animate-pulse" : ""}">
            REASONING: {isReasoning ? "ON" : "OFF"}

            {#if isReasoning}
              <span class="text-sm">minimum 50 characters</span>
            {/if}
          </p>
        {/if}
      {/if}
    </div>
  {/if}
</main>
