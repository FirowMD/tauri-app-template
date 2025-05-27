<script lang="ts">
  import { invoke } from "@tauri-apps/api/core";
  import { Handshake } from "lucide-svelte";

  let name = $state("");
  let greetMsg = $state("");

  async function greet(event: Event) {
    event.preventDefault();
    greetMsg = await invoke("greet", { name });
  }
</script>

<main class="flex flex-col p-2 w-full h-full bg-surface-900">
  <form class="flex flex-col gap-2" onsubmit={greet}>
    <input
      class="input focus:outline-none"
      id="greet-input"
      placeholder="Enter a name..."
      bind:value={name}
    />
    <button type="submit" class="btn preset-filled-surface-500">Greet</button>
  </form>
  {#if greetMsg !== ""}
    <div class="flex items-center gap-2 mt-4">
      <Handshake class="w-6 h-6 text-primary-500" />
      <span class="text-primary-500">{greetMsg}</span>
    </div>
  {/if}
</main>