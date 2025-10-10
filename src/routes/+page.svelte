<script lang="ts">
  import { onMount } from 'svelte';

  // -------------------------
  // ESTADO INICIAL (variables reactivas)
  // -------------------------

  let habits: string[] = []; 
  let habitText = '';
  let message = '';

  // -------------------------
  // COMPARAR STRINGS (sin acentos, mayúsculas, espacios)
  // -------------------------

  const normalize = (str: string) => str.trim().toLowerCase();

    // -------------------------
    // FUNCIONALIDADES (añadir habito, eliminar hábito)
    // -------------------------

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
    saveChecksFor(todayKey()); // persiste el estado dia actual checks en localStorage
  }

  function removeHabitByName() {
    const name = habitText.trim();
    if (!name) {
      message = '⚠️ Write the habit name you want to remove.';
      return;
    }

    /*busca dentro del array habits el indice idx que coincida con el name;
    si el indice no se encuentra (-1) salta mensaje,
    si coincide crea un array con todos los elementos menos el idx(lo mismo con el array checks, importante porque esta asociado)*/
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
  // Racha (streak) y seguimiento diario (checks)
  // -------------------------
  let checks: boolean[] = [];
  let streak = 0;

  // Convierte una fecha a formato YYYY-MM-DD local
  function toLocalYMD(d = new Date()): string {
    const y = d.getFullYear();
    const m = String(d.getMonth() + 1).padStart(2, '0');
    const day = String(d.getDate()).padStart(2, '0');
    return `${y}-${m}-${day}`;
  }

  // Diferencia en días entre dos fechas YYYY-MM-DD
  function daysDiff(a: string, b: string): number {
    const pa = a.split('-').map(Number);
    const pb = b.split('-').map(Number);
    const da = new Date(pa[0], pa[1] - 1, pa[2]);
    const db = new Date(pb[0], pb[1] - 1, pb[2]);
    const MS = 24 * 60 * 60 * 1000;
    return Math.round((db.getTime() - da.getTime()) / MS);
  }

  const STREAK_KEY = 'streak';
  const LAST_DAY_KEY = 'lastDate'; // última fecha que se actualizó la racha
  const COMPLETED_PREFIX = 'completed-'; // indica si se completó el día
  const CHECKS_PREFIX = 'checks-'; // array de booleanos de tareas por día
  const NOTE_PREFIX = 'note-'; 

    // -------------------------
    // FUNCIONES generan nombres 
    // para el local storage msimo formato
    // -------------------------
  
  function todayKey() { return toLocalYMD(new Date()); }
  function completedKey(date: string) { return `${COMPLETED_PREFIX}${date}`; }
  function checksKey(date: string) { return `${CHECKS_PREFIX}${date}`; }
  function noteKey(date: string) { return `${NOTE_PREFIX}${date}`; }

  function saveChecksFor(date: string) {
    localStorage.setItem(checksKey(date), JSON.stringify(checks));
  }

  // Garantiza que checks siempre esté alineado con los hábitos.
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

  // Guarda en localStorage si un día específico se completó (true o false).
  function setCompleted(date: string, value: boolean) {
    localStorage.setItem(completedKey(date), JSON.stringify(!!value));
  }
  // Recupera si un día específico se completó y lo pasa de bool a str (true o false).
  function getCompleted(date: string): boolean {
    const raw = localStorage.getItem(completedKey(date));
    if (!raw) return false;
    try { return !!JSON.parse(raw); } catch { return false; }
  }

  function saveStreak() { localStorage.setItem(STREAK_KEY, String(streak)); }
  function loadStreak() {
    const raw = localStorage.getItem(STREAK_KEY);
    const v = Number(raw);
    streak = Number.isFinite(v) && v >= 0 ? v : 0;
  }

  function saveLastDate(date: string) { localStorage.setItem(LAST_DAY_KEY, date); }
  function loadLastDate(): string | null { return localStorage.getItem(LAST_DAY_KEY); }

  /* Se ejecuta al arrancar: compara fechas, actualiza la racha 
  según si ayer completaste y prepara el estado de hoy.*/

  function rollStreakIfNewDay() {
    const today = todayKey();
    const last = loadLastDate();

    if (!last) { saveLastDate(today); return; }
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
  }

  $: allCompleted =
    habits.length > 0 && checks.length === habits.length && checks.every(Boolean);

  $: if (typeof window !== 'undefined') {
    const today = todayKey();
    setCompleted(today, allCompleted);
    saveChecksFor(today);
  }

  // -------------------------
  // Daily notes 
  // -------------------------
  let noteText: string = '';
  let noteMessage: string = '';

  function saveNoteFor(date: string) {
    const toSave = (noteText ?? '').trim();
    if (!toSave) {
      noteMessage = '⚠️ Write something before saving.';
      return;
    }
    const prev = localStorage.getItem(noteKey(date));
    const next = prev ? prev + '\n\n' + toSave : toSave;
    localStorage.setItem(noteKey(date), next);
    noteMessage = '✅ Note saved';
    setTimeout(() => (noteMessage = ''), 1600);
    noteText = '';
  }
  function loadNoteFor(date: string) {
    const raw = localStorage.getItem(noteKey(date));
    // no lock; just preload nothing
    noteText = '';
    noteMessage = raw ? 'ℹ️ You already have notes today' : '';
  
  }


  // Sidebar 

  let sidebarOpen = false;
  function toggleSidebar() { sidebarOpen = !sidebarOpen; }
  function closeSidebar() { sidebarOpen = false; }

  // -------------------------
  // Celebration GIFs
  // -------------------------
  let showCelebration = false;
  let celebrationGif = '';
  function checkCelebration() {
    if ([5, 10, 20].includes(streak)) {
      celebrationGif =
        streak === 5
          ? 'https://media.giphy.com/media/xT9IgG50Fb7Mi0prBC/giphy.gif'
          : streak === 10
          ? 'https://media.giphy.com/media/26FLdmIp6wJr91JAI/giphy.gif'
          : 'https://media.giphy.com/media/l0Exk8EUzSLsrErEQ/giphy.gif';
      showCelebration = true;
      setTimeout(() => (showCelebration = false), 6000);
    }
  }

 
  // Funciones reactivas para colores segun racha 
  
  $: streakBadgeClasses =
    streak >= 20
      ? 'bg-red-100/80 border-red-300 text-red-900'
      : streak >= 10
      ? 'bg-orange-100/80 border-orange-300 text-orange-900'
      : streak >= 5
      ? 'bg-yellow-100/80 border-yellow-300 text-yellow-900'
      : 'bg-gray-100 border-gray-300 text-slate-900';

  // Al arrancar, carga hábitos y checks, y gestiona racha; inicializa la app

  onMount(() => {
    checks = Array(habits.length).fill(false);
    loadStreak();
    rollStreakIfNewDay();
    loadChecksFor(todayKey());
    loadNoteFor(todayKey());
  });
</script>

<!-- ========= DISTRIBUCIÓN + COLORES (solo clases y layout) ========= -->

<!-- Fondo azul marino, textos por defecto en azul oscuro para contraste en cards -->
<main class="min-h-screen bg-gray-800 text-blue-900 relative p-4 md:p-6">

  <!-- Botón hamburguesa (amarillo, como Rewards) -->
  <button
    class="fixed left-4 top-4 z-50 rounded-xl border border-yellow-400 bg-yellow-200 px-3 py-2 shadow hover:bg-yellow-400"
    on:click={toggleSidebar}
    aria-label="Open/close panel"
    aria-expanded={sidebarOpen}
    aria-controls="app-sidebar"
  >
    {#if !sidebarOpen} ☰ {:else} ✕ {/if}
  </button>

  <!-- Overlay del drawer -->
  {#if sidebarOpen}
    <div class="fixed inset-0 z-40 bg-black/40" on:click={closeSidebar} />
  {/if}

  <!-- Drawer lateral (amarillo) -->
  <aside
    id="app-sidebar"
    class="fixed left-0 top-0 z-50 h-full w-80 -translate-x-full transform transition-transform duration-300"
    class:translate-x-0={sidebarOpen}
  >
    <div class="h-full overflow-y-auto p-4 space-y-4 rounded-r-2xl border-r border-yellow-900 bg-yellow-200 shadow-xl">
      <div class="text-sm font-semibold text-blue-900">NAVIGATION</div>
      <nav class="space-y-2">
        <a
          href="/diario"
          data-sveltekit-preload-data
          class="block w-full text-left px-3 py-2 rounded-xl border border-yellow-300 bg-yellow-100 hover:bg-yellow-300 text-blue-900"
          on:click={closeSidebar}
        >
          📔 Journal
        </a>
        <a
          href="/agenda"
          data-sveltekit-preload-data
          class="block w-full text-left px-3 py-2 rounded-xl border border-yellow-300 bg-yellow-100 hover:bg-yellow-300 text-blue-900"
          on:click={closeSidebar}
        >
          📅 Schedule
        </a>
        <a
          href="/recompensas"
          data-sveltekit-preload-data
          class="block w-full text-left px-3 py-2 rounded-xl border border-yellow-300 bg-yellow-100 hover:bg-yellow-300 text-blue-900"
          on:click={closeSidebar}
        >
          🏆 Your Rewards
        </a>
      </nav>

      <hr class="my-4 border-yellow-300" />

      <!-- STREAK (solo badge cambia de color según racha) -->
      <div>
        <div class="text-xs uppercase tracking-wider text-blue-800">Streak</div>
        <div class={`mt-2 inline-flex items-center gap-2 rounded-xl border px-3 py-2 text-base font-extrabold ${streakBadgeClasses}`}>
          {streak} {streak > 0 ? '🔥' : ''}
        </div>
        <p class="text-[11px] text-blue-800 mt-2">Increases when you completed everything yesterday.</p>
      </div>

      <hr class="my-4 border-yellow-300" />

      <!-- COMPLETED HABITS (en verde claro como claimed en Rewards) -->
      <div>
        <div class="text-xs uppercase tracking-wider text-blue-800 mb-2">Completed habits</div>
        {#if habits.length === 0}
          <p class="text-sm text-blue-800/90">No habits.</p>
        {:else}
          <ul class="space-y-2">
            {#each habits as habit, i}
              {#if checks[i]}
                <li class="flex items-center justify-between rounded-lg border border-emerald-600 bg-emerald-300 px-3 py-2">
                  <label class="flex items-center gap-2">
                    <input type="checkbox" class="h-4 w-4 accent-emerald-700" bind:checked={checks[i]} />
                    <span class="line-through text-emerald-900 font-medium">{habit}</span>
                  </label>
                </li>
              {/if}
            {/each}
          </ul>
        {/if}
      </div>
    </div>
  </aside>

  <!-- Encabezado: título grande, centrado y BLANCO (como pediste) -->
  <header class="relative mb-10 md:mb-12">
    <h1 class="text-center text-white text-5xl md:text-6xl font-extrabold tracking-tight">
      DAYTRACK
    </h1>
    <!-- (sin botón Back aquí en home) -->
  </header>

  <!-- Contenido principal: cards en AZUL CLARO, inputs en AZUL CLARO -->
  <section class="mx-auto max-w-6xl space-y-6">

    <!-- Mensaje bajo el título -->
    <p class="text-center text-blue-400 text-lg md:text-xl">
      Build better habits. Stay consistent. Reach your goals.
    </p>

    <!-- Card principal (azul claro) -->
    <div class="rounded-2xl border border-sky-400 bg-sky-200 p-6 md:p-8 shadow">
      <!-- Input habit (azul claro) -->
      <label for="habit-input" class="block text-sm font-semibold text-blue-900 mb-2">
        Habit name
      </label>
      <input
        id="habit-input"
        type="text"
        bind:value={habitText}
        placeholder="Write a habit..."
        class="w-full px-4 py-3 rounded-xl border border-sky-400 bg-sky-100
               focus:outline-none focus:ring-2 focus:ring-sky-500 shadow-sm mb-5 text-blue-900"
      />

      <!-- Botones (mantenemos estilos/acciones) -->
      <div class="flex flex-wrap gap-3 justify-center mb-6">
        <button
          on:click={addHabit}
          class="px-5 py-3 rounded-xl bg-green-600 hover:bg-green-700 text-white font-semibold shadow"
        >
          ➕ Add Habit
        </button>
        <button
          on:click={removeHabitByName}
          class="px-5 py-3 rounded-xl border border-red-700 bg-red-600 px-4 py-2 text-white hover:bg-red-700 shadow-sm"
          title="Write the exact name of the habit and press"
        >
          🗑️ Remove Habit
        </button>
      </div>

      {#if message}
        <p
          class="text-center font-medium mb-6"
          class:text-emerald-700={message.startsWith('✅')}
          class:text-rose-700={message.startsWith('❌')}
          class:text-amber-600={message.startsWith('⚠️')}
          class:text-slate-700={message.startsWith('🗑️')}
        >
          {message}
        </p>
      {/if}

      <!-- Lista de hábitos pendientes (card items azul muy claro + textos azul oscuro) -->
      <h2 class="text-lg font-semibold text-blue-900 mb-3">Your habits</h2>
      <ul class="grid gap-3 sm:grid-cols-2">
        {#each habits as habit, i}
          {#if !checks[i]}
            <li class="flex items-center gap-3 bg-sky-50 hover:bg-sky-100
                       border border-sky-200 rounded-xl px-4 py-3 transition shadow-sm">
              <input type="checkbox" class="w-5 h-5 accent-emerald-600" bind:checked={checks[i]} />
              <span class="text-blue-900">{habit}</span>
            </li>
          {/if}
        {/each}
      </ul>

      <!-- Nota del día (inputs azul claro, misma lógica) -->
      <div class="mt-8">
        <h2 class="text-lg font-semibold text-blue-900 mb-2">Daily note ({toLocalYMD(new Date())})</h2>
        <p class="text-sm text-blue-800 mb-2">
          Write how you felt today. Stored per date for your Journal.
        </p>
        <textarea
          rows="4"
          bind:value={noteText}
          placeholder="How was your day? What did you learn, what was difficult, what are you proud of?"
          class="w-full px-4 py-3 rounded-xl border border-sky-400 bg-sky-100
                 focus:outline-none focus:ring-2 focus:ring-sky-500 shadow-sm text-blue-900"
        ></textarea>

        <div class="mt-3 flex items-center gap-3">
          <button
            on:click={() => saveNoteFor(todayKey())}
            class="px-4 py-2 rounded-xl bg-blue-800 hover:bg-blue-700 text-white font-semibold shadow transition"
          >
            💾 Save note
          </button>
          {#if noteMessage}
            <span class="text-sm text-black-200">{noteMessage}</span>
          {/if}
        </div>
      </div>
    </div>
  </section>

  <!-- Mini celebración (mantenida) -->
  {#if showCelebration}
    <div class="fixed left-4 bottom-4 z-50 rounded-xl border border-slate-200 bg-white/90 p-2 shadow">
      <div class="text-xs font-semibold text-slate-700 mb-1">Keep going!</div>
      <img src={celebrationGif} alt="Effort celebration" class="h-24 w-auto rounded-lg" />
    </div>
  {/if}

  <!-- DEBUG panel (mantenido; estilos mínimos) -->
  <div class="fixed right-3 bottom-3 z-50 rounded-xl border border-slate-200 bg-white/90 backdrop-blur p-3 shadow space-y-2">
    <div class="text-xs font-semibold text-slate-600">Debug Streak</div>
    <button
      class="w-full rounded-lg px-3 py-2 border text-sm hover:bg-slate-50"
      on:click={() => {
        const now = new Date();
        const y = new Date(now.getFullYear(), now.getMonth(), now.getDate() - 1);
        const yKey = toLocalYMD(y);
        localStorage.setItem(`checks-${yKey}`, JSON.stringify(habits.map(() => true)));
        localStorage.setItem(`completed-${yKey}`, 'true');
        localStorage.setItem('lastDate', yKey);
        location.reload();
      }}
    >
      ✓ Simulate yesterday complete (+1)
    </button>
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
