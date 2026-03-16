<!--
 Copyright 2023 Google LLC

 Licensed under the Apache License, Version 2.0 (the "License");
 you may not use this file except in compliance with the License.
 You may obtain a copy of the License at

      https://www.apache.org/licenses/LICENSE-2.0

 Unless required by applicable law or agreed to in writing, software
 distributed under the License is distributed on an "AS IS" BASIS,
 WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 See the License for the specific language governing permissions and
 limitations under the License.
 -->

<script lang="ts">
  /* global google */

  import * as GMAPILoader from '@googlemaps/js-api-loader';
  const { Loader } = GMAPILoader;

  import { onMount } from 'svelte';
  import type { BuildingInsightsResponse } from './solar';
  import Sections from './sections/Sections.svelte';

  const googleMapsApiKey = import.meta.env.VITE_GOOGLE_MAPS_API_KEY;
  const zoom = 19;

  // ── UI state ──────────────────────────────────────────────────────────────
  let showHero = true;
  let location: google.maps.LatLng | undefined;

  // ── Google Maps ───────────────────────────────────────────────────────────
  let mapElement: HTMLElement;
  let map: google.maps.Map;
  let geometryLibrary: google.maps.GeometryLibrary;
  let mapsLibrary: google.maps.MapsLibrary;
  let placesLibrary: google.maps.PlacesLibrary;
  let heroInputElement: HTMLInputElement;

  // ── Sections state (bound to hidden Sections component) ───────────────────
  let buildingInsights: BuildingInsightsResponse | undefined;
  let configId: number | undefined;
  let showPanels = true;
  let showHeatmap = true;
  let monthlyAverageEnergyBillInput = 300;
  let panelCapacityWattsInput = 250;
  let energyCostPerKwhInput = 0.31;
  let dcToAcDerateInput = 0.85;

  // ── UI toggles ────────────────────────────────────────────────────────────
  let batteryStorage = false;
  let electricVehicle = false;

  // ── Financial constants ───────────────────────────────────────────────────
  const installationCostPerWatt = 4.0;
  const solarIncentives = 7000;
  const installationLifeSpan = 20;
  const efficiencyDepreciationFactor = 0.995;
  const costIncreaseFactor = 1.022;
  const discountRate = 1.04;

  // ── Reactive computations ─────────────────────────────────────────────────
  $: solarPanelConfigs = buildingInsights?.solarPotential?.solarPanelConfigs ?? [];
  $: defaultPanelCapacityWatts =
    buildingInsights?.solarPotential?.panelCapacityWatts ?? panelCapacityWattsInput;
  $: panelConfig = configId !== undefined ? solarPanelConfigs[configId] : undefined;
  $: panelCapacityRatio = panelCapacityWattsInput / defaultPanelCapacityWatts;
  $: installationSizeKw = ((panelConfig?.panelsCount ?? 0) * panelCapacityWattsInput) / 1000;
  $: installationCostTotal = installationCostPerWatt * installationSizeKw * 1000;
  $: yearlyKwhEnergyConsumption = (monthlyAverageEnergyBillInput / energyCostPerKwhInput) * 12;
  $: initialAcKwhPerYear =
    (panelConfig?.yearlyEnergyDcKwh ?? 0) * panelCapacityRatio * dcToAcDerateInput;
  $: energyCoveredPct =
    yearlyKwhEnergyConsumption > 0
      ? Math.round((initialAcKwhPerYear / yearlyKwhEnergyConsumption) * 100)
      : 0;

  let breakEvenYear = -1;
  $: {
    const yearlyProduction = [...Array(installationLifeSpan).keys()].map(
      (y) => initialAcKwhPerYear * efficiencyDepreciationFactor ** y,
    );
    const cumWithSolar = (() => {
      let acc = 0;
      return yearlyProduction.map((produced, i) => {
        const bill = Math.max(
          ((yearlyKwhEnergyConsumption - produced) *
            energyCostPerKwhInput *
            costIncreaseFactor ** i) /
            discountRate ** i,
          0,
        );
        return (acc += i === 0 ? bill + installationCostTotal - solarIncentives : bill);
      });
    })();
    const cumWithoutSolar = (() => {
      let acc = 0;
      return [...Array(installationLifeSpan).keys()].map(
        (y) =>
          (acc +=
            (monthlyAverageEnergyBillInput * 12 * costIncreaseFactor ** y) / discountRate ** y),
      );
    })();
    breakEvenYear = cumWithSolar.findIndex((c, i) => c <= cumWithoutSolar[i]);
  }

  // ── Init ──────────────────────────────────────────────────────────────────
  onMount(async () => {
    const loader = new Loader({ apiKey: googleMapsApiKey });
    const libraries = {
      geometry: loader.importLibrary('geometry'),
      maps: loader.importLibrary('maps'),
      places: loader.importLibrary('places'),
    };
    geometryLibrary = await libraries.geometry;
    mapsLibrary = await libraries.maps;
    placesLibrary = await libraries.places;

    map = new mapsLibrary.Map(mapElement, {
      center: { lat: 47.3769, lng: 8.5417 },
      zoom,
      tilt: 0,
      mapTypeId: 'satellite',
      mapTypeControl: false,
      fullscreenControl: false,
      rotateControl: false,
      streetViewControl: false,
      zoomControl: false,
    });

    const autocomplete = new placesLibrary.Autocomplete(heroInputElement, {
      fields: ['formatted_address', 'geometry', 'name'],
    });
    autocomplete.addListener('place_changed', () => {
      const place = autocomplete.getPlace();
      if (!place.geometry?.location) {
        heroInputElement.value = '';
        return;
      }
      location = place.geometry.location;
      map.setCenter(location);
      map.setZoom(zoom);
      showHero = false;
    });
  });

  function handleAnalyze() {
    if (location) showHero = false;
  }

  function useCurrentLocation() {
    if (!navigator.geolocation) return;
    navigator.geolocation.getCurrentPosition((pos) => {
      const latLng = new google.maps.LatLng(pos.coords.latitude, pos.coords.longitude);
      location = latLng;
      map.setCenter(latLng);
      map.setZoom(zoom);
      showHero = false;
    });
  }

  function zoomIn() { map?.setZoom((map.getZoom() ?? zoom) + 1); }
  function zoomOut() { map?.setZoom((map.getZoom() ?? zoom) - 1); }

  function fmt(n: number, decimals = 2) {
    return n.toLocaleString('de-CH', { minimumFractionDigits: decimals, maximumFractionDigits: decimals });
  }
</script>

<!-- ═══════════════════════════════════════════════════════════════════════
     Map (always in DOM)
═══════════════════════════════════════════════════════════════════════ -->
<div class="flex flex-row h-full w-full">
  <div bind:this={mapElement} class="flex-1" />
</div>

<!-- Hidden Sections — handles API calls + renders heatmap/panels on map -->
{#if !showHero && location}
  <div class="hidden">
    <Sections
      bind:buildingInsights
      bind:configId
      bind:showPanels
      bind:showHeatmap
      bind:monthlyAverageEnergyBillInput
      bind:energyCostPerKwhInput
      bind:panelCapacityWattsInput
      bind:dcToAcDerateInput
      {location}
      {map}
      {geometryLibrary}
      {googleMapsApiKey}
    />
  </div>
{/if}

<!-- ═══════════════════════════════════════════════════════════════════════
     HERO OVERLAY  (Screen 1)
═══════════════════════════════════════════════════════════════════════ -->
{#if showHero}
  <div class="absolute inset-0 z-50 flex flex-col p-2.5 gap-2.5 overflow-hidden" style="color-scheme: light">
    <img
      src="https://images.unsplash.com/photo-1508514177221-188b1cf16e9d?w=1920&q=80"
      alt=""
      class="absolute inset-0 w-full h-full object-cover pointer-events-none"
    />
    <div class="absolute inset-0 bg-black/50 pointer-events-none" />

    <!-- Navbar -->
    <nav class="card relative z-10 rounded-full flex items-center justify-between pl-7 pr-3.5 shrink-0">
      <div class="flex items-center gap-2">
        <a href="https://misok.sg-host.com/" class="shrink-0 pr-4">
          <img src="/logo.svg" alt="Meyer Group" class="h-[23px] w-auto" />
        </a>
        <div class="w-px h-8 bg-gray-200" />
        <div class="flex items-center">
          <a href="https://misok.sg-host.com/" class="relative px-5 py-6 text-sm font-display font-bold text-[#030303] no-underline group">
            <span>Home</span>
            <span class="absolute bottom-4 left-5 right-5 h-[1.5px] bg-[#030303] scale-x-0 group-hover:scale-x-100 transition-transform origin-left duration-200" />
          </a>
          <a href="https://misok.sg-host.com/" class="relative px-5 py-6 text-sm font-display font-bold text-[#030303] no-underline group">
            <span>Solarrechner</span>
            <span class="absolute bottom-4 left-5 right-5 h-[1.5px] bg-[#030303] scale-x-100 transition-transform origin-left duration-200" />
          </a>
          <a href="https://misok.sg-host.com/blog/" class="relative px-5 py-6 text-sm font-display font-bold text-[#030303] no-underline group">
            <span>Blog</span>
            <span class="absolute bottom-4 left-5 right-5 h-[1.5px] bg-[#030303] scale-x-0 group-hover:scale-x-100 transition-transform origin-left duration-200" />
          </a>
        </div>
      </div>
      <div class="flex items-center gap-5">
        <a href="tel:+41798726966" class="font-switzer text-[#02B597] text-base font-semibold no-underline whitespace-nowrap flex items-center gap-2">
          <svg class="w-[18px] h-[18px] fill-[#02B597] shrink-0" viewBox="0 0 24 24">
            <path d="M6.62 10.79c1.44 2.83 3.76 5.14 6.59 6.59l2.2-2.2c.27-.27.67-.36 1.02-.24 1.12.37 2.33.57 3.57.57.55 0 1 .45 1 1V20c0 .55-.45 1-1 1-9.39 0-17-7.61-17-17 0-.55.45-1 1-1h3.5c.55 0 1 .45 1 1 0 1.25.2 2.45.57 3.57.11.35.03.74-.25 1.02l-2.2 2.2z"/>
          </svg>
          +41 79 872 69 66
        </a>
        <a href="https://misok.sg-host.com/contact/" class="font-switzer bg-[#02B597] text-[#030303] text-sm font-semibold px-7 py-3.5 rounded-full whitespace-nowrap no-underline hover:brightness-105 transition-all">
          Zum Solarrechner
        </a>
      </div>
    </nav>

    <!-- Hero content -->
    <div class="relative z-10 flex-1 flex items-center justify-center px-4">
      <div class="flex flex-col items-center gap-6 w-full max-w-4xl text-center">
        <div class="flex flex-col gap-3 items-center">
          <h1 class="font-display font-extrabold text-white text-6xl xl:text-[90px] xl:leading-[96px] leading-tight tracking-[-1.8px]">
            Entdecke das Solarpotenzial deines Dachs
          </h1>
          <p class="font-switzer font-semibold text-[#EEEEEE] text-xl xl:text-[22px] leading-7 tracking-[-0.44px]">
            Analysiere dein Zuhause in Sekunden – mit Satellitendaten und präziser Solarmodellierung.
          </p>
        </div>
        <div class="flex gap-2.5 w-full max-w-2xl">
          <div class="flex-1 bg-[#F2F2F2] rounded-[30px] flex items-center gap-4 pl-5 pr-1.5 py-1.5 min-w-0">
            <svg class="w-[18px] h-[18px] shrink-0" viewBox="0 0 24 24" fill="currentColor">
              <path d="M12 2C8.13 2 5 5.13 5 9c0 5.25 7 13 7 13s7-7.75 7-13c0-3.87-3.13-7-7-7zm0 9.5c-1.38 0-2.5-1.12-2.5-2.5s1.12-2.5 2.5-2.5 2.5 1.12 2.5 2.5-1.12 2.5-2.5 2.5z" />
            </svg>
            <input
              bind:this={heroInputElement}
              type="text"
              placeholder="Deine Hausadresse eingeben"
              class="font-switzer flex-1 bg-transparent text-[#8C8C8C] text-base leading-7 tracking-[-0.32px] outline-none placeholder-[#8C8C8C] min-w-0"
            />
            <button on:click={useCurrentLocation} class="location-btn shrink-0" title="Aktuellen Standort verwenden">
              <!-- Default state -->
              <svg class="location-btn__default" width="50" height="50" viewBox="0 0 50 50" fill="none" xmlns="http://www.w3.org/2000/svg">
                <rect width="50" height="50" rx="25" fill="#CECFCF"/>
                <path d="M26.5 25C26.5 24.6022 26.3419 24.2208 26.0605 23.9395C25.7792 23.6581 25.3978 23.5 25 23.5C24.6022 23.5 24.2208 23.6581 23.9395 23.9395C23.6581 24.2208 23.5 24.6022 23.5 25C23.5 25.3978 23.6581 25.7792 23.9395 26.0605C24.2208 26.3419 24.6022 26.5 25 26.5C25.3978 26.5 25.7792 26.3419 26.0605 26.0605C26.3419 25.7792 26.5 25.3978 26.5 25ZM28 25C28 25.7956 27.6837 26.5585 27.1211 27.1211C26.5585 27.6837 25.7956 28 25 28C24.2044 28 23.4415 27.6837 22.8789 27.1211C22.3163 26.5585 22 25.7956 22 25C22 24.2044 22.3163 23.4415 22.8789 22.8789C23.4415 22.3163 24.2044 22 25 22C25.7956 22 26.5585 22.3163 27.1211 22.8789C27.6837 23.4415 28 24.2044 28 25Z" fill="white"/>
                <path d="M30.25 25C30.25 23.6076 29.6972 22.2719 28.7126 21.2874C27.7281 20.3028 26.3924 19.75 25 19.75C23.6076 19.75 22.2719 20.3028 21.2874 21.2874C20.3028 22.2719 19.75 23.6076 19.75 25C19.75 26.3924 20.3028 27.7281 21.2874 28.7126C22.2719 29.6972 23.6076 30.25 25 30.25C26.3924 30.25 27.7281 29.6972 28.7126 28.7126C29.6972 27.7281 30.25 26.3924 30.25 25ZM31.75 25C31.75 26.7902 31.0391 28.5073 29.7732 29.7732C28.5073 31.0391 26.7902 31.75 25 31.75C23.2098 31.75 21.4927 31.0391 20.2268 29.7732C18.9609 28.5073 18.25 26.7902 18.25 25C18.25 23.2098 18.9609 21.4927 20.2268 20.2268C21.4927 18.9609 23.2098 18.25 25 18.25C26.7902 18.25 28.5073 18.9609 29.7732 20.2268C31.0391 21.4927 31.75 23.2098 31.75 25Z" fill="white"/>
                <path d="M25.75 17.5V19H24.25V17.5H25.75Z" fill="white"/>
                <path d="M25.75 31V32.5H24.25V31H25.75Z" fill="white"/>
                <path d="M32.5 24.25V25.75H31V24.25H32.5Z" fill="white"/>
                <path d="M19 24.25V25.75H17.5V24.25H19Z" fill="white"/>
              </svg>
              <!-- Hover state -->
              <svg class="location-btn__hover" width="50" height="50" viewBox="0 0 50 50" fill="none" xmlns="http://www.w3.org/2000/svg">
                <rect width="50" height="50" rx="25" fill="#02B597"/>
                <path d="M26.5 25C26.5 24.6022 26.3419 24.2208 26.0605 23.9395C25.7792 23.6581 25.3978 23.5 25 23.5C24.6022 23.5 24.2208 23.6581 23.9395 23.9395C23.6581 24.2208 23.5 24.6022 23.5 25C23.5 25.3978 23.6581 25.7792 23.9395 26.0605C24.2208 26.3419 24.6022 26.5 25 26.5C25.3978 26.5 25.7792 26.3419 26.0605 26.0605C26.3419 25.7792 26.5 25.3978 26.5 25ZM28 25C28 25.7956 27.6837 26.5585 27.1211 27.1211C26.5585 27.6837 25.7956 28 25 28C24.2044 28 23.4415 27.6837 22.8789 27.1211C22.3163 26.5585 22 25.7956 22 25C22 24.2044 22.3163 23.4415 22.8789 22.8789C23.4415 22.3163 24.2044 22 25 22C25.7956 22 26.5585 22.3163 27.1211 22.8789C27.6837 23.4415 28 24.2044 28 25Z" fill="#030303"/>
                <path d="M30.25 25C30.25 23.6076 29.6972 22.2719 28.7126 21.2874C27.7281 20.3028 26.3924 19.75 25 19.75C23.6076 19.75 22.2719 20.3028 21.2874 21.2874C20.3028 22.2719 19.75 23.6076 19.75 25C19.75 26.3924 20.3028 27.7281 21.2874 28.7126C22.2719 29.6972 23.6076 30.25 25 30.25C26.3924 30.25 27.7281 29.6972 28.7126 28.7126C29.6972 27.7281 30.25 26.3924 30.25 25ZM31.75 25C31.75 26.7902 31.0391 28.5073 29.7732 29.7732C28.5073 31.0391 26.7902 31.75 25 31.75C23.2098 31.75 21.4927 31.0391 20.2268 29.7732C18.9609 28.5073 18.25 26.7902 18.25 25C18.25 23.2098 18.9609 21.4927 20.2268 20.2268C21.4927 18.9609 23.2098 18.25 25 18.25C26.7902 18.25 28.5073 18.9609 29.7732 20.2268C31.0391 21.4927 31.75 23.2098 31.75 25Z" fill="#030303"/>
                <path d="M25.75 17.5V19H24.25V17.5H25.75Z" fill="#030303"/>
                <path d="M25.75 31V32.5H24.25V31H25.75Z" fill="#030303"/>
                <path d="M32.5 24.25V25.75H31V24.25H32.5Z" fill="#030303"/>
                <path d="M19 24.25V25.75H17.5V24.25H19Z" fill="#030303"/>
              </svg>
            </button>
          </div>
          <button on:click={handleAnalyze} class="btn-primary font-switzer rounded-full flex items-center gap-6 pl-7 pr-1.5 py-1.5 shrink-0">
            <span class="btn-primary__text text-sm font-semibold leading-6 tracking-[-0.28px] whitespace-nowrap">Analysieren</span>
            <div class="btn-primary__icon p-4 rounded-full">
              <!-- arrow fly-out: exits top-right, enters from bottom-left -->
              <span class="arrow-out">
                <svg width="18" height="18" viewBox="0 0 18 18" fill="none"><path d="M15.242 4.24333L15.3683 4.3697C13.8066 5.93182 13.8063 8.46457 15.3683 10.0266L13.9541 11.4408C12.805 10.2917 12.2215 8.79178 12.1995 7.28582L4.6609 14.8244L3.24668 13.4102L10.6672 5.98969C9.24429 5.91119 7.84432 5.3296 6.75736 4.24264L8.17157 2.82843C9.73367 4.39052 12.2663 4.39052 13.8284 2.82843L15.242 4.24333Z" fill="#030303"/></svg>
              </span>
              <span class="arrow-in">
                <svg width="18" height="18" viewBox="0 0 18 18" fill="none"><path d="M15.242 4.24333L15.3683 4.3697C13.8066 5.93182 13.8063 8.46457 15.3683 10.0266L13.9541 11.4408C12.805 10.2917 12.2215 8.79178 12.1995 7.28582L4.6609 14.8244L3.24668 13.4102L10.6672 5.98969C9.24429 5.91119 7.84432 5.3296 6.75736 4.24264L8.17157 2.82843C9.73367 4.39052 12.2663 4.39052 13.8284 2.82843L15.242 4.24333Z" fill="#030303"/></svg>
              </span>
            </div>
          </button>
        </div>
      </div>
    </div>

    <!-- Progress steps -->
    <div class="card relative z-10 rounded-full flex items-stretch overflow-hidden shrink-0 w-full">
      <div class="flex-1 bg-[#D7F3EE] flex items-center justify-center gap-2.5 px-7 py-4 -mr-7 rounded-r-full z-10">
        <div class="bg-[#02B597] w-7 h-7 rounded-full flex items-center justify-center shrink-0">
          <span class="font-switzer text-white text-sm font-semibold leading-none">1</span>
        </div>
        <span class="font-switzer text-black text-base font-semibold whitespace-nowrap">Standort auswählen</span>
      </div>
      <div class="flex-1 flex items-center justify-center gap-2.5 px-7 py-4 -mr-7 z-[5]">
        <div class="bg-[#F1F1F1] w-7 h-7 rounded-full flex items-center justify-center shrink-0">
          <span class="font-switzer text-black text-sm font-semibold leading-none">2</span>
        </div>
        <span class="font-switzer text-black text-base font-semibold whitespace-nowrap">Technische Details</span>
      </div>
      <div class="flex-1 flex items-center justify-center gap-2.5 px-7 py-4">
        <div class="bg-[#F1F1F1] w-7 h-7 rounded-full flex items-center justify-center shrink-0">
          <span class="font-switzer text-black text-sm font-semibold leading-none">3</span>
        </div>
        <span class="font-switzer text-black text-base font-semibold whitespace-nowrap">Dein finanzielles Ergebnis</span>
      </div>
    </div>
  </div>
{/if}

<!-- ═══════════════════════════════════════════════════════════════════════
     MAP OVERLAY  (Screen 2)
═══════════════════════════════════════════════════════════════════════ -->
{#if !showHero}
  <div class="absolute inset-0 z-10 flex flex-col p-2.5 gap-2.5 pointer-events-none overflow-hidden" style="color-scheme: light">

    <!-- Navbar -->
    <nav class="card pointer-events-auto rounded-full flex items-center justify-between pl-7 pr-3.5 shrink-0">
      <div class="flex items-center gap-2">
        <a href="https://misok.sg-host.com/" class="shrink-0 pr-4">
          <img src="/logo.svg" alt="Meyer Group" class="h-[23px] w-auto" />
        </a>
        <div class="w-px h-8 bg-gray-200" />
        <div class="flex items-center">
          <a href="https://misok.sg-host.com/" class="relative px-5 py-6 text-sm font-display font-bold text-[#030303] no-underline group">
            <span>Home</span>
            <span class="absolute bottom-4 left-5 right-5 h-[1.5px] bg-[#030303] scale-x-0 group-hover:scale-x-100 transition-transform origin-left duration-200" />
          </a>
          <a href="https://misok.sg-host.com/" class="relative px-5 py-6 text-sm font-display font-bold text-[#030303] no-underline group">
            <span>Solarrechner</span>
            <span class="absolute bottom-4 left-5 right-5 h-[1.5px] bg-[#030303] scale-x-100 transition-transform origin-left duration-200" />
          </a>
          <a href="https://misok.sg-host.com/blog/" class="relative px-5 py-6 text-sm font-display font-bold text-[#030303] no-underline group">
            <span>Blog</span>
            <span class="absolute bottom-4 left-5 right-5 h-[1.5px] bg-[#030303] scale-x-0 group-hover:scale-x-100 transition-transform origin-left duration-200" />
          </a>
        </div>
      </div>

      <div class="flex items-center gap-5">
        <a href="tel:+41798726966" class="font-switzer text-[#02B597] text-base font-semibold no-underline whitespace-nowrap flex items-center gap-2">
          <svg class="w-[18px] h-[18px] fill-[#02B597] shrink-0" viewBox="0 0 24 24">
            <path d="M6.62 10.79c1.44 2.83 3.76 5.14 6.59 6.59l2.2-2.2c.27-.27.67-.36 1.02-.24 1.12.37 2.33.57 3.57.57.55 0 1 .45 1 1V20c0 .55-.45 1-1 1-9.39 0-17-7.61-17-17 0-.55.45-1 1-1h3.5c.55 0 1 .45 1 1 0 1.25.2 2.45.57 3.57.11.35.03.74-.25 1.02l-2.2 2.2z"/>
          </svg>
          +41 79 872 69 66
        </a>
        <a href="https://misok.sg-host.com/contact/" class="font-switzer bg-[#02B597] text-[#030303] text-sm font-semibold px-7 py-3.5 rounded-full whitespace-nowrap no-underline hover:brightness-105 transition-all">
          Zum Solarrechner
        </a>
      </div>
    </nav>

    <!-- Middle: panels over map -->
    <div class="flex-1 flex items-end justify-between min-h-0 gap-2.5">

      <!-- LEFT: Analysis panel + map controls -->
      <div class="flex flex-col h-full justify-between gap-3">

        <!-- Solar analysis panel -->
        {#if buildingInsights && panelConfig}
          <div class="card pointer-events-auto rounded-2xl pt-5 pb-4 px-4 w-[320px] flex flex-col gap-3">
            <h3 class="font-display font-semibold text-xl text-black tracking-[-0.4px]">Solarpotenzial Analyse</h3>

            <!-- 2×2 metrics grid -->
            <div class="grid grid-cols-2 rounded-lg overflow-hidden border border-[#e0e0e0]">
              <!-- Yearly Energy -->
              <div class="metric-cell border-b border-r border-[#e0e0e0]">
                <div class="flex items-start gap-2">
                  <svg class="w-[18px] h-[18px] shrink-0 mt-0.5" viewBox="0 0 24 24" fill="currentColor"><path d="M7 2v11h3v9l7-12h-4l4-8z"/></svg>
                  <span class="font-switzer text-[#2D2B2B] text-[11px] font-semibold leading-4 tracking-[-0.22px]">Jahresenergie</span>
                </div>
                <div class="flex items-end gap-1 mt-2">
                  <span class="font-switzer text-black text-base font-medium tracking-[-0.32px]">{fmt(initialAcKwhPerYear)}</span>
                  <span class="font-switzer text-[#2D2B2B] text-[11px]">kWh/Jahr</span>
                </div>
              </div>
              <!-- Installation Size -->
              <div class="metric-cell border-b border-[#e0e0e0]">
                <div class="flex items-center gap-2">
                  <svg class="w-[18px] h-[18px] shrink-0" viewBox="0 0 24 24" fill="currentColor"><path d="M19 3H5c-1.1 0-2 .9-2 2v14c0 1.1.9 2 2 2h14c1.1 0 2-.9 2-2V5c0-1.1-.9-2-2-2zm-7 14H7v-2h5v2zm5-4H7v-2h10v2zm0-4H7V7h10v2z"/></svg>
                  <span class="font-switzer text-[#2D2B2B] text-[11px] font-semibold leading-4 tracking-[-0.22px]">Anlagengrösse</span>
                </div>
                <div class="flex items-end gap-1 mt-2">
                  <span class="font-switzer text-black text-base font-medium tracking-[-0.32px]">{fmt(installationSizeKw, 1)}</span>
                  <span class="font-switzer text-[#2D2B2B] text-[11px]">kW</span>
                </div>
              </div>
              <!-- Installation Cost -->
              <div class="metric-cell border-r border-[#e0e0e0]">
                <div class="flex items-center gap-2">
                  <svg class="w-[18px] h-[18px] shrink-0" viewBox="0 0 24 24" fill="currentColor"><path d="M12 2C6.48 2 2 6.48 2 12s4.48 10 10 10 10-4.48 10-10S17.52 2 12 2zm1.41 16.09V20h-2.67v-1.93c-1.71-.36-3.16-1.46-3.27-3.4h1.96c.1 1.05.82 1.87 2.65 1.87 1.96 0 2.4-.98 2.4-1.59 0-.83-.44-1.61-2.67-2.14-2.48-.6-4.18-1.62-4.18-3.67 0-1.72 1.39-2.84 3.11-3.21V4h2.67v1.95c1.86.45 2.79 1.86 2.85 3.39H14.3c-.05-1.11-.64-1.87-2.22-1.87-1.5 0-2.4.68-2.4 1.64 0 .84.65 1.39 2.67 1.91s4.18 1.39 4.18 3.91c-.01 1.83-1.38 2.83-3.12 3.16z"/></svg>
                  <span class="font-switzer text-[#2D2B2B] text-[11px] font-semibold leading-4 tracking-[-0.22px]">Installationskosten</span>
                </div>
                <div class="flex items-end gap-1 mt-2">
                  <span class="font-switzer text-[#2D2B2B] text-[11px]">CHF</span>
                  <span class="font-switzer text-black text-base font-medium tracking-[-0.32px]">{fmt(installationCostTotal)}</span>
                </div>
              </div>
              <!-- Energy Coverage -->
              <div class="metric-cell">
                <div class="flex items-center gap-2">
                  <svg class="w-[18px] h-[18px] shrink-0" viewBox="0 0 24 24" fill="currentColor"><path d="M9 21c0 .55.45 1 1 1h4c.55 0 1-.45 1-1v-1H9v1zm3-19C8.14 2 5 5.14 5 9c0 2.38 1.19 4.47 3 5.74V17c0 .55.45 1 1 1h6c.55 0 1-.45 1-1v-2.26c1.81-1.27 3-3.36 3-5.74 0-3.86-3.14-7-7-7zm2.85 11.1l-.85.6V16h-4v-2.3l-.85-.6C7.8 12.16 7 10.63 7 9c0-2.76 2.24-5 5-5s5 2.24 5 5c0 1.63-.8 3.16-2.15 4.1z"/></svg>
                  <span class="font-switzer text-[#2D2B2B] text-[11px] font-semibold leading-4 tracking-[-0.22px]">Energiedeckung</span>
                </div>
                <div class="flex items-end gap-1 mt-2">
                  <span class="font-switzer text-black text-base font-medium tracking-[-0.32px]">{energyCoveredPct}</span>
                  <span class="font-switzer text-[#2D2B2B] text-[11px]">%</span>
                </div>
              </div>
            </div>

            <!-- Break-even bar -->
            <div class="bg-black rounded-lg flex items-center justify-between p-2.5">
              <div class="flex items-center gap-2">
                <svg class="w-[18px] h-[18px] shrink-0" viewBox="0 0 24 24" fill="white"><path d="M3.5 18.49l6-6.01 4 4L22 6.92l-1.41-1.41-7.09 7.97-4-4L2 16.99z"/></svg>
                <span class="font-switzer text-white text-xs font-semibold tracking-[-0.24px]">Break-even</span>
              </div>
              <div class="flex items-center gap-1.5">
                <span class="font-switzer text-white text-base">{breakEvenYear >= 0 ? breakEvenYear + 1 : '—'}</span>
                <span class="font-switzer text-white text-base">Jahre</span>
              </div>
            </div>
          </div>
        {:else}
          <!-- Loading -->
          <div class="card pointer-events-auto rounded-2xl p-6 w-[320px] flex items-center justify-center gap-3">
            <md-circular-progress four-color indeterminate />
            <span class="font-switzer text-sm text-[#8C8C8C]">Gebäudedaten laden…</span>
          </div>
        {/if}

        <!-- Map controls -->
        <div class="flex flex-col gap-2 pointer-events-auto">
          <button on:click={() => location && map.setCenter(location)} class="btn-icon w-[50px] h-[50px] rounded-[12px] flex items-center justify-center">
            <svg class="w-[18px] h-[18px]" viewBox="0 0 24 24" fill="currentColor">
              <path d="M12 8c-2.21 0-4 1.79-4 4s1.79 4 4 4 4-1.79 4-4-1.79-4-4-4zm8.94 3c-.46-4.17-3.77-7.48-7.94-7.94V1h-2v2.06C6.83 3.52 3.52 6.83 3.06 11H1v2h2.06c.46 4.17 3.77 7.48 7.94 7.94V23h2v-2.06c4.17-.46 7.48-3.77 7.94-7.94H23v-2h-2.06zM12 19c-3.87 0-7-3.13-7-7s3.13-7 7-7 7 3.13 7 7-3.13 7-7 7z"/>
            </svg>
          </button>
          <button on:click={zoomIn} class="btn-icon w-[50px] h-[50px] rounded-[12px] flex items-center justify-center">
            <svg class="w-[18px] h-[18px]" viewBox="0 0 24 24" fill="currentColor"><path d="M19 13h-6v6h-2v-6H5v-2h6V5h2v6h6v2z"/></svg>
          </button>
          <button on:click={zoomOut} class="btn-icon w-[50px] h-[50px] rounded-[12px] flex items-center justify-center">
            <svg class="w-[18px] h-[18px]" viewBox="0 0 24 24" fill="currentColor"><path d="M19 13H5v-2h14v2z"/></svg>
          </button>
          <!-- Heatmap toggle -->
          <div class="card rounded-[12px] flex items-center gap-3 pl-4 pr-2.5 py-2.5 w-[185px]">
            <span class="font-switzer text-black text-xs font-semibold tracking-[-0.24px] whitespace-nowrap">Heatmap anzeigen</span>
            <button
              on:click={() => (showHeatmap = !showHeatmap)}
              class="toggle shrink-0 {showHeatmap ? 'toggle--on' : ''}"
            >
              <span class="toggle__knob" />
            </button>
          </div>
        </div>
      </div>

      <!-- RIGHT: Customize panel -->
      <div class="config-card pointer-events-auto rounded-[16px] pt-5 pb-4 px-4 w-[320px] flex flex-col gap-4 h-[599px]">
        <div class="flex flex-col gap-6 flex-1 min-h-0">
          <h3 class="font-switzer font-semibold text-[20px] text-black tracking-[-0.4px] leading-[24px] shrink-0">Solaranlage konfigurieren</h3>

          <div class="flex flex-col gap-4 overflow-y-auto flex-1 min-h-0 pr-1">
            <!-- Number of panels -->
            {#if solarPanelConfigs.length > 0}
              <div class="flex flex-col gap-3">
                <div class="flex items-center gap-2">
                  <svg class="w-[18px] h-[18px] shrink-0" viewBox="0 0 24 24" fill="currentColor"><path d="M3 3h7v7H3zm0 11h7v7H3zm11-11h7v7h-7zm0 11h7v7h-7z"/></svg>
                  <span class="font-switzer text-[#030303] text-xs font-semibold tracking-[-0.24px] flex-1">Anzahl Panels</span>
                  <div class="input-badge">
                    <span class="font-switzer text-[#2D2B2B] text-base tracking-[-0.32px] flex-1">{panelConfig?.panelsCount ?? '—'} Panels</span>
                  </div>
                </div>
                <div class="flex flex-col gap-2">
                  <input
                    type="range"
                    min="0"
                    max={solarPanelConfigs.length - 1}
                    bind:value={configId}
                    class="slider w-full"
                    style="--pct: {((configId ?? 0) / Math.max(solarPanelConfigs.length - 1, 1)) * 100}%"
                  />
                  <div class="flex justify-between font-switzer text-[#8C8C8C] text-[12px] tracking-[-0.24px]">
                    <span>min</span><span>max</span>
                  </div>
                </div>
              </div>
            {/if}

            <!-- Monthly energy bill -->
            <div class="flex flex-col gap-3">
              <div class="flex items-center gap-2">
                <svg class="w-[18px] h-[18px] shrink-0" viewBox="0 0 24 24" fill="currentColor"><path d="M20 4H4c-1.11 0-1.99.89-1.99 2L2 18c0 1.11.89 2 2 2h16c1.11 0 2-.89 2-2V6c0-1.11-.89-2-2-2zm0 14H4v-6h16v6zm0-10H4V6h16v2z"/></svg>
                <span class="font-switzer text-[#030303] text-xs font-semibold tracking-[-0.24px] leading-[16px] flex-1">Ø Monatliche<br/>Stromrechnung</span>
                <div class="input-badge">
                  <input
                    type="number"
                    bind:value={monthlyAverageEnergyBillInput}
                    class="font-switzer text-[#2D2B2B] text-base tracking-[-0.32px] flex-1 w-0 min-w-0 bg-transparent outline-none"
                  />
                </div>
              </div>
              <div class="flex flex-col gap-2">
                <input
                  type="range"
                  min="50"
                  max="1000"
                  step="10"
                  bind:value={monthlyAverageEnergyBillInput}
                  class="slider w-full"
                  style="--pct: {((monthlyAverageEnergyBillInput - 50) / 950) * 100}%"
                />
                <div class="flex justify-between font-switzer text-[#8C8C8C] text-[12px] tracking-[-0.24px]">
                  <span>min</span><span>max</span>
                </div>
              </div>
            </div>

            <!-- Energy cost per kWh -->
            <div class="flex items-center gap-2">
              <svg class="w-[18px] h-[18px] shrink-0" viewBox="0 0 24 24" fill="currentColor"><path d="M12 2C6.48 2 2 6.48 2 12s4.48 10 10 10 10-4.48 10-10S17.52 2 12 2zm1.41 16.09V20h-2.67v-1.93c-1.71-.36-3.16-1.46-3.27-3.4h1.96c.1 1.05.82 1.87 2.65 1.87 1.96 0 2.4-.98 2.4-1.59 0-.83-.44-1.61-2.67-2.14-2.48-.6-4.18-1.62-4.18-3.67 0-1.72 1.39-2.84 3.11-3.21V4h2.67v1.95c1.86.45 2.79 1.86 2.85 3.39H14.3c-.05-1.11-.64-1.87-2.22-1.87-1.5 0-2.4.68-2.4 1.64 0 .84.65 1.39 2.67 1.91s4.18 1.39 4.18 3.91c-.01 1.83-1.38 2.83-3.12 3.16z"/></svg>
              <span class="font-switzer text-[#030303] text-xs font-semibold tracking-[-0.24px] leading-[16px] flex-1">Energiekosten<br/>pro kWh</span>
              <div class="input-badge">
                <input
                  type="number"
                  step="0.01"
                  bind:value={energyCostPerKwhInput}
                  class="font-switzer text-[#8C8C8C] text-base tracking-[-0.32px] flex-1 w-0 min-w-0 bg-transparent outline-none"
                />
              </div>
            </div>

            <!-- Battery storage toggle -->
            <div class="white-row rounded-[24px] flex items-center justify-between px-4 py-[10px] gap-6">
              <span class="font-switzer text-[#030303] text-sm font-semibold tracking-[-0.28px]">Batteriespeicher</span>
              <button
                on:click={() => (batteryStorage = !batteryStorage)}
                class="toggle {batteryStorage ? 'toggle--on' : ''}"
              >
                <span class="toggle__knob" />
              </button>
            </div>

            <!-- Electric vehicle section — expands when active -->
            <div>
              {#if electricVehicle}
                <!-- ACTIVE: dark card -->
                <div class="ev-card rounded-[10px] overflow-hidden">
                  <div class="flex items-center justify-between pl-4 pr-[10px] py-[10px] gap-6">
                    <div class="flex flex-col gap-0.5">
                      <span class="font-switzer text-white text-xs font-semibold tracking-[-0.24px] leading-[16px]">Elektrofahrzeug (EV)</span>
                      <span class="font-switzer text-[#8C8C8C] text-[12px] tracking-[-0.24px] leading-[16px]">EV charging based on solar output</span>
                    </div>
                    <button
                      on:click={() => (electricVehicle = !electricVehicle)}
                      class="toggle toggle--on shrink-0"
                    >
                      <span class="toggle__knob" />
                    </button>
                  </div>
                  <!-- Illustration -->
                  <div class="flex-1 overflow-hidden flex items-center justify-center" style="background:#0C322C;">
                    <svg width="288" height="77" viewBox="0 0 288 77" fill="none">
                      <rect x="0" y="63" width="288" height="14" fill="#041714"/>
                      <rect x="44" y="36" width="108" height="27" rx="5" fill="#014F42"/>
                      <rect x="60" y="22" width="76" height="20" rx="4" fill="#9EDFD4" opacity="0.6"/>
                      <rect x="63" y="24" width="24" height="14" rx="1.5" fill="#02B597" opacity="0.9"/>
                      <rect x="90" y="24" width="24" height="14" rx="1.5" fill="#02B597" opacity="0.9"/>
                      <rect x="117" y="24" width="14" height="14" rx="1.5" fill="#02B597" opacity="0.9"/>
                      <line x1="75" y1="24" x2="75" y2="38" stroke="#014F42" stroke-width="1"/>
                      <line x1="102" y1="24" x2="102" y2="38" stroke="#014F42" stroke-width="1"/>
                      <circle cx="76" cy="64" r="9" fill="#030303"/>
                      <circle cx="76" cy="64" r="4" fill="#1a3a34"/>
                      <circle cx="140" cy="64" r="9" fill="#030303"/>
                      <circle cx="140" cy="64" r="4" fill="#1a3a34"/>
                      <path d="M168 22 L161 40 L168 40 L161 54 L180 34 L172 34 Z" fill="#02B597"/>
                      <rect x="195" y="22" width="34" height="20" rx="2" fill="#014F42"/>
                      <rect x="234" y="16" width="34" height="20" rx="2" fill="#014F42"/>
                      <rect x="209" y="42" width="2" height="10" fill="#014F42"/>
                      <rect x="248" y="36" width="2" height="10" fill="#014F42"/>
                      <rect x="196" y="54" width="54" height="7" rx="3" fill="#041714"/>
                      <rect x="197" y="55" width="40" height="5" rx="2" fill="#02B597"/>
                    </svg>
                  </div>
                </div>
              {:else}
                <!-- INACTIVE: white pill toggle row + CTA hint below -->
                <div class="white-row rounded-[24px] flex items-center justify-between px-4 py-[10px] gap-6">
                  <span class="font-switzer text-[#030303] text-sm font-semibold tracking-[-0.28px]">Elektroauto</span>
                  <button
                    on:click={() => (electricVehicle = !electricVehicle)}
                    class="toggle shrink-0"
                  >
                    <span class="toggle__knob" />
                  </button>
                </div>
                <div class="flex items-center gap-1 px-4 pt-2">
                  <span style="font-family: 'Caveat', cursive; font-size: 16px; font-weight: 500; color: #8C8C8C; letter-spacing: 0;">Jetzt Gratis-Kilometer berechnen</span>
                  <svg width="56" height="40" viewBox="0 -87 220 90" fill="none" class="shrink-0 opacity-35" style="transform: rotate(155deg) scaleX(-1);">
                    <g transform="scale(1,-1)">
                      <path d="M3.17247 25.8421C5.28745 28.1788 7.82542 30.0907 9.7289 32.6399C26.0142 55.3699 49.279 66.6287 75.0817 72.7892C83.9646 74.9135 94.1165 74.7011 103.211 72.7892C129.225 67.6909 152.913 56.8569 173.428 39.8625C179.35 34.9766 184.426 28.8161 188.656 21.8059C186.33 22.6556 183.792 23.5054 181.465 24.5675C174.697 27.5415 167.929 30.728 160.95 33.2772C157.989 34.3393 154.393 34.3393 151.009 34.1269C149.74 34.1269 147.837 32.215 147.625 30.9404C147.414 29.6658 148.683 27.3291 149.74 27.1167C167.718 21.5935 183.369 10.972 199.654 1.83746C205.364 -1.34899 208.96 -0.49927 211.498 5.23635C217.631 19.4692 220.381 34.5517 219.958 50.0592C219.958 50.484 219.112 51.1213 217.843 52.3959C205.364 47.7224 209.171 34.1269 203.038 23.2929C201.557 25.8421 200.5 27.9664 199.231 29.6658C172.582 62.5926 137.262 80.0118 96.2315 86.3848C90.0981 87.4469 83.3301 87.022 76.9852 85.9599C53.7205 81.9237 32.9937 72.1519 15.8623 55.7948C10.3634 50.484 6.34493 43.4738 2.32647 36.8885C0.634492 34.3393 0.634494 30.728 0 27.754C1.05749 27.1167 2.11498 26.4794 3.17247 25.8421Z" fill="#0D1927"/>
                    </g>
                  </svg>
                </div>
              {/if}
            </div>
          </div>
        </div>

        <!-- CTA button -->
        <button class="btn-primary font-switzer rounded-full flex items-center justify-between pl-[30px] pr-[5px] py-[5px] w-full shrink-0 h-[60px]">
          <span class="btn-primary__text text-sm font-semibold leading-6 tracking-[-0.28px]">Analyse starten</span>
          <div class="btn-primary__icon rounded-full">
            <span class="arrow-out">
              <svg width="18" height="18" viewBox="0 0 18 18" fill="none">
                <g clip-path="url(#arrow-clip2a)">
                  <path d="M15.242 4.24333L15.3683 4.3697C13.8066 5.93182 13.8063 8.46457 15.3683 10.0266L13.9541 11.4408C12.805 10.2917 12.2215 8.79178 12.1995 7.28582L4.6609 14.8244L3.24668 13.4102L10.6672 5.98969C9.24429 5.91119 7.84432 5.3296 6.75736 4.24264L8.17157 2.82843C9.73367 4.39052 12.2663 4.39052 13.8284 2.82843L15.242 4.24333Z" fill="#030303"/>
                </g>
                <defs><clipPath id="arrow-clip2a"><rect width="18" height="18" fill="white"/></clipPath></defs>
              </svg>
            </span>
            <span class="arrow-in">
              <svg width="18" height="18" viewBox="0 0 18 18" fill="none">
                <g clip-path="url(#arrow-clip2b)">
                  <path d="M15.242 4.24333L15.3683 4.3697C13.8066 5.93182 13.8063 8.46457 15.3683 10.0266L13.9541 11.4408C12.805 10.2917 12.2215 8.79178 12.1995 7.28582L4.6609 14.8244L3.24668 13.4102L10.6672 5.98969C9.24429 5.91119 7.84432 5.3296 6.75736 4.24264L8.17157 2.82843C9.73367 4.39052 12.2663 4.39052 13.8284 2.82843L15.242 4.24333Z" fill="#030303"/>
                </g>
                <defs><clipPath id="arrow-clip2b"><rect width="18" height="18" fill="white"/></clipPath></defs>
              </svg>
            </span>
          </div>
        </button>
      </div>
    </div>

    <!-- Progress steps -->
    <div class="card pointer-events-auto rounded-full flex items-stretch overflow-hidden shrink-0 w-full">
      <!-- Step 1 ✓ -->
      <div class="flex-1 bg-[#D7F3EE] flex items-center justify-center gap-2.5 px-7 py-4 -mr-7 rounded-r-full z-10">
        <div class="bg-[#02B597] w-7 h-7 rounded-full flex items-center justify-center shrink-0 p-1.5">
          <svg class="w-full h-full" viewBox="0 0 24 24" fill="white"><path d="M9 16.17L4.83 12l-1.42 1.41L9 19 21 7l-1.41-1.41z"/></svg>
        </div>
        <span class="font-switzer text-black text-base font-semibold whitespace-nowrap">Standort auswählen</span>
      </div>
      <!-- Step 2 (active) -->
      <div class="flex-1 bg-[#D7F3EE] flex items-center justify-center gap-2.5 px-7 py-4 -mr-7 rounded-r-full z-[5]">
        <div class="bg-[#02B597] w-7 h-7 rounded-full flex items-center justify-center shrink-0">
          <span class="font-switzer text-white text-sm font-semibold leading-none">2</span>
        </div>
        <span class="font-switzer text-black text-base font-semibold whitespace-nowrap">Technische Details</span>
      </div>
      <!-- Step 3 -->
      <div class="flex-1 flex items-center justify-center gap-2.5 px-7 py-4">
        <div class="bg-[#F1F1F1] w-7 h-7 rounded-full flex items-center justify-center shrink-0">
          <span class="font-switzer text-black text-sm font-semibold leading-none">3</span>
        </div>
        <span class="font-switzer text-black text-base font-semibold whitespace-nowrap">Dein finanzielles Ergebnis</span>
      </div>
    </div>
  </div>
{/if}

<style>
  /*
   * All UI components use explicit CSS instead of Tailwind utility classes for
   * background-color and box-shadow. This prevents browser forced-dark-mode from
   * overriding white surfaces. color-scheme: light on the parent overlay container
   * is the first line of defence; explicit CSS here is the second.
   */

  /* Primary action buttons ("Analysieren", "Analyse starten")
     Default: teal bg, dark text, white icon circle
     Hover:   black bg, white text, teal icon circle            */
  .btn-primary {
    background-color: #02B597;
    transition: background-color 0.2s ease;
  }
  .btn-primary:hover {
    background-color: #030303;
  }
  .btn-primary__text {
    color: #030303;
    transition: color 0.2s ease;
  }
  .btn-primary:hover .btn-primary__text {
    color: #ffffff;
  }
  .btn-primary__icon {
    color-scheme: light;
    background-color: #ffffff;
    transition: background-color 0.2s ease;
    overflow: hidden;
    position: relative;
    width: 50px;
    height: 50px;
    display: flex;
    align-items: center;
    justify-content: center;
  }
  .btn-primary:hover .btn-primary__icon {
    background-color: #02B597;
  }

  /* Arrow fly-out/fly-in — matches misok.sg-host.com button animation exactly:
     translate3d + scale3d combo: icon shrinks as it flies, grows as it arrives */
  .arrow-out,
  .arrow-in {
    display: flex;
    align-items: center;
    justify-content: center;
    position: absolute;
    transition: transform 0.35s linear;
  }
  /* default: arrow-out visible at center */
  .arrow-out {
    transform: translate3d(0px, 0px, 0px) scale3d(1, 1, 1);
  }
  /* default: arrow-in tiny, waiting bottom-left */
  .arrow-in {
    transform: translate3d(-17px, 17px, 0px) scale3d(0, 0, 1);
  }
  /* hover: arrow-out shrinks and flies to top-right */
  .btn-primary:hover .arrow-out {
    transform: translate3d(17px, -17px, 0px) scale3d(0, 0, 1);
  }
  /* hover: arrow-in grows from bottom-left to center */
  .btn-primary:hover .arrow-in {
    transform: translate3d(0px, 0px, 0px) scale3d(1, 1, 1);
  }

  /* Location button (hero search bar) — swaps SVG on hover */
  .location-btn__hover { display: none; }
  .location-btn:hover .location-btn__default { display: none; }
  .location-btn:hover .location-btn__hover { display: block; }
  .location-btn:active { transform: scale(0.95); }

  /* White card panels (navbar, analysis panel, progress bar) */
  .card {
    color-scheme: light;
    background-color: #ffffff;
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.10);
  }

  /* Customize panel — gradient white→grey with blur */
  .config-card {
    color-scheme: light;
    background: linear-gradient(180deg, #FFFFFF 0%, #E0E0E0 100%);
    backdrop-filter: blur(5px);
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.10);
  }

  /* Icon buttons — matches Figma UI Kit Icon_Button_Square states */
  .btn-icon {
    color-scheme: light;
    background-color: #ffffff;
    color: #030303;               /* SVGs inherit this via fill="currentColor" */
    box-shadow: 0 2px 6px rgba(0, 0, 0, 0.10);
    transition: background-color 0.15s ease, color 0.15s ease, box-shadow 0.15s ease, transform 0.1s ease;
  }
  .btn-icon:hover {
    background-color: #030303;
    color: #ffffff;
    box-shadow: 0 4px 10px rgba(0, 0, 0, 0.20);
  }
  .btn-icon:active {
    background-color: #030303;
    color: #ffffff;
    box-shadow: 0 1px 3px rgba(0, 0, 0, 0.20);
    transform: scale(0.95);
  }

  /* Light grey surface for heatmap toggle row */
  .surface-row {
    color-scheme: light;
    background-color: #f2f2f2;
  }

  /* White toggle rows inside config card */
  .white-row {
    color-scheme: light;
    background-color: #ffffff;
  }

  /* Input badge — white pill, 139×38px, 16px text */
  .input-badge {
    color-scheme: light;
    background-color: #ffffff;
    border-radius: 30px;
    padding: 5px 5px 5px 10px;
    height: 38px;
    width: 139px;
    display: flex;
    align-items: center;
    flex-shrink: 0;
  }

  /* EV dark card */
  .ev-card {
    background-color: #041714;
    display: flex;
    flex-direction: column;
  }

  /* Metric cells inside the 2×2 analysis grid */
  .metric-cell {
    color-scheme: light;
    background-color: #ffffff;
    padding: 10px;
    display: flex;
    flex-direction: column;
    justify-content: space-between;
    min-height: 90px;
  }

  /* Toggle switch — matches Figma UI Kit states */
  .toggle {
    position: relative;
    width: 40px;
    height: 22px;
    border-radius: 999px;
    background-color: #CECFCF;
    border: 1.5px solid transparent;
    transition: background-color 0.2s ease, border-color 0.2s ease;
    flex-shrink: 0;
    cursor: pointer;
  }
  /* OFF + Hover: gray bg + teal border */
  .toggle:hover {
    border-color: #02B597;
  }
  /* ON default: teal bg */
  .toggle--on {
    background-color: #02B597;
    border-color: transparent;
  }
  /* ON + Hover: darker teal */
  .toggle--on:hover {
    background-color: #029980;
    border-color: transparent;
  }

  .toggle__knob {
    position: absolute;
    top: 2px;
    left: 2px;
    width: 14px;
    height: 14px;
    border-radius: 50%;
    background-color: #ffffff;
    box-shadow: 0 1px 3px rgba(0, 0, 0, 0.20);
    transition: left 0.2s ease, background-color 0.2s ease;
  }
  /* OFF + Hover: knob gets slightly gray tint */
  .toggle:not(.toggle--on):hover .toggle__knob {
    background-color: #e8e8e8;
  }
  .toggle--on .toggle__knob {
    left: calc(100% - 16px);
  }

  /* Range slider */
  .slider {
    -webkit-appearance: none;
    appearance: none;
    height: 4px;
    border-radius: 4px;
    outline: none;
    cursor: pointer;
    background: linear-gradient(
      to right,
      #02B597 var(--pct, 0%),
      #ffffff var(--pct, 0%)
    );
  }
  .slider::-webkit-slider-thumb {
    -webkit-appearance: none;
    width: 14px;
    height: 12px;
    background-color: #000000;
    clip-path: polygon(0% 0%, 100% 0%, 50% 100%);
    cursor: pointer;
  }
  .slider::-moz-range-thumb {
    width: 14px;
    height: 12px;
    background-color: #000000;
    clip-path: polygon(0% 0%, 100% 0%, 50% 100%);
    cursor: pointer;
    border: none;
    border-radius: 0;
  }
</style>
