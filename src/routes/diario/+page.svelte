<script lang="ts">
  import { onMount } from 'svelte';

  type NoteItem = { date: string; text: string };

  const NOTE_PREFIX = 'note-';
  let notes: NoteItem[] = [];
  let loaded = false;

  function loadNotes() {
    const out: NoteItem[] = [];

    // Recorremos localStorage y recogemos todas las notas
    for (let i = 0; i < localStorage.length; i++) {
      const key = localStorage.key(i);
      if (!key) continue;
      if (key.startsWith(NOTE_PREFIX)) {
        const date = key.slice(NOTE_PREFIX.length); // YYYY-MM-DD
        const text = localStorage.getItem(key) ?? '';
        out.push({ date, text });
      }
    }

    // Orden descendente por fecha (más reciente primero)
    out.sort((a, b) => (a.date < b.date ? 1 : a.date > b.date ? -1 : 0));

    notes = out;
    loaded = true;
  }

  onMount(loadNotes);
</script>

<main class="min-h-screen bg-gray-700 p-6">
  <div class="mx-auto max-w-4xl">
    <header class="mb-6 flex items-center justify-between">
      <h1 class="text-3xl font-extrabold text-white">📔 DIARY</h1>
      <a
        href="/"
        class="rounded-xl border border-red-700 bg-red-600 px-4 py-2 text-white hover:bg-red-700 shadow-sm"
        aria-label="Volver a la página principal"
      >
        ← Back
      </a>
    </header>

    {#if !loaded}
      <p class="text-slate-300">Loading notes...</p>
    {:else if notes.length === 0}
      <div class="rounded-2xl border border-slate-200 bg-gray-600 text-slate-200 shadow-sm">
       There are no saved thoughts yet. Write your first note on the main page.
      </div>
    {:else}
      <ul class="space-y-4">
        {#each notes as n}
          <li class="rounded-2xl border border-sky-400 bg-sky-200 p-5 shadow-sm">
            <div class="mb-2 flex items-center justify-between">
              <h2 class="text-lg font-semibold text-slate-800">{n.date}</h2>
              <button
                class="text-sm rounded-lg border border-slate-200 bg-sky-300 px-2 py-1 hover:bg-blue-600 text-slate-700"
                on:click={() => navigator.clipboard.writeText(n.text)}
                title="Copiar reflexión"
                aria-label="Copiar reflexión"
              >
                Copy
              </button>
            </div>
            <p class="whitespace-pre-wrap text-slate-700">{n.text || '— Sin texto —'}</p>
          </li>
        {/each}
      </ul>
    {/if}
  </div>
</main>
