<script>
  import { onMount } from 'svelte';

  let timezones = [];
  let selectedTimezone = '';
  let searchQuery = '';
  let timezoneInfo = null;
  let loading = false;
  let error = null;
  let highlightedIndex = -1;
  let searchInput;

  const API_BASE = import.meta.env.PROD ? 'https://epochzone-production.up.railway.app/api' : '/api';
  const API_KEY = import.meta.env.VITE_API_KEY || '';
  const apiHeaders = API_KEY ? { 'X-API-Key': API_KEY } : {};

  onMount(async () => {
    searchInput.focus();
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

  $: filteredTimezones = timezones.filter(tz =>
    tz.display_name.toLowerCase().includes(searchQuery.toLowerCase())
  ).slice(0, 10);

  $: showDropdown = searchQuery && !selectedTimezone && filteredTimezones.length > 0;
</script>

<main>
  <div class="container">
    <div class="logo">
      <h1>🌍 Epoch Zone</h1>
    </div>

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
  }
</style>
