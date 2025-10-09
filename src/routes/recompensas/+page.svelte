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
    rewards = [...rewards]; // fuerza reactividad
    notification = `🎉 Congratulations! You unlocked "${r.title}" 🎉`;
    setTimeout(() => (notification = ''), 3500);
  }

  function undoReward(r: Reward) {
    r.claimed = false;
    saveRewards();
    rewards = [...rewards];
  }

  onMount(() => {
    loadStreak();
    loadRewards();
  });

  $: claimedList = rewards.filter(r => !!r.claimed);
  $: visibleList = rewards.filter(r => !r.claimed);
</script>

<!-- Fondo azul MÁS intenso: cambia aquí (p.ej. bg-sky-400/bg-blue-400 para aún más) -->
<main class="min-h-screen bg-sky-300 p-4 md:p-6">
  <!-- Layout a pantalla completa: aside pegado a la izquierda -->
  <div class="grid grid-cols-12 gap-6">
    <!-- ASIDE IZQUIERDO (pegado a la izquierda del viewport) -->
    <aside class="col-span-12 md:col-span-3">
      <div class="rounded-2xl border border-slate-200 bg-white/90 backdrop-blur p-4 shadow space-y-4">
        <div>
          <div class="text-sm font-semibold text-slate-600">Current streak</div>
          <div class="text-2xl font-extrabold text-slate-800 flex items-center gap-2">
            {streak} {streak > 0 ? '🔥' : ''}
          </div>
        </div>

        <div>
          <div class="text-sm font-semibold text-slate-600 mb-2">Claimed</div>
          {#if claimedList.length === 0}
            <p class="text-sm text-slate-500">No rewards claimed yet.</p>
          {:else}
            <ul class="space-y-2">
              {#each claimedList as r}
                <li class="rounded-lg border border-emerald-200 bg-emerald-50 px-3 py-2 flex items-center justify-between">
                  <span class="text-emerald-700 font-medium">{r.title}</span>
                  <button
                    on:click={() => undoReward(r)}
                    class="text-xs px-2 py-1 rounded bg-rose-600 text-white hover:bg-rose-700"
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

    <!-- CONTENIDO DERECHO -->
    <section class="col-span-12 md:col-span-9">
      <!-- Header: título centrado con botón Back rojo a la derecha -->
      <header class="relative mb-10 md:mb-12">
        <h1 class="text-center text-4xl md:text-5xl font-extrabold text-slate-900">
          🏆 Your Rewards
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

      <!-- Add your own reward (debajo del título) -->
      <div class="mb-6 rounded-2xl border border-slate-200 bg-white/90 backdrop-blur p-5 shadow">
        <h2 class="text-lg font-semibold text-slate-800 mb-3">Add your own reward</h2>
        <div class="flex flex-col gap-3 sm:flex-row">
          <input
            type="text"
            bind:value={rewardTitle}
            placeholder="Title (e.g., Go to the movies)"
            class="flex-1 px-4 py-2 rounded-xl border border-slate-300 focus:outline-none focus:ring-2 focus:ring-indigo-400 shadow-sm"
          />
          <input
            type="number"
            bind:value={rewardDays}
            min="1"
            placeholder="Days"
            class="w-28 px-4 py-2 rounded-xl border border-slate-300 focus:outline-none focus:ring-2 focus:ring-indigo-400 shadow-sm"
          />
          <input
            type="text"
            bind:value={rewardDescription}
            placeholder="Description (optional)"
            class="flex-1 px-4 py-2 rounded-xl border border-slate-300 focus:outline-none focus:ring-2 focus:ring-indigo-400 shadow-sm"
          />
          <button
            on:click={addReward}
            class="px-5 py-2 rounded-xl bg-indigo-600 hover:bg-indigo-700 text-white font-semibold shadow"
          >
            ➕ Add reward
          </button>
        </div>
      </div>

      <!-- Grid de rewards no reclamadas -->
      <div class="grid gap-4 sm:grid-cols-2 lg:grid-cols-3">
        {#if visibleList.length === 0}
          <p class="text-slate-700">No rewards yet. Add one above.</p>
        {:else}
          {#each visibleList as r}
            <div class="rounded-2xl border border-slate-200 bg-white/90 backdrop-blur p-4 shadow flex flex-col justify-between">
              <div>
                <h3 class="font-bold text-slate-800 mb-1">{r.title}</h3>
                {#if r.description}
                  <p class="text-sm text-slate-600 mb-2">{r.description}</p>
                {/if}
                <p class="text-xs text-slate-500">Requires {r.days} days</p>
                <p class="text-xs text-slate-500">
                  Progress: {Math.min(streak, r.days)}/{r.days} days
                  ({Math.round((Math.min(streak, r.days)/r.days)*100)}%)
                </p>
              </div>
              <button
                on:click={() => claimReward(r)}
                class="mt-3 px-3 py-2 rounded-xl bg-indigo-600 hover:bg-indigo-700 text-white font-semibold disabled:opacity-50 disabled:cursor-not-allowed"
                disabled={streak < r.days}
              >
                {streak >= r.days ? 'Claim' : 'Locked'}
              </button>
            </div>
          {/each}
        {/if}
      </div>
    </section>
  </div>
</main>
