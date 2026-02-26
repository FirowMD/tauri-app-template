<script lang="ts">
  import { CalendarDays, CheckCircle2, Circle, ListTodo, Trash2 } from "lucide-svelte";
  import { LazyStore } from "@tauri-apps/plugin-store";
  import { onMount } from "svelte";
  import { z } from "zod";

  type Filter = "all" | "active" | "completed";

  type Task = {
    id: string;
    title: string;
    notes: string;
    dueDate: string | null;
    completed: boolean;
    createdAt: string;
    updatedAt: string;
  };

  type Settings = {
    filter: Filter;
    theme: string;
  };

  type TaskForm = {
    title: string;
    notes: string;
    dueDate: string;
  };

  const availableThemes = ["cerberus", "mint", "seafoam", "rose", "wintry"] as const;
  const defaultSettings: Settings = {
    filter: "all",
    theme: "cerberus"
  };

  const taskSchema = z.object({
    id: z.string().min(1),
    title: z.string().min(1).max(90),
    notes: z.string().max(400),
    dueDate: z.string().date().nullable(),
    completed: z.boolean(),
    createdAt: z.string().datetime(),
    updatedAt: z.string().datetime()
  });

  const formSchema = z.object({
    title: z.string().trim().min(1, "Title is required.").max(90, "Title can be up to 90 characters."),
    notes: z.string().trim().max(400, "Notes can be up to 400 characters."),
    dueDate: z.string().trim().optional().refine((value) => !value || z.string().date().safeParse(value).success, {
      message: "Please choose a valid date."
    })
  });

  const settingsSchema = z.object({
    filter: z.enum(["all", "active", "completed"]),
    theme: z.string()
  });

  const appStore = new LazyStore("focusboard.store.json", {
    defaults: {
      tasks: [],
      settings: defaultSettings
    }
  });

  let tasks = $state<Task[]>([]);
  let filter = $state<Filter>(defaultSettings.filter);
  let currentTheme = $state(defaultSettings.theme);

  let form = $state<TaskForm>({
    title: "",
    notes: "",
    dueDate: ""
  });

  let formErrors = $state<Partial<Record<keyof TaskForm, string>>>({});
  let globalError = $state("");
  let storeReady = false;

  const stats = $derived.by(() => {
    const completed = tasks.filter((task) => task.completed).length;
    return {
      total: tasks.length,
      completed,
      active: tasks.length - completed
    };
  });

  const visibleTasks = $derived.by(() => {
    return filter === "active"
      ? tasks.filter((task) => !task.completed)
      : filter === "completed"
        ? tasks.filter((task) => task.completed)
        : tasks;
  });

  function createTaskId(): string {
    if (typeof crypto !== "undefined" && typeof crypto.randomUUID === "function") {
      return crypto.randomUUID();
    }
    return `${Date.now()}-${Math.random().toString(16).slice(2)}`;
  }

  function applyTheme(theme: string) {
    document.documentElement.setAttribute("data-theme", theme);
    currentTheme = theme;
  }

  function formatDate(value: string | null): string {
    if (!value) return "No due date";
    return new Date(`${value}T00:00:00`).toLocaleDateString(undefined, {
      year: "numeric",
      month: "short",
      day: "numeric"
    });
  }

  async function persistTasks() {
    if (!storeReady) return;
    await appStore.set("tasks", tasks);
  }

  async function persistSettings() {
    if (!storeReady) return;
    await appStore.set("settings", { filter, theme: currentTheme });
  }

  function clearForm() {
    form = { title: "", notes: "", dueDate: "" };
    formErrors = {};
    globalError = "";
  }

  async function addTask(event: Event) {
    event.preventDefault();
    globalError = "";
    formErrors = {};

    const validation = formSchema.safeParse(form);
    if (!validation.success) {
      const nextErrors: Partial<Record<keyof TaskForm, string>> = {};
      for (const issue of validation.error.issues) {
        const field = issue.path[0] as keyof TaskForm | undefined;
        if (field && !nextErrors[field]) {
          nextErrors[field] = issue.message;
        }
      }
      formErrors = nextErrors;
      return;
    }

    const now = new Date().toISOString();
    const task: Task = {
      id: createTaskId(),
      title: validation.data.title,
      notes: validation.data.notes,
      dueDate: validation.data.dueDate ? validation.data.dueDate : null,
      completed: false,
      createdAt: now,
      updatedAt: now
    };

    tasks = [task, ...tasks];
    clearForm();

    try {
      await persistTasks();
    } catch {
      globalError = "Task was created, but it could not be saved to local storage.";
    }
  }

  async function toggleTask(id: string) {
    tasks = tasks.map((task) =>
      task.id === id
        ? {
            ...task,
            completed: !task.completed,
            updatedAt: new Date().toISOString()
          }
        : task
    );
    try {
      await persistTasks();
    } catch {
      globalError = "Task state changed, but saving failed.";
    }
  }

  async function removeTask(id: string) {
    tasks = tasks.filter((task) => task.id !== id);
    try {
      await persistTasks();
    } catch {
      globalError = "Task was removed, but saving failed.";
    }
  }

  async function clearCompleted() {
    tasks = tasks.filter((task) => !task.completed);
    try {
      await persistTasks();
    } catch {
      globalError = "Completed tasks were cleared, but saving failed.";
    }
  }

  async function setFilter(value: Filter) {
    filter = value;
    try {
      await persistSettings();
    } catch {
      globalError = "Filter changed, but app preferences could not be saved.";
    }
  }

  async function onThemeChange(event: Event) {
    const value = (event.currentTarget as HTMLSelectElement).value;
    applyTheme(value);
    try {
      await persistSettings();
    } catch {
      globalError = "Theme changed, but app preferences could not be saved.";
    }
  }

  async function initApp() {
    try {
      await appStore.init();

      const storedTasks = await appStore.get<unknown>("tasks");
      if (storedTasks !== undefined) {
        const taskParse = z.array(taskSchema).safeParse(storedTasks);
        if (taskParse.success) {
          tasks = taskParse.data;
        }
      }

      const storedSettings = await appStore.get<unknown>("settings");
      if (storedSettings !== undefined) {
        const settingsParse = settingsSchema.safeParse(storedSettings);
        if (settingsParse.success) {
          filter = settingsParse.data.filter;
          currentTheme = settingsParse.data.theme;
        }
      }
    } catch {
      globalError = "App data could not be loaded. Changes will not persist until storage is available.";
    } finally {
      applyTheme(currentTheme);
      storeReady = true;
      await persistSettings();
    }
  }

  onMount(() => {
    void initApp();
  });
</script>

<main class="h-screen overflow-y-auto bg-gradient-to-br from-surface-900 via-surface-800 to-surface-900 px-4 py-8">
  <section class="mx-auto w-full max-w-4xl space-y-6">
    <header class="glass rounded-xl border border-surface-700 p-4">
      <div class="flex flex-wrap items-center gap-3">
        <div class="flex items-center gap-2">
          <ListTodo class="h-6 w-6 text-primary-300" />
          <h1 class="text-xl font-semibold text-primary-200">FocusBoard</h1>
        </div>
        <span class="rounded bg-surface-700 px-2 py-1 text-xs text-surface-200">Offline task manager</span>
        <div class="ml-auto flex items-center gap-2">
          <label class="text-xs text-surface-300" for="theme-select">Theme</label>
          <select
            id="theme-select"
            class="select rounded-md border-surface-600 bg-surface-800 px-2 py-1 text-sm focus:outline-none"
            value={currentTheme}
            onchange={onThemeChange}
          >
            {#each availableThemes as theme}
              <option value={theme}>{theme}</option>
            {/each}
          </select>
        </div>
      </div>

      <div class="mt-4 grid grid-cols-3 gap-2 text-center text-sm">
        <div class="rounded-lg bg-surface-800 p-2">
          <p class="text-surface-400">Total</p>
          <p class="text-lg font-semibold text-surface-100">{stats.total}</p>
        </div>
        <div class="rounded-lg bg-surface-800 p-2">
          <p class="text-surface-400">Active</p>
          <p class="text-lg font-semibold text-amber-300">{stats.active}</p>
        </div>
        <div class="rounded-lg bg-surface-800 p-2">
          <p class="text-surface-400">Completed</p>
          <p class="text-lg font-semibold text-emerald-300">{stats.completed}</p>
        </div>
      </div>
    </header>

    <section class="glass rounded-xl border border-surface-700 p-4">
      <h2 class="mb-3 text-sm font-semibold uppercase tracking-wide text-surface-300">Add task</h2>
      <form class="space-y-3" onsubmit={addTask}>
        <div>
          <input
            class="input w-full rounded-lg border-surface-600 bg-surface-900 px-3 py-2"
            placeholder="Task title"
            bind:value={form.title}
            maxlength="90"
          />
          {#if formErrors.title}
            <p class="mt-1 text-xs text-red-300">{formErrors.title}</p>
          {/if}
        </div>

        <div>
          <textarea
            class="textarea h-24 w-full rounded-lg border-surface-600 bg-surface-900 px-3 py-2"
            placeholder="Notes (optional)"
            bind:value={form.notes}
            maxlength="400"
          ></textarea>
          {#if formErrors.notes}
            <p class="mt-1 text-xs text-red-300">{formErrors.notes}</p>
          {/if}
        </div>

        <div>
          <label class="mb-1 block text-xs text-surface-300" for="due-date">Due date (optional)</label>
          <input
            id="due-date"
            type="date"
            class="input w-full rounded-lg border-surface-600 bg-surface-900 px-3 py-2"
            bind:value={form.dueDate}
          />
          {#if formErrors.dueDate}
            <p class="mt-1 text-xs text-red-300">{formErrors.dueDate}</p>
          {/if}
        </div>

        <div class="flex items-center gap-2">
          <button type="submit" class="btn bg-primary-600 text-primary-50 hover:bg-primary-500">Add Task</button>
          <button type="button" class="btn bg-surface-700 text-surface-100 hover:bg-surface-600" onclick={clearForm}>
            Reset
          </button>
        </div>
      </form>
      {#if globalError}
        <p class="mt-3 rounded-md border border-red-400/50 bg-red-950/50 px-3 py-2 text-sm text-red-200">{globalError}</p>
      {/if}
    </section>

    <section class="glass rounded-xl border border-surface-700 p-4">
      <div class="mb-4 flex flex-wrap items-center gap-2">
        <h2 class="text-sm font-semibold uppercase tracking-wide text-surface-300">Tasks</h2>
        <div class="ml-auto flex gap-2">
          <button
            type="button"
            class={`btn btn-sm ${filter === "all" ? "bg-primary-600 text-primary-50" : "bg-surface-700 text-surface-200"}`}
            onclick={() => void setFilter("all")}
          >
            All
          </button>
          <button
            type="button"
            class={`btn btn-sm ${filter === "active" ? "bg-primary-600 text-primary-50" : "bg-surface-700 text-surface-200"}`}
            onclick={() => void setFilter("active")}
          >
            Active
          </button>
          <button
            type="button"
            class={`btn btn-sm ${filter === "completed" ? "bg-primary-600 text-primary-50" : "bg-surface-700 text-surface-200"}`}
            onclick={() => void setFilter("completed")}
          >
            Completed
          </button>
        </div>
      </div>

      {#if visibleTasks.length === 0}
        <div class="rounded-lg border border-dashed border-surface-600 p-6 text-center text-surface-400">
          No tasks in this view yet.
        </div>
      {:else}
        <ul class="max-h-[45vh] space-y-2 overflow-y-auto pr-1">
          {#each visibleTasks as task (task.id)}
            <li class="flex items-start gap-3 rounded-lg border border-surface-700 bg-surface-900/70 p-3">
              <button
                type="button"
                class="mt-0.5 text-surface-300 hover:text-primary-300"
                onclick={() => void toggleTask(task.id)}
                aria-label={task.completed ? "Mark as active" : "Mark as completed"}
              >
                {#if task.completed}
                  <CheckCircle2 class="h-5 w-5 text-emerald-300" />
                {:else}
                  <Circle class="h-5 w-5" />
                {/if}
              </button>

              <div class="min-w-0 flex-1">
                <p class={`font-medium ${task.completed ? "text-surface-400 line-through" : "text-surface-100"}`}>
                  {task.title}
                </p>
                {#if task.notes}
                  <p class={`mt-1 text-sm ${task.completed ? "text-surface-500 line-through" : "text-surface-300"}`}>
                    {task.notes}
                  </p>
                {/if}
                <p class="mt-2 flex items-center gap-1 text-xs text-surface-400">
                  <CalendarDays class="h-3.5 w-3.5" />
                  <span>{formatDate(task.dueDate)}</span>
                </p>
              </div>

              <button
                type="button"
                class="text-surface-400 hover:text-red-300"
                onclick={() => void removeTask(task.id)}
                aria-label="Delete task"
              >
                <Trash2 class="h-4 w-4" />
              </button>
            </li>
          {/each}
        </ul>
      {/if}

      {#if stats.completed > 0}
        <div class="mt-4 flex justify-end">
          <button type="button" class="btn btn-sm bg-surface-700 text-surface-100 hover:bg-surface-600" onclick={() => void clearCompleted()}>
            Clear completed
          </button>
        </div>
      {/if}
    </section>
  </section>
</main>
