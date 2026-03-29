<script lang="ts">
  export let installationSizeKw: number;
  export let initialAcKwhPerYear: number;
  export let installationCostTotal: number;
  export let solarIncentives: number;
  export let energyCoveredPct: number;
  export let breakEvenYear: number;
  export let batteryStorage: boolean;
  export let cumWithSolar: number[];
  export let cumWithoutSolar: number[];
  export let installationLifeSpan: number;
  export let savings: number;
  export let yearlyKwhEnergyConsumption: number;
  export let onClose: () => void;

  function fmt(n: number, decimals = 0) {
    return n.toLocaleString('de-CH', { minimumFractionDigits: decimals, maximumFractionDigits: decimals });
  }

  // ── Seasonal selector ─────────────────────────────────────────────────────
  type Season = 'year' | 'spring' | 'summer' | 'autumn' | 'winter';
  let activeSeason: Season = 'year';
  const seasons: { key: Season; label: string }[] = [
    { key: 'year',   label: 'Ganzes Jahr' },
    { key: 'spring', label: 'Frühling'    },
    { key: 'summer', label: 'Sommer'      },
    { key: 'autumn', label: 'Herbst'      },
    { key: 'winter', label: 'Winter Jahr' },
  ];
  const seasonMult: Record<Season, { prod: number; cons: number }> = {
    year:   { prod: 1.00, cons: 1.00 },
    spring: { prod: 1.10, cons: 0.85 },
    summer: { prod: 1.45, cons: 0.70 },
    autumn: { prod: 0.75, cons: 1.05 },
    winter: { prod: 0.30, cons: 1.40 },
  };

  // ── Local toggles ──────────────────────────────────────────────────────────
  let localBattery = batteryStorage;
  let evCharging   = false;

  // EV assumption: 20'000 km/year × 0.18 kWh/km = 3'600 kWh/year
  const EV_KWH_PER_YEAR = 3600;

  // ── Energy flow calculations ───────────────────────────────────────────────
  // Logic:
  //   Solar production → split between: Eigenverbrauch (direct), Batterie (store), Einspeisung (export)
  //   When solar + battery < consumption → Netz-Bezug (grid import)
  //   EV adds to total consumption

  $: sm      = seasonMult[activeSeason];
  $: sProd   = initialAcKwhPerYear * sm.prod;                           // Solar production
  $: evKwh   = evCharging ? EV_KWH_PER_YEAR * sm.cons : 0;             // EV consumption
  $: sCons   = yearlyKwhEnergyConsumption * sm.cons + evKwh;            // Total consumption

  // How much solar can cover directly
  $: maxDirect  = Math.min(sProd, sCons);
  $: directKwh  = maxDirect * (localBattery ? 0.55 : 0.70);            // Direct self-consumption
  $: batteryKwh = localBattery ? maxDirect * 0.25 : 0;                  // Solar → Battery
  $: solarUsed  = directKwh + batteryKwh;                               // Total solar used locally
  $: einspeisKwh= Math.max(0, sProd - solarUsed);                       // Solar → Netz (export)
  $: gridImport = Math.max(0, sCons - solarUsed);                       // Netz → Haus (import)

  // EV share from solar (rough estimate: proportional to solar coverage)
  $: evFromSolar = evCharging && sCons > 0
    ? Math.min(evKwh, solarUsed * (evKwh / sCons)) : 0;

  $: roofPct    = sCons > 0 ? Math.min(100, Math.round((solarUsed / sCons) * 100)) : 0;
  $: netGridPct = 100 - roofPct;

  // kW peak estimates for labels
  $: solarKwPeak = Math.max(0.1, sProd / 365 / 5.5 * 1.8);
  $: gridKwPeak  = Math.max(0,   gridImport / 365 / 14);
  $: battPct     = localBattery && sProd > 0
    ? Math.min(100, Math.round((batteryKwh / (sProd * 0.35)) * 100)) : 0;

  // ── 24h hourly chart ──────────────────────────────────────────────────────
  const HOURS = Array.from({ length: 24 }, (_, i) => i);
  const baseConsProfile = [
    0.20,0.15,0.12,0.12,0.15,0.20,
    0.35,0.50,0.40,0.35,0.30,0.30,
    0.45,0.35,0.30,0.30,0.35,0.50,
    0.60,0.55,0.50,0.45,0.35,0.25,
  ];
  const baseConsSum = baseConsProfile.reduce((a,b) => a+b, 0);
  $: peakKw      = Math.max(0.5, (initialAcKwhPerYear / 365 / 5.5) * 1.8);
  $: consScaleH  = (yearlyKwhEnergyConsumption / 365) / baseConsSum;

  function hourlyProd(h: number, mult: number): number {
    if (h < 6 || h > 20) return 0;
    const t = (h - 13) / 3.5;
    return Math.max(0, peakKw * mult * Math.exp(-t * t * 0.6));
  }
  $: hourlyData = HOURS.map(h => ({
    h,
    prod: hourlyProd(h, sm.prod),
    cons: baseConsProfile[h] * consScaleH * sm.cons * (evCharging ? 1 + evKwh / (yearlyKwhEnergyConsumption * sm.cons || 1) : 1),
    bat:  localBattery ? Math.max(0, Math.min(100, hourlyProd(h, sm.prod) * 15)) : 0,
  }));
  $: maxHourly = Math.max(...hourlyData.map(d => Math.max(d.prod, d.cons)), 1);

  // ── Bar chart SVG ─────────────────────────────────────────────────────────
  const BW=500,BH=185,BP={t:8,r:6,b:28,l:26};
  const bcW=BW-BP.l-BP.r, bcH=BH-BP.t-BP.b;
  const bStep=bcW/24, bBarW=bStep*0.52;
  function bpx(h:number){return BP.l+h*bStep+bStep/2-bBarW/2;}
  function bph(v:number){return (v/maxHourly)*bcH;}
  function bpy(v:number){return BP.t+bcH-bph(v);}

  // ── Line chart SVG ────────────────────────────────────────────────────────
  const LW=430,LH=185,LP={t:8,r:8,b:28,l:44};
  const lcW=LW-LP.l-LP.r, lcH=LH-LP.t-LP.b;
  $: lMax=Math.max(...cumWithoutSolar,1);
  $: lXS=lcW/Math.max(installationLifeSpan-1,1);
  function lpx(i:number){return LP.l+i*lXS;}
  function lpy(v:number){return LP.t+lcH-(v/lMax)*lcH;}
  $: lPathS =cumWithSolar.map((v,i)   =>`${i===0?'M':'L'}${lpx(i).toFixed(1)},${lpy(v).toFixed(1)}`).join(' ');
  $: lPathN =cumWithoutSolar.map((v,i)=>`${i===0?'M':'L'}${lpx(i).toFixed(1)},${lpy(v).toFixed(1)}`).join(' ');
  $: lFillS =lPathS+` L${lpx(installationLifeSpan-1).toFixed(1)},${(LP.t+lcH).toFixed(1)} L${LP.l.toFixed(1)},${(LP.t+lcH).toFixed(1)} Z`;
  $: lFillN =lPathN+` L${lpx(installationLifeSpan-1).toFixed(1)},${(LP.t+lcH).toFixed(1)} L${LP.l.toFixed(1)},${(LP.t+lcH).toFixed(1)} Z`;
  $: lBEX   =breakEvenYear>=0?lpx(breakEvenYear):-999;
  $: lBEY   =breakEvenYear>=0?lpy(cumWithSolar[breakEvenYear]):-999;
  $: lYLab  =[0,0.333,0.667,1].map(f=>({val:lMax*f,y:lpy(lMax*f)}));
  $: lXLab  =Array.from({length:Math.floor(installationLifeSpan/5)+1},(_,i)=>i*5)
    .filter(y=>y<installationLifeSpan)
    .map(y=>({year:new Date().getFullYear()+y,x:lpx(y)}));

  // ── Semi-donut ────────────────────────────────────────────────────────────
  const DR=66,DRI=45,DCX=110,DCY=110;
  function polar(cx:number,cy:number,r:number,deg:number){
    const rad=(deg-90)*Math.PI/180;
    return{x:cx+r*Math.cos(rad),y:cy+r*Math.sin(rad)};
  }
  function arc(cx:number,cy:number,r:number,s:number,e:number){
    const p1=polar(cx,cy,r,s),p2=polar(cx,cy,r,e);
    return`M${p1.x.toFixed(2)},${p1.y.toFixed(2)} A${r},${r} 0 ${(e-s)>180?1:0} 1 ${p2.x.toFixed(2)},${p2.y.toFixed(2)}`;
  }
  $: donutEnd=180+Math.min(roofPct,100)*1.8;
  $: arcBg  =arc(DCX,DCY,DR,180,360);
  $: arcRoof=arc(DCX,DCY,DR,180,donutEnd);
</script>

<style>
  /*
   * Energy flow line animations
   * All paths are drawn in the direction of energy flow,
   * so a single "forward" animation works for all lines.
   */
  @keyframes flow {
    from { stroke-dashoffset: 24; }
    to   { stroke-dashoffset: 0;  }
  }
  .flow { animation: flow 1.8s linear infinite; }
  .flow-slow  { animation: flow 2.4s linear infinite; }
  .flow-fast  { animation: flow 1.3s linear infinite; }

</style>

<div class="absolute inset-0 z-20 overflow-y-auto" style="background:#f6f1eb; color-scheme:light;">

  <!-- Navbar -->
  <nav class="sticky top-0 z-10 bg-white/90 backdrop-blur-md border-b border-[#e0e0e0] flex items-center justify-between px-8 h-[72px]">
    <div class="flex items-center gap-4">
      <a href="https://misok.sg-host.com/" class="shrink-0">
        <img src="/logo.svg" alt="Meyer Group" class="h-[23px] w-auto" />
      </a>
      <div class="w-px h-8 bg-gray-200" />
      <h1 class="font-switzer font-semibold text-[18px] text-[#030303] tracking-[-0.36px]">
        Kostenanalyse über {installationLifeSpan} Jahre
      </h1>
    </div>
    <button on:click={onClose}
      class="flex items-center gap-2 px-4 py-2 rounded-full border border-[#e0e0e0] bg-white hover:bg-[#f7f7f7] transition-colors font-switzer text-sm font-semibold text-[#030303]">
      <svg class="w-4 h-4" viewBox="0 0 24 24" fill="currentColor">
        <path d="M20 11H7.83l5.59-5.59L12 4l-8 8 8 8 1.41-1.41L7.83 13H20v-2z"/>
      </svg>
      Zurück
    </button>
  </nav>

  <div class="max-w-[1400px] mx-auto px-6 py-6 flex flex-col gap-4">

    <!-- ══ Top Panel ══ -->
    <div class="rounded-[16px] overflow-hidden flex flex-col"
      style="background:linear-gradient(108deg,#f9f9f9 1%,#f0ebe3 100%); border:1px solid #ede8e0;">

      <div class="flex">

        <!-- ── House visualization ── -->
        <!--
          SVG coordinate system: viewBox="0 0 100 54.6"
          This matches the image aspect ratio (2816:1536 = 100:54.6 exactly).
          preserveAspectRatio="none" ensures SVG coords map 1:1 to image pixels.

          Key anchor points (x%, y% of image → SVG x, SVG y):
            Solar panels center : 44%, 15% → (44,  8.2)
            Grid pole / wire    :  9%, 29% → ( 9, 15.9)
            Battery box center  : 21%, 46% → (21, 25.2)
            House interior      : 52%, 41% → (52, 22.5)
            EV charger (wall)   : 73%, 47% → (73, 25.7)

          Energy flow paths (drawn FROM source TO destination):
            1. Solar → House    (yellow  #FFE572)
            2. Solar → Battery  (yellow  #FFE572)
            3. Solar → Grid     (teal    #0BDBB8) — ONLY when overproducing
            4. Grid  → House    (blue    #66A3FF) — ONLY when underproducing
            5. Battery → House  (green   #02B597)
            6. House → EV       (purple  #A78BFA)
        -->
        <div class="flex-1 relative" style="aspect-ratio:2816/1536;">

          <img src="/house.png" alt="Haus"
            class="absolute inset-0 w-full h-full"
            style="object-fit:fill;" />

          <!-- SVG flow overlay: coords map 1:1 to image percentage -->
          <svg class="absolute inset-0 w-full h-full pointer-events-none"
            viewBox="0 0 100 54.6"
            preserveAspectRatio="none">

            <defs>
              <!-- Arrow markers for direction clarity -->
              <marker id="arrY" markerWidth="5" markerHeight="5" refX="4" refY="2.5" orient="auto">
                <path d="M0,0 L5,2.5 L0,5 Z" fill="#FFE572" opacity="0.9"/>
              </marker>
              <marker id="arrT" markerWidth="5" markerHeight="5" refX="4" refY="2.5" orient="auto">
                <path d="M0,0 L5,2.5 L0,5 Z" fill="#0BDBB8" opacity="0.9"/>
              </marker>
              <marker id="arrB" markerWidth="5" markerHeight="5" refX="4" refY="2.5" orient="auto">
                <path d="M0,0 L5,2.5 L0,5 Z" fill="#66A3FF" opacity="0.9"/>
              </marker>
              <marker id="arrG" markerWidth="5" markerHeight="5" refX="4" refY="2.5" orient="auto">
                <path d="M0,0 L5,2.5 L0,5 Z" fill="#02B597" opacity="0.9"/>
              </marker>
              <marker id="arrP" markerWidth="5" markerHeight="5" refX="4" refY="2.5" orient="auto">
                <path d="M0,0 L5,2.5 L0,5 Z" fill="#A78BFA" opacity="0.9"/>
              </marker>
            </defs>

            <!--
              Anchor points (SVG viewBox 0 0 100 54.6, preserveAspectRatio="none"):
                Solar panels : (44,  9)  = x44%, y16%
                Grid pole    : (10, 17)  = x10%, y31%
                Powerwall    : (32, 26)  = x32%, y48%  ← on the left wall of the house
                TV/Wohnzimmer: (50, 30)  = x50%, y55%  ← convergence point, living room
                EV charger   : (76, 25)  = x76%, y46%  ← on the garage wall right
            -->

            <!-- 1. Solar → Wohnzimmer -->
            <path d="M 44 9 C 44 18, 50 25, 50 30"
              fill="none" stroke="#FFE572" stroke-width="0.7"
              stroke-dasharray="3.5 2" marker-end="url(#arrY)"
              class="flow"/>

            <!-- 2. Solar → Powerwall battery -->
            {#if localBattery}
              <path d="M 43 9 C 41 15, 37 20, 32 26"
                fill="none" stroke="#FFE572" stroke-width="0.7"
                stroke-dasharray="3.5 2" marker-end="url(#arrY)"
                class="flow-slow"/>
            {/if}

            <!-- 3. Solar → Einspeisung (only when overproducing) -->
            {#if einspeisKwh > 100}
              <path d="M 42 9 C 34 11, 22 13, 10 16"
                fill="none" stroke="#0BDBB8" stroke-width="0.6"
                stroke-dasharray="3 2" marker-end="url(#arrT)"
                class="flow-slow"/>
            {/if}

            <!-- 4. Netz → Wohnzimmer (only when underproducing) -->
            {#if gridImport > 100}
              <path d="M 10 17 C 22 22, 38 27, 50 30"
                fill="none" stroke="#66A3FF" stroke-width="0.7"
                stroke-dasharray="3.5 2" marker-end="url(#arrB)"
                class="flow"/>
            {/if}

            <!-- 5. Powerwall → Wohnzimmer -->
            {#if localBattery}
              <path d="M 32 26 C 38 28, 45 29, 50 30"
                fill="none" stroke="#02B597" stroke-width="0.7"
                stroke-dasharray="3.5 2" marker-end="url(#arrG)"
                class="flow-fast"/>
            {/if}

            <!-- 6. Wohnzimmer → EV Ladestation -->
            {#if evCharging}
              <path d="M 50 30 C 58 28, 68 26, 76 25"
                fill="none" stroke="#A78BFA" stroke-width="0.7"
                stroke-dasharray="3.5 2" marker-end="url(#arrP)"
                class="flow"/>
            {/if}

          </svg>

          <!-- ── Floating label cards ── -->
          <!-- Positioned using image percentages (left%, top%) -->

          <!-- SOLAR (über den Panels, oben-Mitte) -->
          <div class="absolute" style="left:40%; top:11%;">
            <div class="rounded-[10px] px-3 py-2 shadow-lg" style="background:rgba(3,3,3,0.84); backdrop-filter:blur(8px); min-width:125px;">
              <div class="flex items-center gap-1.5 mb-0.5">
                <div class="w-2 h-2 rounded-full bg-[#FFE572]" />
                <span class="font-switzer text-[10px] text-[#FFE572]">Solar</span>
              </div>
              <p class="font-switzer font-semibold text-[17px] text-white leading-none">{fmt(solarKwPeak,1)} kW</p>
              <p class="font-switzer text-[10px] text-[#8C8C8C] mt-0.5">{fmt(sProd)} kWh/Jahr</p>
            </div>
          </div>

          <!-- EINSPEISUNG (links oben, nur wenn Überschuss) — helles KPI-Design -->
          {#if einspeisKwh > 100}
            <div class="absolute" style="left:20%; top:15%;">
              <div class="rounded-[10px] px-3 py-2 shadow-lg" style="background:rgba(230,252,248,0.96); border:1px solid rgba(11,219,184,0.35); backdrop-filter:blur(8px); min-width:120px;">
                <div class="flex items-center gap-1.5 mb-0.5">
                  <div class="w-2 h-2 rounded-full bg-[#0BDBB8]" />
                  <span class="font-switzer text-[10px] text-[#00897B]">↑ Netz-Einspeisung</span>
                </div>
                <p class="font-switzer font-semibold text-[17px] text-[#004D40] leading-none">{fmt(einspeisKwh)} kWh</p>
                <p class="font-switzer text-[10px] text-[#4CAF9A] mt-0.5">Überschuss → Netz</p>
              </div>
            </div>
          {/if}

          <!-- NETZ-BEZUG (links beim Mast, nur wenn Import nötig)
               Klar: Strom kommt VOM Netz ins Haus -->
          <div class="absolute" style="left:0.5%; top:30%;">
            <div class="rounded-[10px] px-3 py-2 shadow-lg" style="background:rgba(3,3,3,0.84); backdrop-filter:blur(8px); min-width:110px;">
              <div class="flex items-center gap-1.5 mb-0.5">
                <div class="w-2 h-2 rounded-full" style="background:{gridImport > 100 ? '#66A3FF' : '#555'}" />
                <span class="font-switzer text-[10px]" style="color:{gridImport > 100 ? '#66A3FF' : '#555'}">
                  {gridImport > 100 ? '→ Netz-Bezug' : 'Netz'}
                </span>
              </div>
              <p class="font-switzer font-semibold text-[17px] leading-none" style="color:{gridImport > 100 ? 'white' : '#666'}">
                {fmt(gridKwPeak,1)} kW
              </p>
              <p class="font-switzer text-[10px] text-[#8C8C8C] mt-0.5">
                {gridImport > 100 ? fmt(gridImport)+' kWh/Jahr' : 'Kein Bezug'}
              </p>
            </div>
          </div>

          <!-- BATTERIE (Powerwall, links an der Hauswand) -->
          {#if localBattery}
            <div class="absolute" style="left:17%; top:60%;">
              <div class="rounded-[10px] px-3 py-2 shadow-lg" style="background:rgba(3,3,3,0.84); backdrop-filter:blur(8px); min-width:130px;">
                <div class="flex items-center gap-1.5 mb-0.5">
                  <div class="w-2 h-2 rounded-full bg-[#02B597]" />
                  <span class="font-switzer text-[10px] text-[#02B597]">Batterie</span>
                </div>
                <div class="flex items-baseline gap-2">
                  <p class="font-switzer font-semibold text-[17px] text-white leading-none">{fmt(batteryKwh)} kWh</p>
                  <span class="font-switzer text-[12px] text-[#02B597]">· {battPct}%</span>
                </div>
                <p class="font-switzer text-[10px] text-[#8C8C8C] mt-0.5">Speicher aktiv</p>
              </div>
            </div>
          {/if}

          <!-- GESAMTVERBRAUCH — am Konvergenzpunkt (Wohnzimmer/TV) -->
          <div class="absolute" style="left:50%; top:60%;">
            <div class="rounded-[10px] px-3 py-2 shadow-lg" style="background:rgba(255,252,240,0.97); border:1px solid rgba(255,229,114,0.45); backdrop-filter:blur(8px); min-width:150px;">
              <div class="flex items-center gap-1.5 mb-0.5">
                <div class="w-2 h-2 rounded-full bg-[#F59E0B]" />
                <span class="font-switzer text-[10px] text-[#92400E]">Gesamtverbrauch</span>
              </div>
              <p class="font-switzer font-semibold text-[17px] text-[#1C1917] leading-none">{fmt(sCons)} kWh/Jahr</p>
              <p class="font-switzer text-[10px] text-[#78716C] mt-0.5">Solar-Anteil: {fmt(solarUsed)} kWh</p>
            </div>
          </div>

          <!-- EIGENVERBRAUCH — direkt vom Dach, oben rechts -->
          <div class="absolute" style="left:45%; top:30%;">
            <div class="rounded-[10px] px-3 py-2 shadow-lg" style="background:rgba(3,3,3,0.84); backdrop-filter:blur(8px); min-width:135px;">
              <div class="flex items-center gap-1.5 mb-0.5">
                <div class="w-2 h-2 rounded-full bg-[#FFE572]" />
                <span class="font-switzer text-[10px] text-[#FFE572]">Eigenverbrauch Solar</span>
              </div>
              <p class="font-switzer font-semibold text-[17px] text-white leading-none">{fmt(directKwh)} kWh</p>
              <p class="font-switzer text-[10px] text-[#8C8C8C] mt-0.5">direkt vom Dach</p>
            </div>
          </div>

          <!-- EV LADESTATION (rechts, beim Garageneingang) -->
          {#if evCharging}
            <div class="absolute" style="left:70%; top:50%;">
              <div class="rounded-[10px] px-3 py-2 shadow-lg" style="background:rgba(3,3,3,0.84); backdrop-filter:blur(8px); min-width:130px;">
                <div class="flex items-center gap-1.5 mb-0.5">
                  <div class="w-2 h-2 rounded-full bg-[#A78BFA]" />
                  <span class="font-switzer text-[10px] text-[#A78BFA]">Ladestation</span>
                </div>
                <p class="font-switzer font-semibold text-[17px] text-white leading-none">{fmt(evKwh)} kWh</p>
                <p class="font-switzer text-[10px] text-[#8C8C8C] mt-0.5">20'000 km/Jahr</p>
              </div>
            </div>
          {/if}

        </div>

        <!-- ── Seasonal selector + Toggles ── -->
        <div class="flex flex-col gap-2 p-4 shrink-0 w-[190px]"
          style="background:rgba(246,241,235,0.8);">
          <p class="font-switzer font-semibold text-[10px] text-[#8C8C8C] tracking-[0.5px] uppercase mb-1">Zeitraum</p>
          {#each seasons as s}
            <button on:click={() => activeSeason = s.key}
              class="flex items-center gap-2.5 pl-[5px] pr-4 py-[5px] rounded-[64px] transition-all duration-200 text-left w-full"
              style="background:{activeSeason===s.key?'#02B597':'white'}">
              <div class="w-[32px] h-[32px] rounded-full flex items-center justify-center shrink-0"
                style="background:{activeSeason===s.key?'rgba(215,243,238,0.4)':'#f2f2f2'}">
                {#if activeSeason===s.key}
                  <svg class="w-[16px] h-[16px]" viewBox="0 0 24 24" fill="white">
                    <path d="M9 16.17L4.83 12l-1.42 1.41L9 19 21 7l-1.41-1.41z"/>
                  </svg>
                {/if}
              </div>
              <span class="font-switzer font-semibold text-[13px] tracking-[-0.26px]"
                style="color:{activeSeason===s.key?'white':'#030303'}">{s.label}</span>
            </button>
          {/each}

          <!-- Divider -->
          <div class="mt-2 mb-1 border-t border-[#ddd]" />

          <!-- Batterie Toggle -->
          <div class="flex items-center justify-between px-1">
            <div class="flex items-center gap-1.5">
              <svg class="w-[13px] h-[13px] shrink-0" viewBox="0 0 24 24" fill="#02B597">
                <path d="M15.67 4H14V2h-4v2H8.33C7.6 4 7 4.6 7 5.33v15.33C7 21.4 7.6 22 8.33 22h7.33c.74 0 1.34-.6 1.34-1.33V5.33C17 4.6 16.4 4 15.67 4zm-1.67 9h-2v4h-2v-4H8l4-8 4 8h-2z"/>
              </svg>
              <span class="font-switzer font-semibold text-[12px] text-[#030303]">Batterie</span>
            </div>
            <button on:click={() => localBattery=!localBattery}
              class="relative w-[36px] h-[20px] rounded-full shrink-0 transition-colors duration-200"
              style="background:{localBattery?'#02B597':'#CECFCF'}">
              <div class="absolute top-[2px] w-[16px] h-[16px] bg-white rounded-full shadow-sm transition-all duration-200"
                style="left:{localBattery?'calc(100% - 18px)':'2px'}"/>
            </button>
          </div>

          <!-- Ladestation Toggle -->
          <div class="flex items-center justify-between px-1">
            <div class="flex items-center gap-1.5">
              <svg class="w-[13px] h-[13px] shrink-0" viewBox="0 0 24 24" fill="#A78BFA">
                <path d="M18.92 6.01C18.72 5.42 18.16 5 17.5 5h-11c-.66 0-1.21.42-1.42 1.01L3 12v8c0 .55.45 1 1 1h1c.55 0 1-.45 1-1v-1h12v1c0 .55.45 1 1 1h1c.55 0 1-.45 1-1v-8l-2.08-5.99zM6.85 7h10.29l1.08 3.11H5.77L6.85 7zM19 17H5v-5h14v5z"/>
                <circle cx="7.5" cy="14.5" r="1.5" fill="#A78BFA"/>
                <circle cx="16.5" cy="14.5" r="1.5" fill="#A78BFA"/>
              </svg>
              <span class="font-switzer font-semibold text-[12px] text-[#030303]">Ladestation</span>
            </div>
            <button on:click={() => evCharging=!evCharging}
              class="relative w-[36px] h-[20px] rounded-full shrink-0 transition-colors duration-200"
              style="background:{evCharging?'#A78BFA':'#CECFCF'}">
              <div class="absolute top-[2px] w-[16px] h-[16px] bg-white rounded-full shadow-sm transition-all duration-200"
                style="left:{evCharging?'calc(100% - 18px)':'2px'}"/>
            </button>
          </div>

        </div>
      </div>

      <!-- ── Stats bar ── -->
      <div class="flex items-stretch gap-2 p-4 pt-3" style="height:72px;">

        <!-- Ihr System -->
        <div class="bg-black flex items-center gap-3 pl-3 pr-5 py-2 rounded-[8px] shrink-0">
          <svg class="w-[15px] h-[15px] text-white shrink-0" viewBox="0 0 24 24" fill="currentColor">
            <path d="M2.667 2.667h8v8h-8zm10.666 0h8v8h-8zm0 10.666h8v8h-8zM2.667 13.333h8v8h-8z"/>
          </svg>
          <div class="flex flex-col justify-between h-full py-1">
            <span class="font-switzer text-[10px] text-[#8C8C8C]">Solaranlage</span>
            <span class="font-switzer font-semibold text-[13px] text-white">{fmt(installationSizeKw,1)} kW</span>
          </div>
          <div class="w-px h-full bg-[#2a2a2a] mx-1"/>
          <div class="flex flex-col justify-between h-full py-1">
            <span class="font-switzer text-[10px] text-[#8C8C8C]">Jahresproduktion</span>
            <span class="font-switzer font-semibold text-[13px] text-white">{fmt(initialAcKwhPerYear)} kWh</span>
          </div>
        </div>

        <!-- Einsparung -->
        <div class="flex-1 flex flex-col justify-between p-2.5 rounded-[8px]"
          style="background:rgba(255,255,255,0.7); border:1px solid #efefef;">
          <div class="flex items-center gap-1.5">
            <svg class="w-[12px] h-[12px]" viewBox="0 0 24 24" fill="#2D2B2B">
              <path d="M3.5 18.49l6-6.01 4 4L22 6.92l-1.41-1.41-7.09 7.97-4-4L2 16.99z"/>
            </svg>
            <span class="font-switzer font-semibold text-[10px] text-[#2D2B2B]">Einsparung</span>
          </div>
          <div class="flex items-end gap-1">
            <span class="font-switzer text-[10px] text-[#2D2B2B]">CHF</span>
            <span class="font-switzer font-medium text-[14px] text-black tracking-[-0.3px]">{fmt(savings)}</span>
          </div>
        </div>

        <!-- Break Even -->
        <div class="flex-1 flex flex-col justify-between p-2.5 rounded-[8px]"
          style="background:rgba(255,255,255,0.7); border:1px solid #efefef;">
          <div class="flex items-center gap-1.5">
            <svg class="w-[12px] h-[12px]" viewBox="0 0 24 24" fill="#2D2B2B">
              <path d="M12 2C6.48 2 2 6.48 2 12s4.48 10 10 10 10-4.48 10-10S17.52 2 12 2zm-2 14.5v-9l6 4.5-6 4.5z"/>
            </svg>
            <span class="font-switzer font-semibold text-[10px] text-[#2D2B2B]">Break Even</span>
          </div>
          <div class="flex items-end gap-1">
            <span class="font-switzer font-medium text-[14px] text-black tracking-[-0.3px]">{breakEvenYear>=0?breakEvenYear+1:'—'}</span>
            <span class="font-switzer text-[10px] text-[#2D2B2B]">Jahre</span>
          </div>
        </div>

        <!-- Installationskosten -->
        <div class="flex-1 flex flex-col justify-between p-2.5 rounded-[8px]"
          style="background:rgba(255,255,255,0.7); border:1px solid #efefef;">
          <div class="flex items-center gap-1.5">
            <svg class="w-[12px] h-[12px]" viewBox="0 0 24 24" fill="#2D2B2B">
              <path d="M11.8 10.9c-2.27-.59-3-1.2-3-2.15 0-1.09 1.01-1.85 2.7-1.85 1.78 0 2.44.85 2.5 2.1h2.21c-.07-1.72-1.12-3.3-3.21-3.81V3h-3v2.16c-1.94.42-3.5 1.68-3.5 3.61 0 2.31 1.91 3.46 4.7 4.13 2.5.6 3 1.48 3 2.41 0 .69-.49 1.79-2.7 1.79-2.06 0-2.87-.92-2.98-2.1h-2.2c.12 2.19 1.76 3.42 3.68 3.83V21h3v-2.15c1.95-.37 3.5-1.5 3.5-3.55 0-2.84-2.43-3.81-4.7-4.4z"/>
            </svg>
            <span class="font-switzer font-semibold text-[10px] text-[#2D2B2B]">Installationskosten</span>
          </div>
          <div class="flex items-end gap-1">
            <span class="font-switzer text-[10px] text-[#2D2B2B]">CHF</span>
            <span class="font-switzer font-medium text-[14px] text-black tracking-[-0.3px]">{fmt(installationCostTotal-solarIncentives)}</span>
          </div>
        </div>

        <!-- EIV Förderung -->
        <div class="flex-1 flex flex-col justify-between p-2.5 rounded-[8px]"
          style="background:rgba(255,255,255,0.7); border:1px solid #efefef;">
          <div class="flex items-center gap-1.5">
            <svg class="w-[12px] h-[12px]" viewBox="0 0 24 24" fill="#02B597">
              <path d="M12 1L3 5v6c0 5.55 3.84 10.74 9 12 5.16-1.26 9-6.45 9-12V5l-9-4zm-2 16l-4-4 1.41-1.41L10 14.17l6.59-6.59L18 9l-8 8z"/>
            </svg>
            <span class="font-switzer font-semibold text-[10px] text-[#2D2B2B]">EIV Förderung</span>
          </div>
          <div class="flex items-end gap-1">
            <span class="font-switzer text-[10px] text-[#2D2B2B]">CHF</span>
            <span class="font-switzer font-medium text-[14px] text-[#02B597] tracking-[-0.3px]">{fmt(solarIncentives)}</span>
          </div>
        </div>


      </div>
    </div>

    <!-- ══ Charts Row ══ -->
    <div class="grid grid-cols-[1fr_1fr_272px] gap-4">

      <!-- Bar chart: 24h -->
      <div class="bg-white rounded-[16px] p-5 flex flex-col gap-3">
        <div class="flex items-start justify-between gap-2">
          <h3 class="font-switzer font-medium text-[13px] text-[#2D2B2B] tracking-[-0.26px] leading-[18px]">
            Tagesproduktion &amp; Verbrauch
          </h3>
          <div class="flex gap-1 shrink-0 flex-wrap justify-end">
            {#if localBattery}
              <div class="flex items-center gap-1 bg-[#f2f2f2] rounded px-2 py-1">
                <div class="w-[5px] h-[12px] rounded" style="background:#FFE572"/>
                <span class="font-switzer font-semibold text-[9px] text-[#8C8C8C]">Batterie</span>
              </div>
            {/if}
            <div class="flex items-center gap-1 bg-[#f2f2f2] rounded px-2 py-1">
              <div class="w-[5px] h-[12px] rounded" style="background:#0BDBB8"/>
              <span class="font-switzer font-semibold text-[9px] text-[#8C8C8C]">Produktion (kW)</span>
            </div>
            <div class="flex items-center gap-1 bg-[#f2f2f2] rounded px-2 py-1">
              <div class="w-[5px] h-[12px] rounded" style="background:#66A3FF"/>
              <span class="font-switzer font-semibold text-[9px] text-[#8C8C8C]">Verbrauch (kWh)</span>
            </div>
          </div>
        </div>
        <svg viewBox="0 0 {BW} {BH}" class="w-full" style="overflow:visible">
          {#each [0.25,0.5,0.75,1] as f}
            <line x1={BP.l} y1={BP.t+bcH*(1-f)} x2={BW-BP.r} y2={BP.t+bcH*(1-f)} stroke="#f0f0f0" stroke-width="1"/>
          {/each}
          {#each hourlyData as d}
            <rect x={bpx(d.h)} y={bpy(d.cons)} width={bBarW} height={bph(d.cons)} fill="#66A3FF" fill-opacity="0.35" rx="2"/>
          {/each}
          {#each hourlyData as d}
            <rect x={bpx(d.h)} y={bpy(d.prod)} width={bBarW} height={bph(d.prod)} fill="#0BDBB8" fill-opacity="0.75" rx="2"/>
          {/each}
          {#if localBattery}
            <polyline
              points={hourlyData.map(d=>`${(bpx(d.h)+bBarW/2).toFixed(1)},${(BP.t+bcH-(d.bat/100)*bcH).toFixed(1)}`).join(' ')}
              fill="none" stroke="#FFE572" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/>
          {/if}
          <line x1={BP.l} y1={BP.t+bcH} x2={BW-BP.r} y2={BP.t+bcH} stroke="#e0e0e0" stroke-width="1"/>
          {#each [0,6,12,18,23] as h}
            <text x={bpx(h)+bBarW/2} y={BH-5} text-anchor="middle" font-size="9" fill="#8C8C8C" font-family="Switzer,sans-serif">{h}</text>
          {/each}
          <text x={(BP.l+BW-BP.r)/2} y={BH} text-anchor="middle" font-size="8" fill="#8C8C8C" font-family="Switzer,sans-serif">Stunden</text>
          <text x={BP.l-3} y={BP.t+4} text-anchor="end" font-size="8" fill="#8C8C8C" font-family="Switzer,sans-serif">{maxHourly.toFixed(1)}</text>
          <text x={BP.l-3} y={BP.t+bcH/2+3} text-anchor="end" font-size="8" fill="#8C8C8C" font-family="Switzer,sans-serif">{(maxHourly/2).toFixed(1)}</text>
          <text x="6" y={BP.t+bcH/2} text-anchor="middle" font-size="8" fill="#8C8C8C" font-family="Switzer,sans-serif" transform="rotate(-90 6 {BP.t+bcH/2})">kW</text>
        </svg>
      </div>

      <!-- Line chart -->
      <div class="bg-white rounded-[16px] p-5 flex flex-col gap-3">
        <div class="flex items-center justify-between">
          <h3 class="font-switzer font-medium text-[13px] text-[#2D2B2B] tracking-[-0.26px]">
            Kostenvergleich über {installationLifeSpan} Jahre
          </h3>
          <div class="flex gap-1.5">
            <div class="flex items-center gap-1 bg-[#f2f2f2] rounded px-2.5 py-1">
              <div class="w-[5px] h-[13px] rounded" style="background:#FFE572"/>
              <span class="font-switzer font-semibold text-[9px] text-[#8C8C8C]">Mit Solar</span>
            </div>
            <div class="flex items-center gap-1 bg-[#f2f2f2] rounded px-2.5 py-1">
              <div class="w-[5px] h-[13px] rounded" style="background:#66A3FF"/>
              <span class="font-switzer font-semibold text-[9px] text-[#8C8C8C]">Ohne Solar</span>
            </div>
          </div>
        </div>
        <svg viewBox="0 0 {LW} {LH}" class="w-full" style="overflow:visible">
          {#each lYLab as {val,y}}
            <line x1={LP.l} y1={y} x2={LW-LP.r} y2={y} stroke="#e0e0e0" stroke-width="1"/>
            <text x={LP.l-4} y={y+3} text-anchor="end" font-size="9" fill="#8C8C8C" font-family="Switzer,sans-serif">
              {val>=1000?Math.round(val/1000)+'k':Math.round(val)}
            </text>
          {/each}
          <text x="8" y={LP.t+lcH/2} text-anchor="middle" font-size="8" fill="#8C8C8C" font-family="Switzer,sans-serif" transform="rotate(-90 8 {LP.t+lcH/2})">CHF</text>
          <path d={lFillN} fill="#66A3FF" fill-opacity="0.10"/>
          <path d={lFillS} fill="#FFE572" fill-opacity="0.20"/>
          <path d={lPathN} fill="none" stroke="#66A3FF" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/>
          <path d={lPathS} fill="none" stroke="#FFE572" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round"/>
          {#if breakEvenYear>=0}
            <line x1={lBEX} y1={LP.t} x2={lBEX} y2={LP.t+lcH} stroke="#030303" stroke-width="1" stroke-dasharray="4 3"/>
            <circle cx={lBEX} cy={lBEY} r="4" fill="#030303"/>
            <rect x={lBEX-26} y={lBEY-20} width="52" height="16" rx="4" fill="#030303"/>
            <text x={lBEX} y={lBEY-9} text-anchor="middle" font-size="8" fill="white" font-family="Switzer,sans-serif" font-weight="600">Break-even</text>
          {/if}
          {#each lXLab as {year,x}}
            <text x={x} y={LH-5} text-anchor="middle" font-size="9" fill="#8C8C8C" font-family="Switzer,sans-serif">{year}</text>
          {/each}
          <line x1={LP.l} y1={LP.t+lcH} x2={LW-LP.r} y2={LP.t+lcH} stroke="#e0e0e0" stroke-width="1"/>
        </svg>
      </div>

      <!-- Donut: vom Dach -->
      <div class="bg-white rounded-[16px] p-5 flex flex-col gap-4">
        <h3 class="font-switzer font-medium text-[13px] text-[#2D2B2B] tracking-[-0.26px] leading-[18px]">
          Wie viel Ihres Strombedarfs kommt vom Dach?
        </h3>
        <div class="flex-1 flex flex-col items-center justify-between gap-4">
          <svg viewBox="0 0 220 125" class="w-full max-w-[220px]">
            <path d={arcBg} fill="none" stroke="#E0E0E0" stroke-width="{DR-DRI}" stroke-linecap="butt"/>
            {#if roofPct>0}
              <path d={arcRoof} fill="none" stroke="#FFE572" stroke-width="{DR-DRI}" stroke-linecap="butt"/>
            {/if}
            <text x={DCX} y={DCY-14} text-anchor="middle" font-size="10" fill="#8C8C8C" font-family="Switzer,sans-serif">Energiebedarf</text>
            <text x={DCX} y={DCY+8} text-anchor="middle" font-size="22" font-weight="600" fill="#030303" font-family="Switzer,sans-serif">{roofPct}%</text>
            <text x={DCX} y={DCY+24} text-anchor="middle" font-size="9" fill="#8C8C8C" font-family="Switzer,sans-serif">{fmt(yearlyKwhEnergyConsumption)} kWh/Jahr</text>
          </svg>
          <div class="flex gap-4 w-full">
            <div class="flex gap-2 items-start flex-1">
              <div class="w-[6px] h-[13px] rounded shrink-0 mt-0.5" style="background:#FFE572"/>
              <div class="flex flex-col gap-0.5">
                <span class="font-switzer text-[10px] text-[#8C8C8C]">Strom vom Dach</span>
                <span class="font-switzer font-semibold text-[18px] text-[#030303] tracking-[-0.36px]">{roofPct}%</span>
              </div>
            </div>
            <div class="flex gap-2 items-start flex-1">
              <div class="w-[6px] h-[13px] rounded shrink-0 mt-0.5" style="background:#66A3FF"/>
              <div class="flex flex-col gap-0.5">
                <span class="font-switzer text-[10px] text-[#8C8C8C]">Strom aus dem Netz</span>
                <span class="font-switzer font-semibold text-[18px] text-[#030303] tracking-[-0.36px]">{netGridPct}%</span>
              </div>
            </div>
          </div>
        </div>
      </div>

    </div>

    <!-- ══ CTA ══ -->
    <div class="rounded-[16px] px-16 py-14 flex flex-col items-center gap-7 text-center"
      style="background:#f6f1eb; border:1px solid #e0d8ce;">
      <h2 class="font-display font-bold text-[48px] text-[#2D2B2B] tracking-[-0.96px] leading-[56px] max-w-[700px]">
        Sehen Sie Ihre Rendite und Ihr Einsparpotenzial
      </h2>
      <a href="https://misok.sg-host.com/"
        class="flex items-center gap-6 bg-[#02B597] rounded-full pl-[30px] pr-[5px] py-[5px] h-[60px] no-underline hover:bg-[#02a388] transition-colors">
        <span class="font-switzer font-semibold text-sm text-[#030303] tracking-[-0.28px] whitespace-nowrap">
          Persönliches Angebot erhalten
        </span>
        <div class="w-[50px] h-[50px] rounded-full bg-white flex items-center justify-center shrink-0">
          <svg width="18" height="18" viewBox="0 0 18 18" fill="none">
            <path d="M15.242 4.24333L15.3683 4.3697C13.8066 5.93182 13.8063 8.46457 15.3683 10.0266L13.9541 11.4408C12.805 10.2917 12.2215 8.79178 12.1995 7.28582L4.6609 14.8244L3.24668 13.4102L10.6672 5.98969C9.24429 5.91119 7.84432 5.3296 6.75736 4.24264L8.17157 2.82843C9.73367 4.39052 12.2663 4.39052 13.8284 2.82843L15.242 4.24333Z" fill="#030303"/>
          </svg>
        </div>
      </a>
    </div>

  </div>
</div>
