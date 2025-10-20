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
      if (!key.startsWith(NOTE_PREFIX)) continue;
      const text = localStorage.getItem(key) ?? '';
      const date = key.replace(NOTE_PREFIX, '');
      out.push({ date, text });
    }

    // Orden descendente por fecha
    out.sort((a, b) => (a.date > b.date ? -1 : a.date < b.date ? 1 : 0));
    notes = out;
    loaded = true;
  }

  onMount(loadNotes);

  let editOpen = false;
  let editNote: NoteItem | null = null;
  let editText = '';

  function openEdit(n: NoteItem) {
    editNote = { ...n };
    editText = n.text;
    editOpen = true;
  }
  function saveEdit() {
    if (!editNote) return;
    localStorage.setItem(NOTE_PREFIX + editNote.date, editText);
    notes = notes.map(x => (x.date === editNote!.date ? { ...x, text: editText } : x));
    editOpen = false;
    editNote = null;
  }
  function cancelEdit() {
    editOpen = false;
    editNote = null;
  }

  let deleteOpen = false;
  let deleteNoteRef: NoteItem | null = null;

  function openDelete(n: NoteItem) {
    deleteNoteRef = n;
    deleteOpen = true;
  }
  function confirmDelete() {
    if (!deleteNoteRef) return;
    localStorage.removeItem(NOTE_PREFIX + deleteNoteRef.date);
    notes = notes.filter(x => x.date !== deleteNoteRef!.date);
    deleteOpen = false;
    deleteNoteRef = null;
  }
  function cancelDelete() {
    deleteOpen = false;
    deleteNoteRef = null;
  }
</script>

<main class="min-h-screen bg-gray-800 text-blue-900 relative p-4 md:p-6">
  <div class="mb-6 flex items-center justify-between">
    <h1 class="text-2xl font-bold text-slate-200 flex items-center gap-2">
      📔 DIARY
    </h1>
    
    <a
      href="/"
      class="text-sm rounded-lg border border-slate-200 bg-red-600 px-3 py-1.5 hover:bg-red-700 text-white"
      aria-label="Back to main"
    >
      ← Back
    </a>
  </div>

  <div class="bg-transparent">
    {#if !loaded}
      <p class="text-slate-300">Loading…</p>
    {:else if !notes.length}
      <p class="text-slate-300">No notes yet.</p>
    {:else}
      <ul class="space-y-6">
        {#each notes as n (n.date)}
          <li class="rounded-xl border border-slate-600/30 bg-sky-200/70 p-3 sm:p-4 shadow-md">
            <div class="flex items-center justify-between gap-2">
              <div class="flex flex-col">
                <p class="text-base sm:text-lg font-semibold text-slate-800">{n.date}</p>
              </div>

              <div class="flex items-center gap-2">
                <button
                  class="text-sm rounded-lg border border-slate-200 bg-sky-300 px-2 py-1 hover:bg-blue-400 text-slate-700"
                  on:click={() => navigator.clipboard.writeText(n.text)}
                  title="Copy note"
                  aria-label="Copy note"
                >
                  Copy
                </button>

                
                <button
                  class="text-sm rounded-lg border border-slate-200 bg-yellow-200 px-2 py-1 hover:bg-yellow-400 text-slate-700"
                  on:click={() => openEdit(n)}
                  title="Edit note"
                  aria-label="Edit note"
                >
                  Edit
                </button>

                
                <button
                  class="text-sm rounded-lg border border-slate-200 bg-red-300 px-2 py-1 hover:bg-red-500 text-slate-700"
                  on:click={() => openDelete(n)}
                  title="Delete note"
                  aria-label="Delete note"
                >
                  Delete
                </button>
              </div>
            </div>

            <p class="whitespace-pre-wrap text-slate-700 mt-2">
              {n.text || '— No text —'}
            </p>
          </li>
        {/each}
      </ul>
    {/if}
  </div>
     
  {#if editOpen}
    <div
      class="fixed inset-0 z-50"
      style="background: rgba(0,0,0,.4); display: grid; place-items: center; padding: 1rem"
      role="dialog"
      aria-modal="true"
      aria-label="Edit note"
    >
      <div class="bg-white rounded-2xl p-4 w-full max-w-lg shadow-xl">
        <h2 class="text-lg font-semibold mb-2">Edit note</h2>
        <p class="text-sm text-slate-500 mb-2">{editNote?.date}</p>
        <textarea
          class="w-full h-48 border border-slate-300 rounded-xl p-2"
          bind:value={editText}
        ></textarea>
        <div class="mt-3 flex gap-2 justify-end">
          <button class="px-3 py-1 rounded-lg border border-slate-200" on:click={cancelEdit}>Cancel</button>
          <button class="px-3 py-1 rounded-lg bg-blue-600 text-white" on:click={saveEdit}>Save</button>
        </div>
      </div>
    </div>
  {/if}

  <!-- Confirmación de borrado -->
  {#if deleteOpen}
    <div
      class="fixed inset-0 z-50"
      style="background: rgba(0,0,0,.4); display: grid; place-items: center; padding: 1rem"
      role="dialog"
      aria-modal="true"
      aria-label="Delete note"
    >
      <div class="bg-white rounded-2xl p-4 w-full max-w-md shadow-xl">
        <h2 class="text-lg font-semibold mb-2">Delete note?</h2>
        <p class="text-slate-600 mb-4">This action cannot be undone.</p>
        <div class="flex gap-2 justify-end">
          <button class="px-3 py-1 rounded-lg border border-slate-200" on:click={cancelDelete}>Cancel</button>
          <button class="px-3 py-1 rounded-lg bg-red-600 text-white" on:click={confirmDelete}>Delete</button>
        </div>
      </div>
    </div>
  {/if}
</main>
