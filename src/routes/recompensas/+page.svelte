<script lang="ts">
  import { onMount } from 'svelte';

  type Reward = {
    id: string;
    title: string;
    days: number;
    description?: string;
    builtin?: boolean;
    claimed?: boolean;
  };

  let rewards: Reward[] = [];
  let rewardTitle = '';
  let rewardDays: number | null = null;
  let rewardDescription = '';
  let streak = 0;
  let notification = '';

  // Drawer (panel lateral)
  let sidebarOpen = false;
  const toggleSidebar = () => (sidebarOpen = !sidebarOpen);
  const closeSidebar = () => (sidebarOpen = false);

  const LIST_KEY = 'rewards-list';

  function saveRewards() {
    localStorage.setItem(LIST_KEY, JSON.stringify(rewards));
    for (const r of rewards) {
      localStorage.setItem(`reward-${r.id}-claimed`, r.claimed ? 'true' : 'false');
    }
  }
  function sortRewards() {
    rewards.sort((a, b) => a.days - b.days || a.title.localeCompare(b.title));
  }
  function loadRewards() {
    const raw = localStorage.getItem(LIST_KEY);
    if (!raw) {
      rewards = [];
      saveRewards();
    } else {
      try {
        const arr = JSON.parse(raw);
        rewards = Array.isArray(arr) ? arr : [];
      } catch {
        rewards = [];
      }
    }
    for (const r of rewards) {
      r.claimed = localStorage.getItem(`reward-${r.id}-claimed`) === 'true';
    }
    sortRewards();
  }
  function loadStreak() {
    const v = Number(localStorage.getItem('streak'));
    streak = Number.isFinite(v) && v >= 0 ? v : 0;
  }

  function addReward() {
    const t = rewardTitle.trim();
    const d = rewardDays ?? 0;
    const desc = rewardDescription.trim();
    if (!t || d <= 0) return;

    const id = `r-${Date.now()}-${Math.random().toString(36).slice(2, 7)}`;
    rewards = [...rewards, { id, title: t, days: d, description: desc, builtin: false, claimed: false }];
    sortRewards();
    saveRewards();
    rewardTitle = '';
    rewardDays = null;
    rewardDescription = '';
  }

  function claimReward(r: Reward) {
    if (r.claimed) return;
    r.claimed = true;
    saveRewards();
    rewards = [...rewards]; // reactividad
    notification = `🎉 Congratulations! You unlocked "${r.title}" 🎉`;
    setTimeout(() => (notification = ''), 3500);
  }

  function undoReward(r: Reward) {
    r.claimed = false;
    saveRewards();
    rewards = [...rewards];
  }

  function deleteReward(r: Reward) {
    rewards = rewards.filter(x => x.id !== r.id);
    localStorage.removeItem(`reward-${r.id}-claimed`);
    saveRewards();
  }

  onMount(() => {
    loadStreak();
    loadRewards();
  });

  $: claimedList = rewards.filter(r => !!r.claimed);
  $: visibleList = rewards.filter(r => !r.claimed);
</script>

<!-- Fondo azul marino (se mantiene) -->
<main class="min-h-screen bg-gray-700 p-4 md:p-6 relative">

  <!-- Botón hamburguesa -->
  <button
    class="fixed left-4 top-4 z-50 rounded-xl border border-white/20 bg-white/20 text-white px-3 py-2 backdrop-blur hover:bg-white/30"
    aria-label="Open/close panel"
    aria-expanded={sidebarOpen}
    on:click={toggleSidebar}
  >
    ☰
  </button>

  <!-- Drawer lateral -->
  {#if sidebarOpen}
    <div class="fixed inset-0 z-40 bg-black/40" on:click={closeSidebar}></div>
  {/if}
  <aside
    class="fixed left-0 top-0 z-50 h-full w-80 -translate-x-full transform transition-transform duration-300"
    class:translate-x-0={sidebarOpen}
    aria-hidden={!sidebarOpen}
  >
    <!-- Panel en amarillo -->
    <div class="h-full overflow-y-auto p-4 space-y-4 rounded-r-2xl border-r border-blue-400 bg-blue-200 shadow-xl">
      <div>
        <div class="text-sm font-semibold text-blue-800">Current streak</div>
        <div class="text-2xl font-extrabold text-blue-900 flex items-center gap-2">
          {streak} {streak > 0 ? '🔥' : ''}
        </div>
      </div>

      <div>
        <div class="text-sm font-semibold text-blue-900 mb-2">Claimed</div>
        {#if claimedList.length === 0}
          <p class="text-sm text-blue-800/90">No rewards claimed yet.</p>
        {:else}
          <ul class="space-y-2">
            {#each claimedList as r}
              <!-- Claimed en verde MÁS oscuro -->
              <li class="rounded-lg border border-emerald-600 bg-emerald-300 px-3 py-2 flex items-center justify-between">
                <span class="text-emerald-900 font-semibold">{r.title}</span>
                <button
                  on:click={() => undoReward(r)}
                  class="text-xs px-2 py-1 rounded bg-red-600 text-white hover:bg-rose-700"
                >
                  Undo
                </button>
              </li>
            {/each}
          </ul>
        {/if}
      </div>
    </div>
  </aside>

  <!-- Cabecera -->
  <header class="relative mb-10 md:mb-12">
    <h1 class="text-center text-white text-3xl font-extrabold tracking-tight">
      🏆 YOUR REWARDS 🏆
    </h1>
    <a
      href="/"
      class="absolute right-0 top-1/2 -translate-y-1/2 inline-block rounded-lg bg-rose-600 px-3 py-1.5 text-white text-sm font-semibold shadow hover:bg-rose-700"
      aria-label="Back to home"
    >
      ← Back
    </a>
  </header>

  <!-- Notificación -->
  {#if notification}
    <div class="mb-4 rounded-xl border border-emerald-300 bg-emerald-50 text-emerald-800 px-4 py-3 shadow">
      {notification}
    </div>
  {/if}

  <!-- Card Add (azul más claro) -->
  <div class="mb-6 rounded-2xl gap-3 bg-sky-50 hover:bg-sky-100
                       border border-sky-200 rounded-xl px-4 py-3 transition shadow-sm">
    <h2 class="text-lg text-gray-700 font-semibold mb-3">Add your own reward</h2>
    <div class="flex flex-col gap-3 sm:flex-row">
      <!-- Inputs en azul claro -->
      <input
        type="text"
        bind:value={rewardTitle}
        placeholder="Title (e.g., Go to the movies)"
        class="flex-1 px-4 py-2 rounded-xl border border-sky-300 focus:outline-none focus:ring-2 focus:ring-sky-400 shadow-sm bg-sky-100 text-blue-900"
      />
      <input
        type="number"
        bind:value={rewardDays}
        min="1"
        placeholder="Days"
        class="w-28 px-4 py-2 rounded-xl border border-sky-300 focus:outline-none focus:ring-2 focus:ring-sky-400 shadow-sm bg-sky-100 text-blue-900"
      />
      <input
        type="text"
        bind:value={rewardDescription}
        placeholder="Description (optional)"
        class="flex-1 px-4 py-2 rounded-xl border border-sky-300 focus:outline-none focus:ring-2 focus:ring-sky-400 shadow-sm bg-sky-100 text-blue-900"
      />
      <!-- Botón Add en tono más claro -->
      <button
        on:click={addReward}
        class="px-5 py-3 rounded-xl bg-green-600 hover:bg-green-700 text-white font-semibold shadow"
      >
        ➕ Add reward
      </button>
    </div>
  </div>

  <!-- Recompensas no reclamadas: azul claro + textos en azul más oscuro -->
  <div class="grid gap-4 sm:grid-cols-2 lg:grid-cols-3">
    {#if visibleList.length === 0}
      <p class="text-blue-200">No rewards yet. Add one above.</p>
    {:else}
      {#each visibleList as r}
        <div class="rounded-2xl border border-sky-300 bg-sky-100 p-4 shadow flex flex-col justify-between">
          <div>
            <h3 class="font-bold text-blue-900 mb-1">{r.title}</h3>
            {#if r.description}
              <p class="text-sm text-blue-800 mb-2">{r.description}</p>
            {/if}
            <p class="text-xs text-blue-700">Requires {r.days} days</p>
            <p class="text-xs text-blue-700">
              Progress: {Math.min(streak, r.days)}/{r.days} days
              ({Math.round((Math.min(streak, r.days)/r.days)*100)}%)
            </p>
          </div>

          <!-- Claim verde -->
          <button
            on:click={() => claimReward(r)}
            class="mt-3 px-3 py-2 rounded-xl bg-emerald-600 hover:bg-emerald-700 text-white font-semibold disabled:opacity-50 disabled:cursor-not-allowed"
            disabled={streak < r.days}
          >
            {streak >= r.days ? 'Claim' : 'Locked'}
          </button>

          <!-- Delete rojo debajo -->
          <button
            on:click={() => deleteReward(r)}
            class="mt-2 px-3 py-2 rounded-xl bg-rose-600 hover:bg-rose-700 text-white font-semibold"
            title="Delete this reward"
          >
            Delete
          </button>
        </div>
      {/each}
    {/if}
  </div>
</main>
