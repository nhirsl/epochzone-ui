<!--
  Epoch Zone
  Copyright (C) 2026 Nemanja Hiršl

  This program is free software: you can redistribute it and/or modify
  it under the terms of the GNU General Public License as published by
  the Free Software Foundation, either version 3 of the License, or
  (at your option) any later version.

  This program is distributed in the hope that it will be useful,
  but WITHOUT ANY WARRANTY; without even the implied warranty of
  MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
  GNU General Public License for more details.

  You should have received a copy of the GNU General Public License
  along with this program.  If not, see <https://www.gnu.org/licenses/>.
-->
<script>
  import { onMount, onDestroy } from 'svelte';

  let timezones = [];
  let mode = 'lookup';

  // Lookup mode state
  let selectedTimezone = '';
  let searchQuery = '';
  let timezoneInfo = null;
  let loading = false;
  let error = null;
  let highlightedIndex = -1;
  let searchInput;

  // Convert mode state
  let fromQuery = '';
  let toQuery = '';
  let fromTimezone = '';
  let toTimezone = '';
  let convertDatetime = '';
  let convertResult = null;
  let convertLoading = false;
  let convertError = null;
  let fromHighlightedIndex = -1;
  let toHighlightedIndex = -1;

  // World Clock mode state
  let worldClocks = [];
  let worldClockQuery = '';
  let worldClockHighlightedIndex = -1;
  let currentTick = new Date();
  let tickInterval = null;

  const API_BASE = import.meta.env.PROD ? 'https://epochzone-production.up.railway.app/api' : '/api';
  const API_KEY = import.meta.env.VITE_API_KEY || '';
  const apiHeaders = API_KEY ? { 'X-API-Key': API_KEY } : {};

  onMount(async () => {
    searchInput?.focus();
    await loadTimezones();
    loadWorldClocks();
    startTicking();
  });

  onDestroy(() => {
    stopTicking();
  });

  async function loadTimezones() {
    try {
      const response = await fetch(`${API_BASE}/timezones`, { headers: apiHeaders });
      timezones = await response.json();
    } catch (err) {
      console.error('Failed to load timezones:', err);
    }
  }

  // Lookup mode functions
  async function fetchTimezoneInfo() {
    if (!selectedTimezone) return;

    loading = true;
    error = null;
    timezoneInfo = null;
    let encodedSelectedTimezone = encodeURIComponent(selectedTimezone);

    try {
      const response = await fetch(`${API_BASE}/time/${encodedSelectedTimezone}`, { headers: apiHeaders });
      if (!response.ok) {
        throw new Error('Failed to fetch timezone info');
      }
      timezoneInfo = await response.json();
    } catch (err) {
      error = err.message;
    } finally {
      loading = false;
    }
  }

  function handleTimezoneSelect(timezone) {
    selectedTimezone = timezone;
    searchQuery = timezone.replace(/_/g, ' ');
    highlightedIndex = -1;
    fetchTimezoneInfo();
  }

  function handleKeydown(event) {
    if (!showDropdown) return;

    if (event.key === 'ArrowDown') {
      event.preventDefault();
      highlightedIndex = (highlightedIndex + 1) % filteredTimezones.length;
    } else if (event.key === 'ArrowUp') {
      event.preventDefault();
      highlightedIndex = highlightedIndex <= 0 ? filteredTimezones.length - 1 : highlightedIndex - 1;
    } else if (event.key === 'Enter' && highlightedIndex >= 0) {
      event.preventDefault();
      handleTimezoneSelect(filteredTimezones[highlightedIndex].name);
    } else if (event.key === 'Escape') {
      searchQuery = '';
      highlightedIndex = -1;
    }
  }

  function parseIso(isoString) {
    const [datePart, rest] = isoString.split('T');
    const [year, month, day] = datePart.split('-').map(Number);
    const timePart = rest.replace(/[+-]\d{2}:\d{2}$/, '');
    const [hour, minute, secondRaw] = timePart.split(':');
    return { year, month, day, hour: Number(hour), minute, second: Math.floor(Number(secondRaw)) };
  }

  function formatDate(isoString) {
    const { year, month, day } = parseIso(isoString);
    const weekdays = ['Sunday', 'Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday', 'Saturday'];
    const months = ['January', 'February', 'March', 'April', 'May', 'June',
      'July', 'August', 'September', 'October', 'November', 'December'];
    const weekday = weekdays[new Date(year, month - 1, day).getDay()];
    return `${weekday}, ${months[month - 1]} ${day}, ${year}`;
  }

  function formatTime(isoString) {
    const { hour, minute, second } = parseIso(isoString);
    const h = hour % 12 || 12;
    const ampm = hour >= 12 ? 'PM' : 'AM';
    return `${String(h).padStart(2, '0')}:${minute}:${String(second).padStart(2, '0')} ${ampm}`;
  }

  // Convert mode functions
  function handleFromSelect(timezone) {
    fromTimezone = timezone;
    fromQuery = timezone.replace(/_/g, ' ');
    fromHighlightedIndex = -1;
  }

  function handleToSelect(timezone) {
    toTimezone = timezone;
    toQuery = timezone.replace(/_/g, ' ');
    toHighlightedIndex = -1;
  }

  function handleFromKeydown(event) {
    if (!fromDropdown) return;

    if (event.key === 'ArrowDown') {
      event.preventDefault();
      fromHighlightedIndex = (fromHighlightedIndex + 1) % filteredFromTimezones.length;
    } else if (event.key === 'ArrowUp') {
      event.preventDefault();
      fromHighlightedIndex = fromHighlightedIndex <= 0 ? filteredFromTimezones.length - 1 : fromHighlightedIndex - 1;
    } else if (event.key === 'Enter' && fromHighlightedIndex >= 0) {
      event.preventDefault();
      handleFromSelect(filteredFromTimezones[fromHighlightedIndex].name);
    } else if (event.key === 'Escape') {
      fromQuery = '';
      fromHighlightedIndex = -1;
    }
  }

  function handleToKeydown(event) {
    if (!toDropdown) return;

    if (event.key === 'ArrowDown') {
      event.preventDefault();
      toHighlightedIndex = (toHighlightedIndex + 1) % filteredToTimezones.length;
    } else if (event.key === 'ArrowUp') {
      event.preventDefault();
      toHighlightedIndex = toHighlightedIndex <= 0 ? filteredToTimezones.length - 1 : toHighlightedIndex - 1;
    } else if (event.key === 'Enter' && toHighlightedIndex >= 0) {
      event.preventDefault();
      handleToSelect(filteredToTimezones[toHighlightedIndex].name);
    } else if (event.key === 'Escape') {
      toQuery = '';
      toHighlightedIndex = -1;
    }
  }

  function swapTimezones() {
    const tmpTz = fromTimezone;
    const tmpQuery = fromQuery;
    fromTimezone = toTimezone;
    fromQuery = toQuery;
    toTimezone = tmpTz;
    toQuery = tmpQuery;
  }

  async function convertTime() {
    if (!fromTimezone || !toTimezone) return;

    convertLoading = true;
    convertError = null;
    convertResult = null;

    let datetime = convertDatetime;
    if (!datetime) {
      datetime = new Date().toLocaleString('sv', { timeZone: fromTimezone }).replace(' ', 'T');
    }

    try {
      const response = await fetch(`${API_BASE}/convert`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json', ...apiHeaders },
        body: JSON.stringify({ datetime, from: fromTimezone, to: toTimezone }),
      });
      if (!response.ok) {
        const errBody = await response.json().catch(() => null);
        throw new Error(errBody?.error || 'Failed to convert timezone');
      }
      convertResult = await response.json();
    } catch (err) {
      convertError = err.message;
    } finally {
      convertLoading = false;
    }
  }


  // World Clock functions
  function startTicking() {
    tickInterval = setInterval(() => {
      currentTick = new Date();
    }, 1000);
  }

  function stopTicking() {
    if (tickInterval) {
      clearInterval(tickInterval);
      tickInterval = null;
    }
  }

  async function loadWorldClocks() {
    try {
      const stored = localStorage.getItem('epochzone-world-clocks');
      if (!stored) return;
      const names = JSON.parse(stored);
      if (!Array.isArray(names) || names.length === 0) return;
      const results = await Promise.all(
        names.map(async (tz) => {
          try {
            const response = await fetch(`${API_BASE}/time/${encodeURIComponent(tz)}`, { headers: apiHeaders });
            if (!response.ok) return null;
            const data = await response.json();
            return { timezone: data.timezone, utc_offset: data.utc_offset, abbreviation: data.abbreviation, is_dst: data.is_dst };
          } catch {
            return null;
          }
        })
      );
      worldClocks = results.filter(Boolean);
    } catch (err) {
      console.error('Failed to load world clocks:', err);
    }
  }

  function saveWorldClocks() {
    localStorage.setItem('epochzone-world-clocks', JSON.stringify(worldClocks.map(c => c.timezone)));
  }

  async function addWorldClock(timezoneName) {
    if (worldClocks.some(c => c.timezone === timezoneName)) return;
    try {
      const response = await fetch(`${API_BASE}/time/${encodeURIComponent(timezoneName)}`, { headers: apiHeaders });
      if (!response.ok) return;
      const data = await response.json();
      worldClocks = [...worldClocks, { timezone: data.timezone, utc_offset: data.utc_offset, abbreviation: data.abbreviation, is_dst: data.is_dst }];
      saveWorldClocks();
    } catch (err) {
      console.error('Failed to add world clock:', err);
    }
  }

  function removeWorldClock(timezoneName) {
    worldClocks = worldClocks.filter(c => c.timezone !== timezoneName);
    saveWorldClocks();
  }

  function handleWorldClockSelect(tz) {
    addWorldClock(tz);
    worldClockQuery = '';
    worldClockHighlightedIndex = -1;
  }

  function handleWorldClockKeydown(event) {
    if (!worldClockDropdown) return;

    if (event.key === 'ArrowDown') {
      event.preventDefault();
      worldClockHighlightedIndex = (worldClockHighlightedIndex + 1) % filteredWorldClockTimezones.length;
    } else if (event.key === 'ArrowUp') {
      event.preventDefault();
      worldClockHighlightedIndex = worldClockHighlightedIndex <= 0 ? filteredWorldClockTimezones.length - 1 : worldClockHighlightedIndex - 1;
    } else if (event.key === 'Enter' && worldClockHighlightedIndex >= 0) {
      event.preventDefault();
      handleWorldClockSelect(filteredWorldClockTimezones[worldClockHighlightedIndex].name);
    } else if (event.key === 'Escape') {
      worldClockQuery = '';
      worldClockHighlightedIndex = -1;
    }
  }

  function formatWorldClockTime(timezone, tick) {
    return tick.toLocaleString('en-US', {
      timeZone: timezone,
      hour: '2-digit',
      minute: '2-digit',
      second: '2-digit',
    });
  }

  function formatWorldClockDate(timezone, tick) {
    return tick.toLocaleString('en-US', {
      timeZone: timezone,
      weekday: 'long',
      year: 'numeric',
      month: 'long',
      day: 'numeric',
    });
  }

  // Lookup reactive declarations
  $: filteredTimezones = timezones.filter(tz =>
    tz.display_name.toLowerCase().includes(searchQuery.toLowerCase())
  ).slice(0, 10);

  $: showDropdown = searchQuery && !selectedTimezone && filteredTimezones.length > 0;

  // Convert reactive declarations
  $: filteredFromTimezones = timezones.filter(tz =>
    tz.display_name.toLowerCase().includes(fromQuery.toLowerCase())
  ).slice(0, 10);

  $: filteredToTimezones = timezones.filter(tz =>
    tz.display_name.toLowerCase().includes(toQuery.toLowerCase())
  ).slice(0, 10);

  $: fromDropdown = fromQuery && !fromTimezone && filteredFromTimezones.length > 0;
  $: toDropdown = toQuery && !toTimezone && filteredToTimezones.length > 0;

  // World Clock reactive declarations
  $: filteredWorldClockTimezones = timezones.filter(tz =>
    tz.display_name.toLowerCase().includes(worldClockQuery.toLowerCase()) &&
    !worldClocks.some(c => c.timezone === tz.name)
  ).slice(0, 10);

  $: worldClockDropdown = worldClockQuery && filteredWorldClockTimezones.length > 0;
</script>

<main>
  <div class="container">
    <div class="logo">
      <h1><span class="logo-accent">Epoch</span> Zone</h1>
    </div>

    <div class="mode-toggle">
      <button class:active={mode === 'lookup'} on:click={() => mode = 'lookup'}>Lookup</button>
      <button class:active={mode === 'convert'} on:click={() => mode = 'convert'}>Convert</button>
      <button class:active={mode === 'clock'} on:click={() => mode = 'clock'}>World Clock</button>
    </div>

    {#if mode === 'lookup'}
      <div class="search-container">
        <div class="search-box">
          <input
            type="text"
            bind:this={searchInput}
            bind:value={searchQuery}
            on:input={() => { selectedTimezone = ''; highlightedIndex = -1; }}
            on:keydown={handleKeydown}
            placeholder="Search for a timezone..."
            class="search-input"
          />

          {#if showDropdown}
            <div class="dropdown">
              {#each filteredTimezones as tz, i}
                <button
                  class="dropdown-item"
                  class:highlighted={i === highlightedIndex}
                  on:click={() => handleTimezoneSelect(tz.name)}
                >
                  {tz.display_name}
                </button>
              {/each}
            </div>
          {/if}
        </div>
      </div>

      {#if loading}
        <div class="loading">Loading...</div>
      {/if}

      {#if error}
        <div class="error">{error}</div>
      {/if}

      {#if timezoneInfo}
        <div class="result-card">
          <h2 class="timezone-name">{timezoneInfo.timezone.replace(/_/g, ' ')}</h2>
          <div class="date">{formatDate(timezoneInfo.current_time)}</div>
          <div class="time">{formatTime(timezoneInfo.current_time)}</div>

          <div class="metadata">
            <div class="meta-item">
              <span class="label">UTC Offset:</span>
              <span class="value">{timezoneInfo.utc_offset}</span>
            </div>
            <div class="meta-item">
              <span class="label">Abbreviation:</span>
              <span class="value">{timezoneInfo.abbreviation}</span>
            </div>
            <div class="meta-item">
              <span class="label">DST Active:</span>
              <span class="value">{timezoneInfo.is_dst ? 'Yes' : 'No'}</span>
            </div>
          </div>
        </div>
      {/if}
    {/if}

    {#if mode === 'convert'}
      <div class="convert-form">
        <div class="convert-field">
          <label class="convert-label" for="from-tz">From</label>
          <div class="search-box">
            <input
              id="from-tz"
              type="text"
              bind:value={fromQuery}
              on:input={() => { fromTimezone = ''; fromHighlightedIndex = -1; }}
              on:keydown={handleFromKeydown}
              placeholder="Search source timezone..."
              class="search-input"
            />
            {#if fromDropdown}
              <div class="dropdown">
                {#each filteredFromTimezones as tz, i}
                  <button
                    class="dropdown-item"
                    class:highlighted={i === fromHighlightedIndex}
                    on:click={() => handleFromSelect(tz.name)}
                  >
                    {tz.display_name}
                  </button>
                {/each}
              </div>
            {/if}
          </div>
        </div>

        <div class="swap-row">
          <button class="swap-btn" on:click={swapTimezones} title="Swap timezones">⇅</button>
        </div>

        <div class="convert-field">
          <label class="convert-label" for="to-tz">To</label>
          <div class="search-box">
            <input
              id="to-tz"
              type="text"
              bind:value={toQuery}
              on:input={() => { toTimezone = ''; toHighlightedIndex = -1; }}
              on:keydown={handleToKeydown}
              placeholder="Search target timezone..."
              class="search-input"
            />
            {#if toDropdown}
              <div class="dropdown">
                {#each filteredToTimezones as tz, i}
                  <button
                    class="dropdown-item"
                    class:highlighted={i === toHighlightedIndex}
                    on:click={() => handleToSelect(tz.name)}
                  >
                    {tz.display_name}
                  </button>
                {/each}
              </div>
            {/if}
          </div>
        </div>

        <div class="convert-field">
          <label class="convert-label" for="convert-datetime">Date & time <span class="optional-hint">(optional — defaults to now)</span></label>
          <input
            id="convert-datetime"
            type="datetime-local"
            bind:value={convertDatetime}
            class="search-input datetime-input"
          />
        </div>

        <button
          class="convert-btn"
          on:click={convertTime}
          disabled={!fromTimezone || !toTimezone || convertLoading}
        >
          {convertLoading ? 'Converting...' : 'Convert'}
        </button>
      </div>

      {#if convertLoading}
        <div class="loading">Converting...</div>
      {/if}

      {#if convertError}
        <div class="error">{convertError}</div>
      {/if}

      {#if convertResult}
        <div class="convert-result">
          <div class="tz-column">
            <div class="tz-label">From</div>
            <h3 class="tz-name">{convertResult.from.timezone.replace(/_/g, ' ')}</h3>
            <div class="tz-date">{formatDate(convertResult.from.datetime)}</div>
            <div class="tz-time">{formatTime(convertResult.from.datetime)}</div>
            <div class="tz-meta">
              <span class="label">UTC Offset:</span>
              <span class="value">{convertResult.from.utc_offset}</span>
            </div>
            <div class="tz-meta">
              <span class="label">Abbreviation:</span>
              <span class="value">{convertResult.from.abbreviation}</span>
            </div>
            <div class="tz-meta">
              <span class="label">DST Active:</span>
              <span class="value">{convertResult.from.is_dst ? 'Yes' : 'No'}</span>
            </div>
          </div>
          <div class="tz-column">
            <div class="tz-label">To</div>
            <h3 class="tz-name">{convertResult.to.timezone.replace(/_/g, ' ')}</h3>
            <div class="tz-date">{formatDate(convertResult.to.datetime)}</div>
            <div class="tz-time">{formatTime(convertResult.to.datetime)}</div>
            <div class="tz-meta">
              <span class="label">UTC Offset:</span>
              <span class="value">{convertResult.to.utc_offset}</span>
            </div>
            <div class="tz-meta">
              <span class="label">Abbreviation:</span>
              <span class="value">{convertResult.to.abbreviation}</span>
            </div>
            <div class="tz-meta">
              <span class="label">DST Active:</span>
              <span class="value">{convertResult.to.is_dst ? 'Yes' : 'No'}</span>
            </div>
          </div>
        </div>
      {/if}
    {/if}

    {#if mode === 'clock'}
      <div class="search-container">
        <div class="search-box">
          <label class="sr-only" for="wc-tz">Search and add a timezone</label>
          <input
            id="wc-tz"
            type="text"
            bind:value={worldClockQuery}
            on:input={() => { worldClockHighlightedIndex = -1; }}
            on:keydown={handleWorldClockKeydown}
            placeholder="Search and add a timezone..."
            class="search-input"
          />

          {#if worldClockDropdown}
            <div class="dropdown">
              {#each filteredWorldClockTimezones as tz, i}
                <button
                  class="dropdown-item"
                  class:highlighted={i === worldClockHighlightedIndex}
                  on:click={() => handleWorldClockSelect(tz.name)}
                >
                  {tz.display_name}
                </button>
              {/each}
            </div>
          {/if}
        </div>
      </div>

      {#if worldClocks.length === 0}
        <div class="world-clock-empty">No timezones added yet. Search above to add one.</div>
      {:else}
        <div class="world-clock-list">
          {#each worldClocks as clock (clock.timezone)}
            <div class="world-clock-card">
              <div class="world-clock-header">
                <h2 class="world-clock-name">{clock.timezone.replace(/_/g, ' ')}</h2>
                <button class="remove-btn" on:click={() => removeWorldClock(clock.timezone)} title="Remove">&times;</button>
              </div>
              <div class="world-clock-time">{formatWorldClockTime(clock.timezone, currentTick)}</div>
              <div class="world-clock-date">{formatWorldClockDate(clock.timezone, currentTick)}</div>
              <div class="world-clock-meta">
                {clock.abbreviation} &middot; {clock.utc_offset} &middot; DST: {clock.is_dst ? 'Yes' : 'No'}
              </div>
            </div>
          {/each}
        </div>
      {/if}
    {/if}

    <footer>
      Timezone tools — fast and simple · © 2026 Epoch Zone
    </footer>
  </div>
</main>

<style>
  :global(body) {
    margin: 0;
    padding: 0;
    font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, sans-serif;
    background-color: #fff;
  }

  main {
    min-height: 100vh;
    display: flex;
    flex-direction: column;
    align-items: center;
    padding: clamp(0.75rem, 3vw, 1.25rem);
  }

  .container {
    width: 100%;
    max-width: 600px;
    margin-top: clamp(3vh, 10vw, 15vh);
  }

  .logo {
    text-align: center;
    margin-bottom: clamp(1.25rem, 4vw, 1.875rem);
  }

  .logo h1 {
    font-size: clamp(1.75rem, 5vw, 3rem);
    font-weight: 300;
    color: #202124;
    margin: 0;
  }

  .logo-accent {
    color: #1a73e8;
  }

  .mode-toggle {
    display: flex;
    justify-content: center;
    margin-bottom: clamp(1.25rem, 4vw, 1.875rem);
    background: #f1f3f4;
    border-radius: 20px;
    padding: 4px;
    width: fit-content;
    margin-left: auto;
    margin-right: auto;
  }

  .mode-toggle button {
    padding: clamp(0.375rem, 1.5vw, 0.5rem) clamp(0.875rem, 3vw, 1.5rem);
    border: none;
    border-radius: 20px;
    background: transparent;
    color: #5f6368;
    font-size: clamp(0.8rem, 2vw, 0.875rem);
    font-weight: 500;
    cursor: pointer;
    transition: all 0.2s;
  }

  .mode-toggle button.active {
    background: #1a73e8;
    color: #fff;
  }

  .search-container {
    position: relative;
    margin-bottom: clamp(1.5rem, 5vw, 2.5rem);
  }

  .search-box {
    position: relative;
  }

  .search-input {
    width: 100%;
    padding: clamp(0.75rem, 2vw, 0.875rem) clamp(1rem, 3vw, 1.25rem);
    font-size: clamp(0.9rem, 2.5vw, 1rem);
    border: 1px solid #dfe1e5;
    border-radius: 24px;
    outline: none;
    transition: box-shadow 0.2s;
    box-sizing: border-box;
  }

  .search-input:hover {
    box-shadow: 0 1px 6px rgba(32, 33, 36, 0.28);
  }

  .search-input:focus {
    box-shadow: 0 1px 6px rgba(32, 33, 36, 0.28);
    border-color: transparent;
  }

  .dropdown {
    position: absolute;
    top: calc(100% + 5px);
    left: 0;
    right: 0;
    background: white;
    border: 1px solid #dfe1e5;
    border-radius: 8px;
    box-shadow: 0 4px 6px rgba(32, 33, 36, 0.28);
    max-height: 300px;
    overflow-y: auto;
    z-index: 1000;
  }

  .dropdown-item {
    width: 100%;
    padding: clamp(0.625rem, 2vw, 0.75rem) clamp(1rem, 3vw, 1.25rem);
    text-align: left;
    background: none;
    border: none;
    cursor: pointer;
    font-size: clamp(0.8rem, 2vw, 0.875rem);
    color: #202124;
    transition: background-color 0.1s;
  }

  .dropdown-item:hover,
  .dropdown-item.highlighted {
    background-color: #f1f3f4;
  }

  .loading {
    text-align: center;
    color: #5f6368;
    font-size: 0.875rem;
    padding: 1.25rem;
  }

  .error {
    text-align: center;
    color: #d93025;
    background-color: #fce8e6;
    padding: 0.75rem 1.25rem;
    border-radius: 8px;
    font-size: 0.875rem;
    margin-bottom: 1.25rem;
  }

  .result-card {
    background: #fff;
    border: 1px solid #e8eaed;
    border-radius: 8px;
    padding: clamp(1.25rem, 4vw, 1.875rem);
    box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
    animation: fadeIn 0.3s ease-in;
  }

  @keyframes fadeIn {
    from {
      opacity: 0;
      transform: translateY(10px);
    }
    to {
      opacity: 1;
      transform: translateY(0);
    }
  }

  .timezone-name {
    font-size: clamp(1.1rem, 3.5vw, 1.5rem);
    font-weight: 400;
    color: #202124;
    margin: 0 0 0.9rem 0;
  }

  .date {
    font-size: clamp(0.95rem, 3vw, 1.15rem);
    font-weight: 400;
    color: #555;
    line-height: 1.3;
  }

  .time {
    font-size: clamp(1.3rem, 4.5vw, 2rem);
    font-weight: 300;
    color: #1a73e8;
    margin-bottom: 1.5rem;
    line-height: 1.3;
  }

  .metadata {
    display: grid;
    gap: 0.75rem;
  }

  .meta-item {
    display: flex;
    justify-content: space-between;
    padding: 0.5rem 0;
    border-bottom: 1px solid #f1f3f4;
  }

  .meta-item:last-child {
    border-bottom: none;
  }

  .label {
    color: #5f6368;
    font-size: 0.875rem;
  }

  .value {
    color: #202124;
    font-weight: 500;
    font-size: 0.875rem;
  }

  /* Convert mode styles */
  .convert-form {
    background: #fff;
    border: 1px solid #e8eaed;
    border-radius: 8px;
    padding: clamp(1rem, 3.5vw, 1.5rem);
    box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
    margin-bottom: 1.25rem;
  }

  .convert-field {
    margin-bottom: 1rem;
  }

  .convert-label {
    display: block;
    font-size: 0.8125rem;
    font-weight: 500;
    color: #5f6368;
    margin-bottom: 0.375rem;
  }

  .optional-hint {
    font-weight: 400;
    color: #9aa0a6;
  }

  .swap-row {
    display: flex;
    justify-content: center;
    margin-bottom: 1rem;
  }

  .swap-btn {
    width: 2.5rem;
    height: 2.5rem;
    border-radius: 50%;
    border: 1px solid #dfe1e5;
    background: #fff;
    font-size: 1.125rem;
    cursor: pointer;
    display: flex;
    align-items: center;
    justify-content: center;
    transition: all 0.2s;
    color: #5f6368;
  }

  .swap-btn:hover {
    background: #f1f3f4;
    border-color: #1a73e8;
    color: #1a73e8;
  }

  .datetime-input {
    border-radius: 24px;
    font-family: inherit;
  }

  .convert-btn {
    width: 100%;
    padding: clamp(0.75rem, 2vw, 0.875rem) 1.25rem;
    font-size: clamp(0.9rem, 2.5vw, 1rem);
    font-weight: 500;
    border: none;
    border-radius: 24px;
    background: #1a73e8;
    color: #fff;
    cursor: pointer;
    transition: background 0.2s;
    margin-top: 0.5rem;
  }

  .convert-btn:hover:not(:disabled) {
    background: #1557b0;
  }

  .convert-btn:disabled {
    background: #a8c7fa;
    cursor: not-allowed;
  }

  .convert-result {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 1rem;
    animation: fadeIn 0.3s ease-in;
  }

  .tz-column {
    background: #fff;
    border: 1px solid #e8eaed;
    border-radius: 8px;
    padding: clamp(1rem, 3.5vw, 1.5rem);
    box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
  }

  .tz-label {
    font-size: 0.75rem;
    font-weight: 600;
    color: #1a73e8;
    text-transform: uppercase;
    letter-spacing: 0.5px;
    margin-bottom: 0.5rem;
  }

  .tz-name {
    font-size: clamp(1rem, 3vw, 1.2rem);
    font-weight: 400;
    color: #202124;
    margin: 0 0 0.625rem 0;
  }

  .tz-date {
    font-size: clamp(0.8rem, 2vw, 0.95rem);
    font-weight: 400;
    color: #555;
    line-height: 1.3;
  }

  .tz-time {
    font-size: clamp(0.9rem, 2.5vw, 1.1rem);
    font-weight: 300;
    color: #1a73e8;
    margin-bottom: 1rem;
    line-height: 1.3;
  }

  .tz-meta {
    display: flex;
    justify-content: space-between;
    padding: 0.375rem 0;
    border-bottom: 1px solid #f1f3f4;
  }

  .tz-meta:last-child {
    border-bottom: none;
  }

  /* World Clock mode styles */
  .sr-only {
    position: absolute;
    width: 1px;
    height: 1px;
    padding: 0;
    margin: -1px;
    overflow: hidden;
    clip: rect(0, 0, 0, 0);
    white-space: nowrap;
    border-width: 0;
  }

  .world-clock-list {
    display: flex;
    flex-direction: column;
    gap: 0;
  }

  .world-clock-card {
    background: #fff;
    border: 1px solid #e8eaed;
    padding: clamp(1rem, 3vw, 1.5rem) clamp(1.25rem, 4vw, 1.875rem);
    box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
    animation: fadeIn 0.3s ease-in;
  }

  .world-clock-card:first-child {
    border-radius: 8px 8px 0 0;
  }

  .world-clock-card:last-child {
    border-radius: 0 0 8px 8px;
  }

  .world-clock-card:only-child {
    border-radius: 8px;
  }

  .world-clock-card + .world-clock-card {
    border-top: none;
  }

  .world-clock-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
  }

  .world-clock-name {
    font-size: clamp(1.1rem, 3.5vw, 1.5rem);
    font-weight: 400;
    color: #202124;
    margin: 0 0 0.625rem 0;
  }

  .world-clock-time {
    font-size: clamp(1.4rem, 4.5vw, 2rem);
    font-weight: 300;
    color: #1a73e8;
    margin-bottom: 0.5rem;
    line-height: 1.3;
  }

  .world-clock-date {
    font-size: 0.875rem;
    color: #5f6368;
    margin-bottom: 0.5rem;
  }

  .world-clock-meta {
    font-size: 0.8125rem;
    color: #5f6368;
  }

  .remove-btn {
    background: none;
    border: none;
    font-size: 1.5rem;
    color: #9aa0a6;
    cursor: pointer;
    padding: 0.25rem 0.5rem;
    line-height: 1;
    border-radius: 4px;
    transition: color 0.2s;
  }

  .remove-btn:hover {
    color: #d93025;
  }

  .world-clock-empty {
    text-align: center;
    color: #9aa0a6;
    font-size: 0.875rem;
    padding: 2.5rem 1.25rem;
  }

  footer {
    text-align: center;
    margin-top: clamp(1.5rem, 5vw, 2.5rem);
    color: #5f6368;
    font-size: clamp(0.75rem, 2vw, 0.8125rem);
  }

  /* Only keep breakpoint for layout changes that can't be handled with clamp() */
  @media (max-width: 640px) {
    .convert-result {
      grid-template-columns: 1fr;
    }
  }
</style>
