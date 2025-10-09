<script lang="ts">
  import { onMount } from 'svelte';

  // -------------------------
  // State + messages
  // -------------------------
  let habits: string[] = []; // start with no habits
  let habitText = '';
  let message = '';

  const normalize = (str: string) => str.trim().toLowerCase();

  function addHabit() {
    const text = habitText.trim();
    if (!text) {
      message = '⚠️ You must write a habit first.';
      return;
    }
    if (habits.some((h) => normalize(h) === normalize(text))) {
      message = `❌ The habit "${text}" already exists.`;
      return;
    }
    habits = [...habits, text];
    checks = [...checks, false];
    habitText = '';
    message = `✅ Habit "${text}" added.`;
    saveChecksFor(todayKey());
  }

  function removeHabitByName() {
    const name = habitText.trim();
    if (!name) {
      message = '⚠️ Write the habit name you want to remove.';
      return;
    }
    const idx = habits.findIndex((h) => normalize(h) === normalize(name));
    if (idx === -1) {
      message = `❌ The habit "${name}" does not exist.`;
      return;
    }
    habits = habits.filter((_, i) => i !== idx);
    checks = checks.filter((_, i) => i !== idx);
    message = `🗑️ Habit "${name}" removed.`;
    habitText = '';
    saveChecksFor(todayKey());
  }

  // -------------------------
  // Streak logic
  // -------------------------
  let checks: boolean[] = [];
  let streak = 0;

  function toLocalYMD(d = new Date()): string {
    const y = d.getFullYear();
    const m = String(d.getMonth() + 1).padStart(2, '0');
    const day = String(d.getDate()).padStart(2, '0');
    return `${y}-${m}-${day}`;
  }

  function daysDiff(a: string, b: string): number {
    const pa = a.split('-').map(Number);
    const pb = b.split('-').map(Number);
    const da = new Date(pa[0], pa[1] - 1, pa[2]);
    const db = new Date(pb[0], pb[1] - 1, pb[2]);
    const MS = 24 * 60 * 60 * 1000;
    return Math.round((db.getTime() - da.getTime()) / MS);
  }

  const STREAK_KEY = 'streak';
  const LAST_DAY_KEY = 'lastDate';
  const COMPLETED_PREFIX = 'completed-';
  const CHECKS_PREFIX = 'checks-';
  const NOTE_PREFIX = 'note-';

  function todayKey() {
    return toLocalYMD(new Date());
  }
  function completedKey(date: string) {
    return `${COMPLETED_PREFIX}${date}`;
  }
  function checksKey(date: string) {
    return `${CHECKS_PREFIX}${date}`;
  }
  function noteKey(date: string) {
    return `${NOTE_PREFIX}${date}`;
  }

  function saveChecksFor(date: string) {
    localStorage.setItem(checksKey(date), JSON.stringify(checks));
  }
  function loadChecksFor(date: string) {
    const raw = localStorage.getItem(checksKey(date));
    const len = habits.length;
    if (raw) {
      try {
        const arr = JSON.parse(raw);
        if (Array.isArray(arr) && arr.length === len) {
          checks = arr.map(Boolean);
          return;
        }
      } catch {}
    }
    checks = Array(len).fill(false);
  }

  function setCompleted(date: string, value: boolean) {
    localStorage.setItem(completedKey(date), JSON.stringify(!!value));
  }
  function getCompleted(date: string): boolean {
    const raw = localStorage.getItem(completedKey(date));
    if (!raw) return false;
    try {
      return !!JSON.parse(raw);
    } catch {
      return false;
    }
  }

  function saveStreak() {
    localStorage.setItem(STREAK_KEY, String(streak));
  }
  function loadStreak() {
    const raw = localStorage.getItem(STREAK_KEY);
    const v = Number(raw);
    streak = Number.isFinite(v) && v >= 0 ? v : 0;
  }

  function saveLastDate(date: string) {
    localStorage.setItem(LAST_DAY_KEY, date);
  }
  function loadLastDate(): string | null {
    return localStorage.getItem(LAST_DAY_KEY);
  }

  function rollStreakIfNewDay() {
    const today = todayKey();
    const last = loadLastDate();

    if (!last) {
      saveLastDate(today);
      return;
    }
    if (last === today) return;

    const diff = daysDiff(last, today);
    const yesterdayCompleted = getCompleted(last);

    if (diff === 1) {
      streak = yesterdayCompleted ? Math.max(0, streak) + 1 : 0;
      saveStreak();
      checkCelebration();
    } else if (diff > 1) {
      streak = 0;
      saveStreak();
    }

    loadChecksFor(today);
    loadNoteFor(today);
    saveLastDate(today);
    noteLocked = (noteText?.trim().length ?? 0) > 0;
    if (noteLocked) noteMessage = '✅ Daily note submitted';
    else noteMessage = '';
  }

  $: allCompleted =
    habits.length > 0 && checks.length === habits.length && checks.every(Boolean);

  $: if (typeof window !== 'undefined') {
    const today = todayKey();
    setCompleted(today, allCompleted);
    saveChecksFor(today);
  }

  // -------------------------
  // Daily note (one per day)
  // -------------------------
  let noteText: string = '';
  let noteMessage: string = '';
  let noteLocked: boolean = false;

  function saveNoteFor(date: string) {
    if (noteLocked) return;
    const toSave = (noteText ?? '').trim();
    if (!toSave) {
      noteMessage = '⚠️ Write something before saving.';
      return;
    }
    localStorage.setItem(noteKey(date), toSave);
    noteMessage = '✅ Daily note submitted';
    noteLocked = true;
  }
  function loadNoteFor(date: string) {
    const raw = localStorage.getItem(noteKey(date));
    noteText = raw ?? '';
    noteLocked = (noteText?.trim().length ?? 0) > 0;
    noteMessage = noteLocked ? '✅ Daily note submitted' : '';
  }

  onMount(() => {
    checks = Array(habits.length).fill(false);
    loadStreak();
    rollStreakIfNewDay();
    loadChecksFor(todayKey());
    loadNoteFor(todayKey());
  });

  // -------------------------
  // Sidebar (drawer)
  // -------------------------
  let sidebarOpen = false;
  function toggleSidebar() {
    sidebarOpen = !sidebarOpen;
  }
  function closeSidebar() {
    sidebarOpen = false;
  }

  // -------------------------
  // Colors only for the streak badge (not the whole card)
  // -------------------------
  $: streakBadgeClasses =
    streak >= 20
      ? 'bg-red-100/80 border-red-300 text-red-800'
      : streak >= 10
      ? 'bg-orange-100/80 border-orange-300 text-orange-800'
      : streak >= 5
      ? 'bg-yellow-100/80 border-yellow-300 text-yellow-800'
      : 'bg-gray-100 border-gray-200 text-slate-800';

  // -------------------------
  // Celebration GIFs (effort-themed) as a small thumbnail under the sidebar
  // -------------------------
  let showCelebration = false;
  let celebrationGif = '';

  function checkCelebration() {
    if ([5, 10, 20].includes(streak)) {
      celebrationGif =
        streak === 5
          // push / grit
          ? 'https://media.giphy.com/media/xT9IgG50Fb7Mi0prBC/giphy.gif'
          : streak === 10
          // keep running / perseverance
          ? 'https://media.giphy.com/media/26FLdmIp6wJr91JAI/giphy.gif'
          // summit / big effort payoff
          : 'https://media.giphy.com/media/l0Exk8EUzSLsrErEQ/giphy.gif';
      showCelebration = true;
      // hide after 6s
      setTimeout(() => (showCelebration = false), 6000);
    }
  }
</script>

<main class="min-h-screen bg-gradient-to-br from-slate-50 via-white to-slate-100">
  <!-- Menu Button -->
  <button
    class="fixed left-3 top-3 z-40 rounded-xl border border-slate-200 bg-white/90 backdrop-blur px-3 py-2 shadow hover:bg-slate-50"
    on:click={toggleSidebar}
    aria-label="Open/close menu"
    aria-expanded={sidebarOpen}
    aria-controls="app-sidebar"
  >
    {#if !sidebarOpen} ☰ {:else} ✕ {/if}
  </button>

  {#if sidebarOpen}
    <div class="fixed inset-0 z-30 bg-black/30" on:click={closeSidebar} />
  {/if}

  <!-- Sidebar -->
  <aside
    id="app-sidebar"
    class="fixed left-0 top-0 z-40 h-full w-80 -translate-x-full transform shadow-xl transition-transform duration-300"
    class:translate-x-0={sidebarOpen}
  >
    <div class="p-4">
      <div class="rounded-2xl border border-slate-200 bg-white/90 backdrop-blur p-4 shadow">
        <h2 class="text-sm font-semibold text-slate-600 mb-2">Navigation</h2>
        <a
          href="/diario"
          data-sveltekit-preload-data
          class="block w-full text-left px-3 py-2 rounded-xl border border-slate-200 hover:bg-slate-50 text-slate-800"
          on:click={closeSidebar}
        >
          📔 Journal
        </a>


        <a
  href="/agenda"
  data-sveltekit-preload-data
  class="block w-full text-left px-3 py-2 rounded-xl border border-slate-200 hover:bg-slate-50 text-slate-800"
  on:click={closeSidebar}
>
  📅 Schedule
</a>

<a
  href="/recompensas"
  data-sveltekit-preload-data
  class="block w-full text-left px-3 py-2 rounded-xl border border-slate-200 hover:bg-slate-50 text-slate-800"
  on:click={closeSidebar}
>
  🏆 Your Rewards
</a>



        <hr class="my-4 border-slate-200" />

        <!-- STREAK BADGE (only this changes color) -->
        <div>
          <div class="text-xs uppercase tracking-wider text-slate-500">Streak</div>
          <div class={`mt-2 inline-flex items-center gap-2 rounded-xl border px-3 py-2 text-base font-extrabold ${streakBadgeClasses}`}>
            {streak} {streak > 0 ? '🔥' : ''}
          </div>
          <p class="text-[11px] text-slate-500 mt-2">
            Increases when you completed everything yesterday.
          </p>
        </div>

        <hr class="my-4 border-slate-200" />

        <div>
          <div class="text-xs uppercase tracking-wider text-slate-500 mb-2">
            Completed habits
          </div>
          {#if habits.length === 0}
            <p class="text-sm text-slate-500">No habits.</p>
          {:else}
            <ul class="space-y-2">
              {#each habits as habit, i}
                {#if checks[i]}
                  <li class="flex items-center justify-between rounded-lg border border-slate-200 bg-white/70 px-3 py-2">
                    <label class="flex items-center gap-2">
                      <input
                        type="checkbox"
                        class="h-4 w-4 accent-indigo-600"
                        bind:checked={checks[i]}
                      />
                      <span class="line-through text-slate-600">{habit}</span>
                    </label>
                  </li>
                {/if}
              {/each}
            </ul>
          {/if}
        </div>
      </div>

      <!-- Mini celebration GIF BELOW the sidebar card -->
      {#if showCelebration}
        <div class="mt-3 rounded-xl border border-slate-200 bg-white/90 p-2 shadow">
          <div class="text-xs font-semibold text-slate-600 mb-1">Keep going!</div>
          <img src={celebrationGif} alt="Effort celebration" class="h-24 w-auto rounded-lg" />
        </div>
      {/if}
    </div>
  </aside>

  <!-- Main content -->
  <section class="mx-auto max-w-6xl px-4 md:px-6 pt-16 pb-8">
    <section class="text-center mb-10">
      <h1 class="text-5xl md:text-8xl font-extrabold tracking-tight text-red-500 bg-clip-text bg-gradient-to-r from-indigo-700 via-sky-600 to-cyan-500 drop-shadow-sm">
        DAYTRACK
      </h1>
      <p class="mt-8 text-lg md:text-xl text-slate-600">
        Build better habits. Stay consistent. Reach your goals.
      </p>
      <p class="mt-4 text-xl font-semibold text-slate-800">
        🔥 Streak: {streak} days
      </p>
    </section>

    <div class="bg-white/90 backdrop-blur border border-slate-200 shadow-2xl rounded-3xl p-8">
      <!-- Input -->
      <label for="habit-input" class="block text-sm font-medium text-slate-600 mb-2">
        Habit name
      </label>
      <input
        id="habit-input"
        type="text"
        bind:value={habitText}
        placeholder="Write a habit..."
        class="w-full px-4 py-3 rounded-xl border border-slate-300 focus:outline-none focus:ring-2 focus:ring-indigo-400 shadow-sm mb-5"
      />

      <!-- Buttons -->
      <div class="flex flex-wrap gap-3 justify-center mb-6">
        <button
          on:click={addHabit}
          class="px-5 py-3 rounded-xl bg-indigo-600 hover:bg-indigo-700 text-white font-semibold shadow"
        >
          ➕ Add Habit
        </button>
        <button
          on:click={removeHabitByName}
          class="px-5 py-3 rounded-xl bg-rose-600 hover:bg-rose-700 text-white font-semibold shadow"
          title="Write the exact name of the habit and press"
        >
          🗑️ Remove Habit
        </button>
      </div>

      {#if message}
        <p
          class="text-center font-medium mb-6
                 {message.startsWith('✅') ? 'text-emerald-600' : ''}
                 {message.startsWith('❌') ? 'text-rose-600' : ''}
                 {message.startsWith('⚠️') ? 'text-amber-600' : ''}
                 {message.startsWith('🗑️') ? 'text-slate-600' : ''}"
        >
          {message}
        </p>
      {/if}

      <!-- Pending habits -->
      <h2 class="text-lg font-semibold text-slate-800 mb-3">Your habits</h2>
      <ul class="grid gap-3 sm:grid-cols-2">
        {#each habits as habit, i}
          {#if !checks[i]}
            <li class="flex items-center gap-3 bg-slate-50 hover:bg-slate-100 border border-slate-200 rounded-xl px-4 py-3 transition shadow-sm">
              <input type="checkbox" class="w-5 h-5 accent-indigo-600" bind:checked={checks[i]} />
              <span class="text-slate-800">{habit}</span>
            </li>
          {/if}
        {/each}
      </ul>

      <!-- Daily note -->
      <div class="mt-8">
        <h2 class="text-lg font-semibold text-slate-800 mb-2">
          Daily note
        </h2>
        <p class="text-sm text-slate-600 mb-2">
          Write how you felt today.
        </p>
        <textarea
          rows="4"
          bind:value={noteText}
          placeholder="How was your day? What did you learn, what was difficult, what are you proud of?"
          class="w-full px-4 py-3 rounded-xl border border-slate-300 focus:outline-none focus:ring-2 focus:ring-indigo-400 shadow-sm"
          disabled={noteLocked}
        ></textarea>

        <div class="mt-3 flex items-center gap-3">
          <button
            on:click={() => saveNoteFor(todayKey())}
            class="px-4 py-2 rounded-xl text-white font-semibold shadow transition bg-emerald-600 hover:bg-emerald-700 disabled:opacity-50 disabled:cursor-not-allowed"
            disabled={noteLocked}
          >
            💾 Save note
          </button>
          {#if noteMessage}
            <span class="text-sm text-emerald-700">{noteMessage}</span>
          {/if}
        </div>
      </div>
    </div>
  </section>
  

  <!-- DEBUG: Streak test panel -->
<div class="fixed right-3 bottom-3 z-50 rounded-xl border border-slate-200 bg-white/90 backdrop-blur p-3 shadow space-y-2">
  <div class="text-xs font-semibold text-slate-600">Debug Streak</div>

  <!-- Mark YESTERDAY as complete to increase streak -->
  <button
    class="w-full rounded-lg px-3 py-2 border text-sm hover:bg-slate-50"
    on:click={() => {
      const now = new Date();
      const y = new Date(now.getFullYear(), now.getMonth(), now.getDate() - 1);
      const yKey = toLocalYMD(y);

      // force yesterday as complete
      localStorage.setItem(`checks-${yKey}`, JSON.stringify(habits.map(() => true)));
      localStorage.setItem(`completed-${yKey}`, 'true');

      localStorage.setItem('lastDate', yKey);
      location.reload();
    }}
  >
    ✓ Simulate yesterday complete (+1)
  </button>

  <!-- Simulate a missed day -->
  <button
    class="w-full rounded-lg px-3 py-2 border text-sm hover:bg-slate-50"
    on:click={() => {
      const now = new Date();
      const twoDaysAgo = new Date(now.getFullYear(), now.getMonth(), now.getDate() - 2);
      const yesterday = new Date(now.getFullYear(), now.getMonth(), now.getDate() - 1);
      localStorage.setItem('lastDate', toLocalYMD(twoDaysAgo));
      localStorage.setItem(`completed-${toLocalYMD(yesterday)}`, 'false');
      location.reload();
    }}
  >
    ⛔ Simulate missed day (reset)
  </button>

  <!-- Reset streak -->
  <button
    class="w-full rounded-lg px-3 py-2 border text-sm hover:bg-slate-50"
    on:click={() => {
      localStorage.setItem('streak', '0');
      localStorage.removeItem('lastDate');
      checks = checks.map(() => false);
      saveChecksFor(todayKey());
      alert('Streak reset. Start fresh!');
    }}
  >
    ♻️ Reset streak
  </button>
</div>

</main>
