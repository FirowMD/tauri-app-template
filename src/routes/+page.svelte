<script lang="ts">
  import { invoke } from "@tauri-apps/api/core";
  import { Handshake } from "lucide-svelte";

  let name = $state("");
  let greetMsg = $state("");

  async function greet(event: Event) {
    event.preventDefault();
    greetMsg = await invoke("greet", { name });
  }
  import { onMount } from "svelte";
  let skeletonThemes = [
    "catppuccin","cerberus","concord","crimson","fennec","hamlindigo","legacy","mint","modern",
    "mona","nosh","nouveau","pine","reign","rocket","rose","sahara","seafoam","terminus",
    "vintage","vox","wintry"
  ];
  let currentTheme = $state(document.documentElement.getAttribute("data-theme") ?? "cerberus");

  function switchTheme(theme: string) {
    document.documentElement.setAttribute("data-theme", theme);
    currentTheme = theme;
  }

  onMount(() => {
    switchTheme(currentTheme);
  });
</script>

<!-- App Glass Header -->
<header class="sticky top-0 z-10 glass border-b border-surface-800 shadow-md flex flex-nowrap items-center px-6 py-3 rounded-b-md mb-6 backdrop-blur-md bg-opacity-90">
  <div class="flex items-center gap-2 flex-shrink-0 min-w-0">
    <img src="/tauri.svg" alt="Tauri" class="h-6 w-6" />
    <span class="text-lg font-bold text-primary-300 tracking-wide truncate">Tauri Skeleton</span>
    <span class="ml-2 px-2 py-0.5 rounded bg-surface-600 text-surface-50 text-xs font-mono">v3</span>
  </div>
  <div class="flex-1 flex justify-end overflow-x-auto gap-1 flex-nowrap scrollbar-thin bg-surface-800/80 rounded-md p-1 backdrop-blur-md ml-4">
    {#each skeletonThemes as theme}
      <button
        type="button"
        class="btn btn-sm rounded-md border-none px-2
          {currentTheme === theme ? 'glass bg-primary-600 text-primary-50 outline outline-2 outline-primary-400' : 'preset-filled-surface-700 hover:brightness-110 text-surface-200'}"
        aria-pressed={currentTheme === theme}
        onclick={() => switchTheme(theme)}
        title={`Switch to ${theme} theme`}
      >
        {theme}
      </button>
    {/each}
  </div>
</header>

<main class="flex flex-col items-center justify-center min-h-[70vh] px-2 pb-8 w-full bg-gradient-to-br from-surface-900 via-surface-800 to-surface-900">
  <!-- Card/Panel for main content -->
  <section class="glass max-w-sm w-full rounded-xl shadow-xl border border-surface-700 p-8 flex flex-col items-center gap-6">
    <form class="flex flex-col gap-4 w-full items-center" onsubmit={greet}>
      <div class="w-full flex items-center gap-2">
        <input
          class="input input-lg w-full rounded-lg focus:outline-none ring-2 ring-primary-400/40 focus:ring-primary-500/70 transition"
          id="greet-input"
          placeholder="Enter a name..."
          bind:value={name}
          autocomplete="off"
        />
        <button type="submit" class="btn btn-lg rounded-lg glass preset-filled-surface-600 text-primary-200 hover:bg-primary-600 transition">
          <Handshake class="w-6 h-6 mr-1 inline-block" />
          <span>Greet</span>
        </button>
      </div>
    </form>
    {#if greetMsg !== ""}
      <div class="flex items-center gap-2 mt-2 bg-primary-800/80 rounded-lg px-4 py-2 glass border border-primary-400">
        <Handshake class="w-6 h-6 text-primary-300" />
        <span class="text-lg text-primary-50 font-bold tracking-wide">{greetMsg}</span>
      </div>
    {/if}
    <div class="text-xs text-surface-500 font-mono mt-4">Styled with Skeleton v3 & TailwindCSS</div>
  </section>
</main>