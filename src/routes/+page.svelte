<script lang="ts">
  import { invoke } from "@tauri-apps/api/core";
  import { Handshake, Palette } from "lucide-svelte";
  import { onMount } from "svelte";

  let name = $state("");
  let greetMsg = $state("");

  async function greet(event: Event) {
    event.preventDefault();
    if (!name.trim()) {
      greetMsg = "Please enter a name.";
      return;
    }
    greetMsg = await invoke("greet", { name });
  }

  const skeletonThemes = [
    "catppuccin","cerberus","concord","crimson","fennec","hamlindigo","legacy","mint","modern",
    "mona","nosh","nouveau","pine","reign","rocket","rose","sahara","seafoam","terminus",
    "vintage","vox","wintry"
  ];

  let currentTheme = $state(document.documentElement.getAttribute("data-theme") ?? "cerberus");
  function switchTheme(theme: string) {
    document.documentElement.setAttribute("data-theme", theme);
    currentTheme = theme;
  }

  onMount(() => switchTheme(currentTheme));
</script>

<!-- App Header -->
<header class="sticky top-0 z-30 glass border-b border-surface-700/60 backdrop-blur-md">
  <div class="container mx-auto flex items-center gap-3 px-4 py-3">
    <img src="/tauri.svg" alt="Tauri" class="h-6 w-6" />
    <span class="text-base font-semibold text-primary-300 tracking-wide">Tauri + SvelteKit</span>
    <span class="ml-2 px-2 py-0.5 rounded bg-surface-600 text-surface-50 text-xs font-mono">Skeleton v4</span>
    <div class="ml-auto flex items-center gap-2 text-surface-300">
      <Palette class="w-5 h-5" />
      <span class="text-xs">Theme</span>
    </div>
  </div>
  <div class="container mx-auto px-4 pb-3">
    <div class="flex gap-2 flex-wrap">
      {#each skeletonThemes as theme}
        <button
          type="button"
          class="btn btn-xs rounded-md border-none px-3 py-1 glass
                 {currentTheme === theme ? 'bg-primary-600 text-primary-50 outline outline-2 outline-primary-400' : 'preset-filled-surface-700 text-surface-200 hover:bg-primary-700 hover:text-primary-50'}"
          aria-pressed={currentTheme === theme}
          onclick={() => switchTheme(theme)}
          title={`Switch to ${theme} theme`}
        >
          {theme}
        </button>
      {/each}
    </div>
  </div>
</header>

<main class="relative min-h-[70vh] w-full bg-gradient-to-br from-surface-900 via-surface-800 to-surface-900">
  <!-- Decorative background -->
  <div class="absolute inset-0 -z-10 opacity-40">
    <div class="absolute -top-20 left-1/4 h-72 w-72 bg-primary-800/30 blur-3xl rounded-full"></div>
    <div class="absolute bottom-0 right-1/4 h-72 w-72 bg-secondary-800/20 blur-3xl rounded-full"></div>
  </div>

  <section class="container mx-auto px-4 pt-10 pb-8 flex flex-col items-center gap-6">
    <!-- Hero copy -->
    <div class="text-center">
      <h1 class="text-3xl font-bold tracking-tight text-primary-200">Tauri + SvelteKit + Skeleton v4</h1>
      <p class="mt-2 text-surface-300">Starter UI using Skeleton v4 and Tailwind CSS.</p>
    </div>

    <!-- Main Card -->
    <div class="glass max-w-md w-full rounded-2xl shadow-xl border border-surface-700 p-6 space-y-4">
      <h2 class="text-lg font-semibold text-primary-200">Say hello</h2>
      <form class="flex flex-col gap-4 w-full" onsubmit={greet}>
        <input
          class="input input-lg w-full rounded-lg focus:outline-none ring-2 ring-surface-500/40 focus:ring-primary-500/70 transition"
          id="greet-input"
          placeholder="Enter a name..."
          bind:value={name}
          autocomplete="off"
        />
        <button type="submit" class="btn btn-lg rounded-lg glass bg-primary-600 text-primary-50 hover:bg-primary-500 transition">
          <Handshake class="w-6 h-6 mr-1 inline-block" />
          <span>Greet</span>
        </button>
      </form>

      {#if greetMsg !== ""}
        <div class="flex items-center gap-2 mt-2 bg-primary-800/70 rounded-lg px-4 py-2 glass border border-primary-400">
          <Handshake class="w-6 h-6 text-primary-300" />
          <span class="text-base text-primary-50 font-semibold tracking-wide">{greetMsg}</span>
        </div>
      {/if}

      <div class="text-xs text-surface-500 font-mono">Styled with Skeleton v4 & TailwindCSS</div>
    </div>
  </section>
</main>