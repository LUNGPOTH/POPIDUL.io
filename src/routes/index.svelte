<script lang="ts">
  export const ssr = false;
  import { onMount } from 'svelte';
  import { browser } from '$app/env';
  import { tweened } from 'svelte/motion';
  import { cubicOut } from 'svelte/easing';
  import { fade, blur, fly, slide, scale } from 'svelte/transition';
  import axios from 'axios';
  import 'twind/shim';
  import { Howl } from 'howler';
  import { goto } from '$app/navigation';

  import { count } from '../lib/store';
  import { guilds, getName, getSlug } from '../lib/guilds';
  import Kofi from '../lib/Kofi.svelte';

  const audioPath = 'https://storage.googleapis.com/assets.prayut.click/sounds';

  const pops = [
    new Howl({ src: [audioPath + '/pop1.ogg', audioPath + '/pop1.mp3'] }),
    new Howl({ src: [audioPath + '/pop2.ogg', audioPath + '/pop2.mp3'] }),
    new Howl({ src: [audioPath + '/pop3.ogg', audioPath + '/pop3.mp3'] }),
    new Howl({ src: [audioPath + '/pop4.ogg', audioPath + '/pop4.mp3'] }),
  ];

  const comboSounds = [
    new Howl({ src: [audioPath + '/naja.mp3'] }),
    new Howl({ src: [audioPath + '/najaaaaa.mp3'] }),
    new Howl({ src: [audioPath + '/thaiwin.mp3'] }),
    new Howl({ src: [audioPath + '/not-my-senpai.mp3'] }),
  ];

  let najaCount = 100;
  let audioIndex = 0;
  let main: HTMLElement;
  let bgIndex = 0;
  let debounceState = false;
  let pps: number;
  let guildName: string;
  let cityGuild;
  let leaderboardGuilds: Array<any>;
  let showFullLeaderboard = false;
  let showBodyLeader = false;
  let trueTotal = 0;
  let touchEvents = false;

  let lastCount = $count;

  const total = tweened(0, {
    duration: 5000,
    easing: cubicOut,
  });

  const intervalSeconds = 10;
  const axiosInstance = axios.create();
  const images = [
    'images/fre1.png',
    'images/fre2.png"',
  ];

  function incrementCount() {
    if (debounceState) {
      return;
    }
    debounceState = true;

    count.update((n) => n + 1);
    total.set(++trueTotal);
    playPop();
    bgIndex = 1;
  }

  function touchStart() {
    touchEvents = true;
    incrementCount();
  }

  function mouseDown() {
    if (!touchEvents) {
      incrementCount();
    }
  }

  function unlockDebounce() {
    debounceState = false;
    bgIndex = 0;
  }

  function playPop() {
    if ($count % najaCount === 0) {
      const randIndex = ~~(Math.random() * comboSounds.length);
      comboSounds[randIndex].play();
    } else {
      pops[audioIndex].play();
      audioIndex = (audioIndex + 1) % 4;
    }
  }

  function abbreviateNumber(value: number) {
    return new Intl.NumberFormat('th-TH', { notation: 'compact' }).format(value);
  }

  function spin(node: HTMLParagraphElement, t: {}) {
    return {
      duration: 100,
      css: (t: number) => `transform: scale(${1.5 - 0.5 * t}) rotate(${-10 + t * 10}deg);`,
    };
  }

  async function fetchLeaderboard() {
    try {
      const res = await axiosInstance.get('https://api.prayut.click/leaderboard');
      pps = res.data.rate;
      trueTotal = res.data.total;
      total.set(trueTotal);
      leaderboardGuilds = res.data.guilds
        .filter((guild) => guild.total > 0)
        .sort((a, b) => {
          if (a.total === b.total) return 0;
          return a.total > b.total ? -1 : 1;
        });
    } catch (e) {
      console.error(e);
    }
  }

  async function submitCount(count: number, countUpdate: number) {
    if (count == 0) {
      await fetchLeaderboard();
      return;
    }

    try {
      const t = String(Date.now());
      await axiosInstance.post('https://api.prayut.click/clicks', {
        n: count,
        g: cityGuild?.id,
        t,
      });

      lastCount = countUpdate;

      await fetchLeaderboard();
    } catch (e) {
      console.error(e);
    }
  }

  async function fetchGeoData() {
    const ipRes = await axiosInstance.get('https://api.ipify.org?format=json');
    const ip = ipRes.data.ip;

    const geoRes = await axiosInstance.get(`https://reallyfreegeoip.org/json/${ip}`);
    const city = geoRes.data.city;
    cityGuild = guilds.find((g) => g.en.toLowerCase() === city.toLowerCase());

    guildName = getName(cityGuild);

    if (browser) {
      goto(`?g=${getSlug(cityGuild)}`, { replaceState: true });
    }
  }

  function changeGuild(e) {
    const guildId: string = e.target.value;
    cityGuild = guilds.find((g) => g.id === +guildId);

    guildName = getName(cityGuild);

    if (browser) {
      goto(`?g=${getSlug(cityGuild)}`, { replaceState: true });
    }
  }

  function setSubmitCountInterval() {
    setInterval(() => {
      const intervalCount = $count - lastCount;

      submitCount(intervalCount, $count);
    }, intervalSeconds * 1000);
  }

  function setGuildFromQueryString() {
    if (browser) {
      const searchParams = new URLSearchParams(window.location.search);
      const guildParam = searchParams.get('g');

      if (guildParam) {
        cityGuild = guilds.find((g) => getSlug(g) === guildParam);

        guildName = getName(cityGuild);
      } else {
        fetchGeoData();
      }
    }
  }

  function showHideLeaderboard() {
    showFullLeaderboard = !showFullLeaderboard;
    if (showFullLeaderboard) {
      showBodyLeader = showFullLeaderboard;
    } else {
      setTimeout(() => {
        showBodyLeader = showFullLeaderboard;
      }, 400);
    }
  }

  setGuildFromQueryString();
  setSubmitCountInterval();

  onMount(fetchLeaderboard);
</script>

<svelte:body on:keydown={incrementCount} on:keyup={unlockDebounce} />

<svelte:head>
  <title>POPYUT</title>

  <meta name="title" content="POPYUT" />
  <meta name="description" content="A loud-mouthed popping dictator" />
  <meta
    name="viewport"
    content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no"
  />
  <meta property="og:url" content="https://prayut.click" />
  <meta property="og:type" content="website" />
  <meta property="og:title" content="POPYUT" />
  <meta property="og:description" content="A loud-mouthed popping dictator" />
  <meta property="og:image" content={imageUrls[1]} />
  <meta name="twitter:title" content="POPYUT" />
  <meta name="twitter:card" content="summary_large_image" />
  <meta name="twitter:image" content={imageUrls[1]} />

  <!-- Global site tag (gtag.js) - Google Analytics -->
  <script async src="https://www.googletagmanager.com/gtag/js?id=G-6FLPY30SGR"></script>
  <script>
    window.dataLayer = window.dataLayer || [];
    function gtag() {
      dataLayer.push(arguments);
    }
    gtag('js', new Date());
    gtag('config', 'G-6FLPY30SGR');
  </script>

  <link
    rel="stylesheet"
    href="//fonts.googleapis.com/css?family=Permanent+Marker:300,400,600,700&amp;lang=en"
  />
</svelte:head>

<main
  bind:this={main}
  class="w-full h-screen flex flex-col items-center justify-center bg-transparent"
  on:touchstart={touchStart}
  on:mousedown={mouseDown}
  on:touchend={unlockDebounce}
  on:touchcancel={unlockDebounce}
  on:mouseup={unlockDebounce}
>
  {#each imageUrls as url, idx}
    <img
      src={url}
      alt="Popyut"
      class="{idx === bgIndex ? 'block' : 'hidden'} fixed object-cover w-full h-full -z-10"
    />
  {/each}

  <select
    class="justify-center rounded-md border border-gray-300 shadow-sm px-4 py-2 bg-white text-sm font-medium text-gray-700 hover:bg-gray-50 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-offset-gray-100 focus:ring-indigo-500"
    on:change={changeGuild}
  >
    <option>เลือก Guild ของคุณ</option>
    {#each guilds as guild}
      <option value={guild.id} selected={guildName === getName(guild)}>{getName(guild)}</option>
    {/each}
  </select>

  {#key $count}
    <h1 id="localCounter" class="noselect text-7xl text-white mt-4 rounded p-2 items-start" in:spin>
      {$count % 100 ? $count.toLocaleString() : 'POPYUT'}
    </h1>
  {/key}

  <div style="min-height: 70vh" />

  {#if leaderboardGuilds !== undefined}
    <div
      class="modalContent absolute bottom-0 w-80 max-h-36 sm:max-h-20 bg-white rounded-t-xl overflow-y-hidden {showFullLeaderboard &&
        'open'}"
    >
      <div
        class="cursor-pointer border-b whitespace-nowrap overflow-y-hidden p-3 sm:p-3 {showFullLeaderboard
          ? 'pb-3'
          : 'pb-16'}"
        on:click={showHideLeaderboard}
      >
        <div class="flex justify-between items-center {!showFullLeaderboard && 'pb-1 border-b'}">
          <span>Leaderboard</span>
          <span class="text-right">
            <span class="text-xs mr-2 text-green-400"
              >{pps !== undefined ? `${abbreviateNumber(pps)} PPS` : '...'}</span
            >
            {Math.round($total).toLocaleString()}
          </span>
        </div>
        {#if !showFullLeaderboard}
          <div class="flex sm:justify-between items-center pt-2 justify-center">
            {#each leaderboardGuilds.slice(0, 3) as guild, idx}
              <span class={idx === 0 ? 'block' : idx === 1 ? 'hidden sm:block' : 'hidden lg:block'}
                >{idx + 1}. {guild.emoji} {guild.name}: {abbreviateNumber(guild.total)}</span
              >
            {/each}
          </div>
        {/if}
      </div>
      <div class="modalBody p-3 overflow-y-scroll {showFullLeaderboard && 'max-h-60'}">
        {#if showBodyLeader}
          {#each leaderboardGuilds as guild, idx}
            <div class="flex">
              <span class="flex-1">{idx + 1}. {guild.emoji} {guild.name}</span>
              <span>
                {#if guild.rate > 0}
                  <span class="text-green-400 text-xs mr-2">{abbreviateNumber(guild.rate)} PPS</span
                  >
                {/if}
                {guild.total.toLocaleString()}
              </span>
            </div>
          {/each}
        {/if}
      </div>
    </div>
  {/if}

  <Kofi name="narze" />

  <div class="text-xs fixed sm:text-base bottom-4 right-4 text-right z-10">
    <div class="mb-2">
      <a
        href="https://twitter.com/PrayutClick/status/1427336223803072513"
        class="p-1 mb-4 bg-white rounded"
        target="_blank"
        rel="noreferrer">ตั้ง Guild</a
      >
    </div>
    <a
      href="https://twitter.com/PrayutClick"
      class="p-1 bg-white rounded"
      target="_blank"
      rel="noreferrer">@PrayutClick</a
    >
    <a
      href="https://github.com/narze/popyut"
      class="p-1 bg-white rounded"
      target="_blank"
      rel="noreferrer">GitHub</a
    >
    <!--
    <a
      href="https://thailand-grand-opening.web.app"
      class="p-1 bg-white rounded"
      target="_blank"
      rel="noreferrer">120วันเปิดประเทศ?</a
    >
    <a
      href="https://watasalim.vercel.app"
      class="p-1 bg-white rounded"
      target="_blank"
      rel="noreferrer">วาทะสลิ่มสุดเจ๋ง</a
    >
    -->
  </div>
</main>

<style>
  * {
    user-select: none; /* supported by Chrome and Opera */
    -webkit-user-select: none; /* Safari */
    -khtml-user-select: none; /* Konqueror HTML */
    -moz-user-select: none; /* Firefox */
    -ms-user-select: none; /* Internet Explorer/Edge */
  }

  select {
    text-align-last: center;
  }

  #localCounter {
    font-family: 'Permanent Marker';
    -webkit-text-stroke: 3px black;
  }

  .modalContent {
    --max-leaderboard-height: 60vh; /* Change this value according to your needs */
    min-width: 50%;
    transition: max-height 0.4s;
  }

  .modalContent.open {
    /* TODO: Make height not be estimated? */
    max-height: calc(var(--max-leaderboard-height) + 3.05rem) /* 3.05rem is an estimated value */;
  }

  .modalBody {
    max-height: var(--max-leaderboard-height);
    transition: max-height 0.4s;
  }
</style>
