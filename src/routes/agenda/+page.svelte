<script lang="ts">
  import { onMount } from 'svelte';

  type EventItem = { time: string; text: string };

  let events: EventItem[] = [];
  let eventText = '';
  let eventTime = '';

  const KEY = 'agenda-events';

  function loadEvents() {
    const raw = localStorage.getItem(KEY);
    if (raw) {
      try {
        const arr = JSON.parse(raw);
        if (Array.isArray(arr)) {
          events = arr;
          sortEvents();
        }
      } catch {}
    }
  }

  function saveEvents() {
    localStorage.setItem(KEY, JSON.stringify(events));
  }

  function sortEvents() {
    events.sort((a, b) => (a.time < b.time ? -1 : a.time > b.time ? 1 : 0));
  }

  function addEvent() {
    const text = eventText.trim();
    const time = eventTime.trim();
    if (!text || !time) return;

    events = [...events, { time, text }];
    sortEvents();
    saveEvents();

    eventText = '';
    eventTime = '';
  }

  function removeEvent(index: number) {
    events = events.filter((_, i) => i !== index);
    saveEvents();
  }

  onMount(loadEvents);
</script>

<main class="min-h-screen bg-gradient-to-br from-slate-50 via-white to-slate-100 p-6">
  <div class="mx-auto max-w-3xl">
    <header class="mb-6 flex items-center justify-between">
      <h1 class="text-3xl font-extrabold text-slate-800">📅 Schedule</h1>
      <a
        href="/"
        class="rounded-xl border border-slate-200 bg-white px-4 py-2 text-slate-700 hover:bg-slate-50 shadow-sm"
      >
        ← Back
      </a>
    </header>

    <!-- Input for new event -->
    <div class="mb-6 rounded-2xl border border-slate-200 bg-white/90 p-5 shadow-sm">
      <h2 class="text-lg font-semibold text-slate-800 mb-3">Add Event</h2>
      <div class="flex flex-col gap-3 sm:flex-row">
        <input
          type="time"
          bind:value={eventTime}
          class="px-4 py-2 rounded-xl border border-slate-300 focus:outline-none focus:ring-2 focus:ring-indigo-400 shadow-sm"
        />
        <input
          type="text"
          bind:value={eventText}
          placeholder="Event description"
          class="flex-1 px-4 py-2 rounded-xl border border-slate-300 focus:outline-none focus:ring-2 focus:ring-indigo-400 shadow-sm"
        />
        <button
          on:click={addEvent}
          class="px-5 py-2 rounded-xl bg-indigo-600 hover:bg-indigo-700 text-white font-semibold shadow"
        >
          ➕ Add
        </button>
      </div>
    </div>

    <!-- List of events -->
    {#if events.length === 0}
      <p class="text-slate-500">No events scheduled yet.</p>
    {:else}
      <ul class="space-y-3">
        {#each events as ev, i}
          <li class="flex items-center justify-between rounded-xl border border-slate-200 bg-white/90 px-4 py-3 shadow-sm">
            <div>
              <span class="font-semibold text-slate-700">{ev.time}</span> — {ev.text}
            </div>
            <button
              on:click={() => removeEvent(i)}
              class="px-2 py-1 rounded-lg bg-rose-600 hover:bg-rose-700 text-white text-sm shadow"
            >
              Delete
            </button>
          </li>
        {/each}
      </ul>
    {/if}
  </div>
</main>
