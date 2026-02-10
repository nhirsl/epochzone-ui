<script>
  import { onMount } from 'svelte';

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

  const API_BASE = import.meta.env.PROD ? 'https://epochzone-production.up.railway.app/api' : '/api';
  const API_KEY = import.meta.env.VITE_API_KEY || '';
  const apiHeaders = API_KEY ? { 'X-API-Key': API_KEY } : {};

  onMount(async () => {
    searchInput?.focus();
    await loadTimezones();
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

  function formatTime(isoString) {
    const date = new Date(isoString);
    return date.toLocaleString('en-US', {
      weekday: 'long',
      year: 'numeric',
      month: 'long',
      day: 'numeric',
      hour: '2-digit',
      minute: '2-digit',
      second: '2-digit',
      timeZoneName: 'short'
    });
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

  function formatConvertTime(isoString) {
    const date = new Date(isoString);
    return date.toLocaleString('en-US', {
      weekday: 'long',
      year: 'numeric',
      month: 'long',
      day: 'numeric',
      hour: '2-digit',
      minute: '2-digit',
      second: '2-digit',
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
</script>

<main>
  <div class="container">
    <div class="logo">
      <h1>🌍 Epoch Zone</h1>
    </div>

    <div class="mode-toggle">
      <button class:active={mode === 'lookup'} on:click={() => mode = 'lookup'}>Lookup</button>
      <button class:active={mode === 'convert'} on:click={() => mode = 'convert'}>Convert</button>
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
            <div class="tz-time">{formatConvertTime(convertResult.from.datetime)}</div>
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
            <div class="tz-time">{formatConvertTime(convertResult.to.datetime)}</div>
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

    <footer>
      Powered by Rust + Svelte
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
    padding: 20px;
  }

  .container {
    width: 100%;
    max-width: 600px;
    margin-top: 15vh;
  }

  .logo {
    text-align: center;
    margin-bottom: 30px;
  }

  .logo h1 {
    font-size: 3rem;
    font-weight: 300;
    color: #202124;
    margin: 0;
  }

  .mode-toggle {
    display: flex;
    justify-content: center;
    margin-bottom: 30px;
    background: #f1f3f4;
    border-radius: 20px;
    padding: 4px;
    width: fit-content;
    margin-left: auto;
    margin-right: auto;
  }

  .mode-toggle button {
    padding: 8px 24px;
    border: none;
    border-radius: 20px;
    background: transparent;
    color: #5f6368;
    font-size: 14px;
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
    margin-bottom: 40px;
  }

  .search-box {
    position: relative;
  }

  .search-input {
    width: 100%;
    padding: 14px 20px;
    font-size: 16px;
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
    padding: 12px 20px;
    text-align: left;
    background: none;
    border: none;
    cursor: pointer;
    font-size: 14px;
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
    font-size: 14px;
    padding: 20px;
  }

  .error {
    text-align: center;
    color: #d93025;
    background-color: #fce8e6;
    padding: 12px 20px;
    border-radius: 8px;
    font-size: 14px;
    margin-bottom: 20px;
  }

  .result-card {
    background: #fff;
    border: 1px solid #e8eaed;
    border-radius: 8px;
    padding: 30px;
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
    font-size: 1.5rem;
    font-weight: 400;
    color: #202124;
    margin: 0 0 15px 0;
  }

  .time {
    font-size: 2rem;
    font-weight: 300;
    color: #1a73e8;
    margin-bottom: 25px;
    line-height: 1.3;
  }

  .metadata {
    display: grid;
    gap: 12px;
  }

  .meta-item {
    display: flex;
    justify-content: space-between;
    padding: 8px 0;
    border-bottom: 1px solid #f1f3f4;
  }

  .meta-item:last-child {
    border-bottom: none;
  }

  .label {
    color: #5f6368;
    font-size: 14px;
  }

  .value {
    color: #202124;
    font-weight: 500;
    font-size: 14px;
  }

  /* Convert mode styles */
  .convert-form {
    background: #fff;
    border: 1px solid #e8eaed;
    border-radius: 8px;
    padding: 24px;
    box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
    margin-bottom: 20px;
  }

  .convert-field {
    margin-bottom: 16px;
  }

  .convert-label {
    display: block;
    font-size: 13px;
    font-weight: 500;
    color: #5f6368;
    margin-bottom: 6px;
  }

  .optional-hint {
    font-weight: 400;
    color: #9aa0a6;
  }

  .swap-row {
    display: flex;
    justify-content: center;
    margin-bottom: 16px;
  }

  .swap-btn {
    width: 40px;
    height: 40px;
    border-radius: 50%;
    border: 1px solid #dfe1e5;
    background: #fff;
    font-size: 18px;
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
    padding: 14px 20px;
    font-size: 16px;
    font-weight: 500;
    border: none;
    border-radius: 24px;
    background: #1a73e8;
    color: #fff;
    cursor: pointer;
    transition: background 0.2s;
    margin-top: 8px;
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
    gap: 16px;
    animation: fadeIn 0.3s ease-in;
  }

  .tz-column {
    background: #fff;
    border: 1px solid #e8eaed;
    border-radius: 8px;
    padding: 24px;
    box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
  }

  .tz-label {
    font-size: 12px;
    font-weight: 600;
    color: #1a73e8;
    text-transform: uppercase;
    letter-spacing: 0.5px;
    margin-bottom: 8px;
  }

  .tz-name {
    font-size: 1.2rem;
    font-weight: 400;
    color: #202124;
    margin: 0 0 10px 0;
  }

  .tz-time {
    font-size: 1.1rem;
    font-weight: 300;
    color: #1a73e8;
    margin-bottom: 16px;
    line-height: 1.3;
  }

  .tz-meta {
    display: flex;
    justify-content: space-between;
    padding: 6px 0;
    border-bottom: 1px solid #f1f3f4;
  }

  .tz-meta:last-child {
    border-bottom: none;
  }

  footer {
    text-align: center;
    margin-top: 40px;
    color: #5f6368;
    font-size: 13px;
  }

  @media (max-width: 640px) {
    .container {
      margin-top: 10vh;
    }

    .logo h1 {
      font-size: 2rem;
    }

    .time {
      font-size: 1.5rem;
    }

    .convert-result {
      grid-template-columns: 1fr;
    }
  }
</style>
