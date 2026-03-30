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
    { key: 'winter', label: 'Winter' },
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
  let flowPaused   = false;

  // ── Hover highlight ────────────────────────────────────────────────────────
  type FlowGroup = 'solar' | 'einspeisung' | 'netz' | 'batterie' | 'eigenverbrauch' | 'gesamtverbrauch' | 'ev';
  let hoveredFlow: FlowGroup | null = null;

  // Reactive derived opacities — Svelte tracks hoveredFlow directly here
  // Lines
  $: oSolarHouse   = !hoveredFlow || hoveredFlow==='solar' || hoveredFlow==='eigenverbrauch' || hoveredFlow==='gesamtverbrauch' ? 1 : 0.2;
  $: oSolarBat     = !hoveredFlow || hoveredFlow==='solar' || hoveredFlow==='batterie'                                         ? 1 : 0.2;
  $: oEinspeisung  = !hoveredFlow || hoveredFlow==='solar' || hoveredFlow==='einspeisung'                                      ? 1 : 0.2;
  $: oNetz         = !hoveredFlow || hoveredFlow==='netz'  || hoveredFlow==='gesamtverbrauch'                                  ? 1 : 0.2;
  $: oBatHouse     = !hoveredFlow || hoveredFlow==='batterie' || hoveredFlow==='gesamtverbrauch'                               ? 1 : 0.2;
  $: oEV           = !hoveredFlow || hoveredFlow==='ev' || hoveredFlow==='gesamtverbrauch'                                     ? 1 : 0.2;
  // Boxes
  $: bSolar          = !hoveredFlow || hoveredFlow==='solar' || hoveredFlow==='einspeisung' || hoveredFlow==='batterie' || hoveredFlow==='eigenverbrauch' || hoveredFlow==='gesamtverbrauch' ? 1 : 0.25;
  $: bEinspeisung    = !hoveredFlow || hoveredFlow==='einspeisung' || hoveredFlow==='solar'                                                                                                   ? 1 : 0.25;
  $: bNetz           = !hoveredFlow || hoveredFlow==='netz' || hoveredFlow==='gesamtverbrauch' || hoveredFlow==='einspeisung' || hoveredFlow==='solar'                                        ? 1 : 0.25;
  $: bBatterie       = !hoveredFlow || hoveredFlow==='batterie' || hoveredFlow==='solar' || hoveredFlow==='gesamtverbrauch'                                                                   ? 1 : 0.25;
  $: bGesamtverbrauch= !hoveredFlow || hoveredFlow==='gesamtverbrauch' || hoveredFlow==='netz' || hoveredFlow==='batterie' || hoveredFlow==='eigenverbrauch' || hoveredFlow==='ev' || hoveredFlow==='solar' ? 1 : 0.25;
  $: bEigenverbrauch = !hoveredFlow || hoveredFlow==='eigenverbrauch' || hoveredFlow==='solar' || hoveredFlow==='gesamtverbrauch'                                                              ? 1 : 0.25;
  $: bEV             = !hoveredFlow || hoveredFlow==='ev' || hoveredFlow==='gesamtverbrauch'                                                                                                  ? 1 : 0.25;


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

  // ── Monthly savings chart ─────────────────────────────────────────────────
  const MONTHS = ['J','F','M','A','M','J','J','A','S','O','N','D'];
  const monthProdMult = [0.25,0.40,0.80,1.10,1.35,1.50,1.55,1.40,1.00,0.65,0.35,0.20];
  const monthConsMult = [1.45,1.35,1.10,0.90,0.75,0.65,0.60,0.65,0.80,1.00,1.25,1.50];
  const PRICE_KWH = 0.29;

  const FEED_IN_PRICE = 0.10; // ~10 Rp/kWh Einspeisevergütung CH
  $: monthlyData = MONTHS.map((label, i) => {
    const mCons   = (yearlyKwhEnergyConsumption / 12) * monthConsMult[i];
    const mProd   = (initialAcKwhPerYear / 12) * monthProdMult[i];
    const mDirect = Math.min(mProd, mCons) * (localBattery ? 0.80 : 0.70);
    const mGrid   = Math.max(0, mCons - mDirect);
    const mExport = Math.max(0, mProd - mDirect);
    const costWithout = mCons * PRICE_KWH;
    const costWith    = mGrid * PRICE_KWH - mExport * FEED_IN_PRICE;
    return { label, costWithout, costWith: Math.max(0, costWith), saving: costWithout - Math.max(0, costWith), gridWith: Math.round(mGrid), gridWithout: Math.round(mCons) };
  });
  $: maxMonthlyCost = Math.max(...monthlyData.map(d => d.costWithout), 1);
  $: maxMonthlySaving = Math.max(...monthlyData.map(d => d.saving), 1);
  $: avgMonthlySaving = monthlyData.reduce((s, d) => s + d.saving, 0) / 12;

  // ── Monthly chart dimensions ───────────────────────────────────────────────
  const MW=300, MH=150, MP={t:4,r:0,b:14,l:0};
  const mcH=MH-MP.t-MP.b;
  const mStep=MW/12, mBW=mStep*0.55;
  function mpx(i:number){return i*mStep+mStep/2-mBW/2;}
  $: mMaxCost = maxMonthlySaving;
  function mScaledH(v:number, max:number){return max>0?(v/max)*mcH:0;}
  function mScaledY(v:number, max:number){return MP.t+mcH-mScaledH(v,max);}
  let mHoverIdx: number | null = null;
  const MONTH_FULL = ['Januar','Februar','März','April','Mai','Juni','Juli','August','September','Oktober','November','Dezember'];

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
  const BW=520,BH=200,BP={t:16,r:8,b:32,l:36};
  const bcW=BW-BP.l-BP.r, bcH=BH-BP.t-BP.b;
  const bStep=bcW/24, bBarW=bStep*0.55;
  function bpx(h:number){return BP.l+h*bStep+bStep/2-bBarW/2;}
  function bph(v:number){return (v/maxHourly)*bcH;}
  function bpy(v:number){return BP.t+bcH-bph(v);}

  let bHoverHour: number | null = null;
  function handleBarMove(e: MouseEvent) {
    const svg = e.currentTarget as SVGSVGElement;
    const rect = svg.getBoundingClientRect();
    const svgX = (e.clientX - rect.left) * (BW / rect.width);
    const h = Math.round((svgX - BP.l - bStep/2) / bStep);
    bHoverHour = h >= 0 && h < 24 ? h : null;
  }

  // ── Line chart SVG ────────────────────────────────────────────────────────
  const LW=520,LH=200,LP={t:16,r:12,b:32,l:52};
  const lcW=LW-LP.l-LP.r, lcH=LH-LP.t-LP.b;
  $: lMax=Math.max(...cumWithoutSolar,1);
  $: lXS=lcW/Math.max(installationLifeSpan-1,1);
  function lpx(i:number){return LP.l+i*lXS;}
  function lpy(v:number){return LP.t+lcH-(v/lMax)*lcH;}

  // ── Line chart hover ─────────────────────────────────────────────────────
  let lHoverYear: number | null = null;
  $: lHoverX   = lHoverYear !== null ? lpx(lHoverYear) : -999;
  $: lHoverValS = lHoverYear !== null ? (cumWithSolar[lHoverYear] ?? 0) : 0;
  $: lHoverValN = lHoverYear !== null ? (cumWithoutSolar[lHoverYear] ?? 0) : 0;
  $: lHoverYS  = lHoverYear !== null ? lpy(lHoverValS) : -999;
  $: lHoverYN  = lHoverYear !== null ? lpy(lHoverValN) : -999;
  $: lHoverCalendarYear = lHoverYear !== null ? new Date().getFullYear() + lHoverYear : null;
  // Tooltip left or right of cursor depending on position
  $: lTipLeft  = lHoverYear !== null && lHoverX > LW * 0.5;
  $: lTipX     = lHoverYear !== null
    ? lTipLeft
      ? lHoverX - 210 - 8 - 28 * ((lHoverX - LW * 0.5) / (LW * 0.5))
      : lHoverX + 8 + 28 * (1 - lHoverX / (LW * 0.5))
    : -999;

  function handleLineMove(e: MouseEvent) {
    const svg = e.currentTarget as SVGSVGElement;
    const rect = svg.getBoundingClientRect();
    const svgX = (e.clientX - rect.left) * (LW / rect.width);
    const yearF = (svgX - LP.l) / lXS;
    lHoverYear = Math.max(0, Math.min(installationLifeSpan - 1, Math.round(yearF)));
  }
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
  // Gauge goes left (9 o'clock) → top (12 o'clock) → right (3 o'clock)
  // In our coord system (0°=top, clockwise): left=270°, right=90°, through top
  const DR=58, DRI=40, DCX=110, DCY=72;
  const DSW = DR - DRI; // stroke width
  function polar(cx:number,cy:number,r:number,deg:number){
    const rad=(deg-90)*Math.PI/180;
    return{x:cx+r*Math.cos(rad),y:cy+r*Math.sin(rad)};
  }
  function arc(cx:number,cy:number,r:number,s:number,e:number){
    const p1=polar(cx,cy,r,s),p2=polar(cx,cy,r,e);
    const large=(e-s+360)%360>180?1:0;
    return`M${p1.x.toFixed(2)},${p1.y.toFixed(2)} A${r},${r} 0 ${large} 1 ${p2.x.toFixed(2)},${p2.y.toFixed(2)}`;
  }
  // 270° = left endpoint, 270+180=450=90° = right endpoint
  $: donutEnd = 270 + Math.min(roofPct, 100) * 1.8;
  $: arcBg    = arc(DCX, DCY, DR, 270, 90);   // full background: left→right through top
  $: arcRoof  = arc(DCX, DCY, DR, 270, donutEnd);

  let donutHovered = false;

  // ── Energieunabhängigkeit 24h chart ──────────────────────────────────────
  // Solar bell curve centered on hour 13, consumption with morning+evening peaks
  function solarH(h: number): number {
    if (h < 6 || h > 20) return 0;
    return Math.max(0, Math.sin(((h - 6) / 14) * Math.PI));
  }
  function consH(h: number): number {
    const morning = Math.exp(-0.5 * ((h - 7.5) / 1.5) ** 2) * 0.7;
    const evening = Math.exp(-0.5 * ((h - 19.5) / 2.0) ** 2) * 1.0;
    const base = 0.15;
    return base + morning + evening;
  }
  $: indepHours = (() => {
    const dailySolar = initialAcKwhPerYear / 365;
    const dailyCons  = yearlyKwhEnergyConsumption / 365;
    let count = 0;
    for (let h = 0; h < 24; h++) {
      const prod = dailySolar * solarH(h) * 2.2;
      const batBonus = localBattery ? dailySolar * 0.3 * (h >= 17 && h <= 23 ? 0.18 : 0) : 0;
      const cons = dailyCons * consH(h) * 1.8;
      if (prod + batBonus >= cons * 0.85) count++;
    }
    return count;
  })();
  $: indepPct = localBattery ? Math.min(100, roofPct + 15) : roofPct;
  $: batCoverHours = localBattery ? Math.round(indepHours * 0.3) : 0;
  const EW = 280, EH = 110, EP = { t: 8, r: 4, b: 18, l: 4 };
  $: ecH = EH - EP.t - EP.b;
  $: hourlyBars = Array.from({ length: 24 }, (_, h) => {
    const dailySolar = initialAcKwhPerYear / 365;
    const dailyCons  = yearlyKwhEnergyConsumption / 365;
    const prod = dailySolar * solarH(h) * 2.2;
    const cons = dailyCons * consH(h) * 1.8;
    const batBonus = localBattery ? dailySolar * 0.3 * (h >= 17 && h <= 22 ? 0.5 : 0) : 0;
    const solarCovered = Math.min(cons, prod);
    const grid = Math.max(0, cons - solarCovered - batBonus);
    return { h, prod, cons, solarCovered, grid, batBonus };
  });
  $: maxBarVal = Math.max(...hourlyBars.map(b => Math.max(b.prod, b.grid)), 1);
  // Dashed line follows consumption curve (what needs to be covered)
  $: linePoints = hourlyBars.map((b, i) => {
    const x = EP.l + (i / 23) * (EW - EP.l - EP.r);
    const y = EP.t + ecH - (b.cons / maxBarVal) * ecH;
    return `${x.toFixed(1)},${y.toFixed(1)}`;
  }).join(' ');
  const barW = 8;
  function ex(h: number) { return EP.l + (h / 23) * (EW - EP.l - EP.r) - barW / 2; }
  let eHoverH: number | null = null;
  $: solarStartH  = hourlyBars.findIndex(b => b.prod > b.grid * 0.5);
  $: solarEndH    = 23 - [...hourlyBars].reverse().findIndex(b => b.prod > b.grid * 0.5);
  $: tlStartPct   = (solarStartH / 24) * 100;
  $: tlEndPct     = (solarEndH   / 24) * 100;
  $: tlSpanPct    = tlEndPct - tlStartPct;
</script>

<style>
  /*
   * Energy flow line animations
   * All paths are drawn in the direction of energy flow,
   * so a single "forward" animation works for all lines.
   */
  /* stroke-dasharray "3.5 2" → pattern length = 5.5 → use multiples: 27.5, 33, 55 … */
  /* stroke-dasharray "3 2"   → pattern length = 5   → use multiples: 25,   30, 50 … */
  @keyframes flow {
    from { stroke-dashoffset: 33; }
    to   { stroke-dashoffset: 0;  }
  }
  @keyframes flow-short {
    from { stroke-dashoffset: 30; }
    to   { stroke-dashoffset: 0;  }
  }
  .flow      { animation: flow       5s   linear infinite; will-change: stroke-dashoffset; }
  .flow-slow { animation: flow       7s   linear infinite; will-change: stroke-dashoffset; }
  .flow-fast { animation: flow       4s   linear infinite; will-change: stroke-dashoffset; }
  .flow-short{ animation: flow-short 5s   linear infinite; will-change: stroke-dashoffset; }
  :global(.paused) .flow, :global(.paused) .flow-slow,
  :global(.paused) .flow-fast, :global(.paused) .flow-short {
    animation-play-state: paused;
  }

  @keyframes tipFadeIn {
    from { opacity: 0; transform: translateY(4px); }
    to   { opacity: 1; transform: translateY(0);   }
  }
  .tip-enter {
    animation: tipFadeIn 0.18s ease forwards;
  }
  .tip-move {
    transition: transform 0.2s cubic-bezier(0.25, 0.46, 0.45, 0.94);
  }

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

          <!-- Pause / Play button -->
          <button
            on:click={() => flowPaused = !flowPaused}
            class="absolute flex items-center gap-2 px-3 py-1.5 rounded-full transition-all duration-200"
            style="left:1rem; top:1rem; background:rgba(3,3,3,0.18); backdrop-filter:blur(4px); border:1px solid rgba(255,255,255,0.06);">
            {#if flowPaused}
              <svg class="w-3.5 h-3.5 shrink-0" viewBox="0 0 24 24" fill="white">
                <path d="M8 5v14l11-7z"/>
              </svg>
              <span class="font-switzer text-[11px] font-semibold text-white whitespace-nowrap">Energie-Flow starten</span>
            {:else}
              <svg class="w-3.5 h-3.5 shrink-0" viewBox="0 0 24 24" fill="white">
                <path d="M6 19h4V5H6v14zm8-14v14h4V5h-4z"/>
              </svg>
              <span class="font-switzer text-[11px] font-semibold text-white whitespace-nowrap">Energie-Flow pausieren</span>
            {/if}
          </button>

          <!-- SVG flow overlay: coords map 1:1 to image percentage -->
          <svg class="absolute inset-0 w-full h-full pointer-events-none {flowPaused ? 'paused' : ''}"
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
                <path d="M0,0 L5,2.5 L0,5 Z" fill="#F97316" opacity="0.9"/>
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
            <path d="M 44 9 C 53 18, 55 25, 56 32"
              fill="none" stroke="#FFE572"
              stroke-width="0.7"
              stroke-dasharray="3.5 2" marker-end="url(#arrY)"
              opacity="{oSolarHouse}"
              class="flow" style="transition:opacity 0.5s"/>

            <!-- 2. Solar → Powerwall battery -->
            {#if localBattery}
              <path d="M 43 9 C 32 15, 28 20, 26 32"
                fill="none" stroke="#FFE572"
                stroke-width="0.7"
                stroke-dasharray="3.5 2" marker-end="url(#arrY)"
                opacity="{oSolarBat}"
                class="flow-slow" style="transition:opacity 0.5s"/>
            {/if}

            <!-- 3. Solar → Einspeisung (only when overproducing) -->
            {#if einspeisKwh > 100}
              <path d="M 42 9 C 34 9, 22 11, 12 15"
                fill="none" stroke="#0BDBB8"
                stroke-width="0.6"
                stroke-dasharray="3 2" marker-end="url(#arrT)"
                opacity="{oEinspeisung}"
                class="flow-short" style="transition:opacity 0.5s"/>
            {/if}

            <!-- 4. Netz → Wohnzimmer (only when underproducing) -->
            {#if gridImport > 100}
              <path d="M 10 17 C 20 20, 30 25, 49.5 34"
                fill="none" stroke="#66A3FF"
                stroke-width="0.7"
                stroke-dasharray="3.5 2" marker-end="url(#arrB)"
                opacity="{oNetz}"
                class="flow" style="transition:opacity 0.5s"/>
            {/if}

            <!-- 5. Powerwall → Wohnzimmer -->
            {#if localBattery}
              <path d="M 29 36 C 37 38, 45 38, 49.5 37"
                fill="none" stroke="#F97316"
                stroke-width="0.7"
                stroke-dasharray="3.5 2" marker-end="url(#arrG)"
                opacity="{oBatHouse}"
                class="flow-fast" style="transition:opacity 0.5s"/>
            {/if}

            <!-- 6. Wohnzimmer → EV Ladestation -->
            {#if evCharging}
              <path d="M 64 38 C 68 40, 69 40, 74.5 41"
                fill="none" stroke="#A78BFA"
                stroke-width="0.7"
                stroke-dasharray="3.5 2" marker-end="url(#arrP)"
                opacity="{oEV}"
                class="flow" style="transition:opacity 0.5s"/>
            {/if}

          </svg>

          <!-- ── Floating label cards ── -->
          <!-- Positioned using image percentages (left%, top%) -->

          <!-- SOLAR (über den Panels, oben-Mitte) -->
          <div class="absolute" role="region" aria-label="Solar" style="left:40%; top:11%; opacity:{bSolar}; transition:opacity 0.5s;"
            on:mouseenter={() => hoveredFlow='solar'} on:mouseleave={() => hoveredFlow=null}>
            <div class="rounded-[10px] px-3 py-2 shadow-lg cursor-default" style="background:rgba(3,3,3,0.84); backdrop-filter:blur(8px); min-width:125px;">
              <div class="flex items-center gap-1.5 mb-0.5">
                <div class="w-2 h-2 rounded-full bg-[#FFE572]" />
                <span class="font-switzer text-[10px] text-[#FFE572]">Solar</span>
              </div>
              <p class="font-switzer font-semibold text-[17px] text-white leading-none">{fmt(solarKwPeak,1)} kW</p>
              <p class="font-switzer text-[10px] text-[#8C8C8C] mt-0.5">{fmt(sProd)} kWh / Jahr</p>
            </div>
          </div>

          <!-- EINSPEISUNG (links oben, nur wenn Überschuss) — helles KPI-Design -->
          {#if einspeisKwh > 100}
            <div class="absolute" role="region" aria-label="Netz-Einspeisung" style="left:20%; top:13%; opacity:{bEinspeisung}; transition:opacity 0.5s;"
              on:mouseenter={() => hoveredFlow='einspeisung'} on:mouseleave={() => hoveredFlow=null}>
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
          <div class="absolute" role="region" aria-label="Netz-Bezug" style="left:2%; top:25%; opacity:{bNetz}; transition:opacity 0.5s;"
            on:mouseenter={() => hoveredFlow='netz'} on:mouseleave={() => hoveredFlow=null}>
            <div class="rounded-[10px] px-3 py-2 shadow-lg cursor-default" style="background:rgba(3,3,3,0.84); backdrop-filter:blur(8px); min-width:110px;">
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
                {gridImport > 100 ? fmt(gridImport)+' kWh / Jahr' : 'Kein Bezug'}
              </p>
            </div>
          </div>

          <!-- BATTERIE (Powerwall, links an der Hauswand) -->
          {#if localBattery}
            <div class="absolute" role="region" aria-label="Batterie" style="left:17%; top:60%; opacity:{bBatterie}; transition:opacity 0.5s;"
              on:mouseenter={() => hoveredFlow='batterie'} on:mouseleave={() => hoveredFlow=null}>
              <div class="rounded-[10px] px-3 py-2 shadow-lg" style="background:rgba(3,3,3,0.84); backdrop-filter:blur(8px); min-width:130px;">
                <div class="flex items-center gap-1.5 mb-0.5">
                  <div class="w-2 h-2 rounded-full bg-[#F97316]" />
                  <span class="font-switzer text-[10px] text-[#F97316]">Batterie</span>
                </div>
                <div class="flex items-baseline gap-2">
                  <p class="font-switzer font-semibold text-[17px] text-white leading-none">{fmt(batteryKwh)} kWh</p>
                  <span class="font-switzer text-[12px] text-[#F97316]">· {battPct}%</span>
                </div>
                <p class="font-switzer text-[10px] text-[#8C8C8C] mt-0.5">Speicher aktiv</p>
              </div>
            </div>
          {/if}

          <!-- GESAMTVERBRAUCH — am Konvergenzpunkt (Wohnzimmer/TV) — dunkles Design -->
          <div class="absolute" role="region" aria-label="Gesamtverbrauch" style="left:50%; top:60%; opacity:{bGesamtverbrauch}; transition:opacity 0.5s;"
            on:mouseenter={() => hoveredFlow='gesamtverbrauch'} on:mouseleave={() => hoveredFlow=null}>
            <div class="rounded-[10px] px-3 py-2 shadow-lg cursor-default" style="background:rgba(3,3,3,0.84); backdrop-filter:blur(8px); min-width:150px;">
              <div class="flex items-center gap-1.5 mb-0.5">
                <div class="w-2 h-2 rounded-full bg-white" />
                <span class="font-switzer text-[10px] text-[#aaa]">Gesamtverbrauch</span>
              </div>
              <p class="font-switzer font-semibold text-[17px] text-white leading-none">{fmt(sCons)} kWh / Jahr</p>
              <p class="font-switzer text-[10px] text-[#8C8C8C] mt-0.5">Solar-Anteil: {fmt(solarUsed)} kWh</p>
            </div>
          </div>

          <!-- EIGENVERBRAUCH — direkt vom Dach — farbiges helles Design -->
          <div class="absolute" role="region" aria-label="Eigenverbrauch" style="left:48%; top:33%; opacity:{bEigenverbrauch}; transition:opacity 0.5s;"
            on:mouseenter={() => hoveredFlow='eigenverbrauch'} on:mouseleave={() => hoveredFlow=null}>
            <div class="rounded-[10px] px-3 py-2 shadow-lg cursor-default" style="background:rgba(255,252,230,0.97); border:1px solid rgba(255,229,114,0.5); backdrop-filter:blur(8px); min-width:135px;">
              <div class="flex items-center gap-1.5 mb-0.5">
                <div class="w-2 h-2 rounded-full bg-[#F59E0B]" />
                <span class="font-switzer text-[10px] text-[#92400E]">Eigenverbrauch Solar</span>
              </div>
              <p class="font-switzer font-semibold text-[17px] text-[#1C1917] leading-none">{fmt(directKwh)} kWh</p>
              <p class="font-switzer text-[10px] text-[#78716C] mt-0.5">direkt vom Dach</p>
            </div>
          </div>

          <!-- EV LADESTATION (rechts, beim Garageneingang) -->
          {#if evCharging}
            <div class="absolute" role="region" aria-label="Ladestation" style="left:75%; top:70%; opacity:{bEV}; transition:opacity 0.5s;"
              on:mouseenter={() => hoveredFlow='ev'} on:mouseleave={() => hoveredFlow=null}>
              <div class="rounded-[10px] px-3 py-2 shadow-lg" style="background:rgba(3,3,3,0.84); backdrop-filter:blur(8px); min-width:130px;">
                <div class="flex items-center gap-1.5 mb-0.5">
                  <div class="w-2 h-2 rounded-full bg-[#A78BFA]" />
                  <span class="font-switzer text-[10px] text-[#A78BFA]">Ladestation</span>
                </div>
                <p class="font-switzer font-semibold text-[17px] text-white leading-none">{fmt(evKwh)} kWh</p>
                <p class="font-switzer text-[10px] text-[#8C8C8C] mt-0.5">20'000 km / Jahr</p>
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
              <svg class="w-[13px] h-[13px] shrink-0" viewBox="0 0 24 24" fill="#F97316">
                <path d="M15.67 4H14V2h-4v2H8.33C7.6 4 7 4.6 7 5.33v15.33C7 21.4 7.6 22 8.33 22h7.33c.74 0 1.34-.6 1.34-1.33V5.33C17 4.6 16.4 4 15.67 4zm-1.67 9h-2v4h-2v-4H8l4-8 4 8h-2z"/>
              </svg>
              <span class="font-switzer font-semibold text-[12px] text-[#030303]">Batterie</span>
            </div>
            <button on:click={() => localBattery=!localBattery}
              class="relative w-[36px] h-[20px] rounded-full shrink-0 transition-colors duration-200"
              style="background:{localBattery?'#F97316':'#CECFCF'}">
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

    </div>

    <!-- ══ KPI Cards Row ══ -->
    <div class="grid grid-cols-4 gap-3">

      <!-- Einsparung — Hero -->
      <div class="rounded-[16px] px-6 py-5 flex flex-col gap-4" style="background:#030303;">
        <div class="flex items-center gap-2.5">
          <div class="w-9 h-9 rounded-[10px] flex items-center justify-center shrink-0" style="background:rgba(2,181,151,0.18);">
            <svg class="w-5 h-5" viewBox="0 0 24 24" fill="#02B597">
              <path d="M3.5 18.49l6-6.01 4 4L22 6.92l-1.41-1.41-7.09 7.97-4-4L2 16.99z"/>
            </svg>
          </div>
          <div>
            <p class="font-switzer font-semibold text-[17px] text-white leading-none">Einsparung</p>
            <p class="font-switzer text-[12px] text-[#666] mt-0.5">über {installationLifeSpan} Jahre</p>
          </div>
        </div>
        <p class="font-switzer font-semibold text-[36px] text-white leading-none tracking-[-1px]">
          CHF {fmt(savings)}
        </p>
      </div>

      <!-- Solaranlage -->
      <div class="rounded-[16px] px-6 py-5 flex flex-col gap-4" style="background:white; border:1px solid #e8e3db;">
        <div class="flex items-center gap-2.5">
          <div class="w-9 h-9 rounded-[10px] flex items-center justify-center shrink-0" style="background:rgba(255,229,114,0.25);">
            <svg class="w-5 h-5" viewBox="0 0 24 24" fill="#D97706">
              <path d="M12 7c-2.76 0-5 2.24-5 5s2.24 5 5 5 5-2.24 5-5-2.24-5-5-5zM2 13h2c.55 0 1-.45 1-1s-.45-1-1-1H2c-.55 0-1 .45-1 1s.45 1 1 1zm18 0h2c.55 0 1-.45 1-1s-.45-1-1-1h-2c-.55 0-1 .45-1 1s.45 1 1 1zM11 2v2c0 .55.45 1 1 1s1-.45 1-1V2c0-.55-.45-1-1-1s-1 .45-1 1zm0 18v2c0 .55.45 1 1 1s1-.45 1-1v-2c0-.55-.45-1-1-1s-1 .45-1 1z"/>
            </svg>
          </div>
          <div>
            <p class="font-switzer font-semibold text-[17px] text-[#030303] leading-none">Solaranlage</p>
            <p class="font-switzer text-[12px] text-[#8C8C8C] mt-0.5">{fmt(initialAcKwhPerYear)} kWh Jahresproduktion</p>
          </div>
        </div>
        <p class="font-switzer font-semibold text-[36px] text-[#030303] leading-none tracking-[-1px]">
          {fmt(installationSizeKw,1)} <span class="text-[20px] font-normal text-[#8C8C8C]">kWp</span>
        </p>
      </div>

      <!-- Amortisationszeit -->
      <div class="rounded-[16px] px-6 py-5 flex flex-col gap-4" style="background:white; border:1px solid #e8e3db;">
        <div class="flex items-center gap-2.5">
          <div class="w-9 h-9 rounded-[10px] flex items-center justify-center shrink-0" style="background:#f2f2f2;">
            <svg class="w-5 h-5" viewBox="0 0 24 24" fill="#555">
              <path d="M12 2C6.48 2 2 6.48 2 12s4.48 10 10 10 10-4.48 10-10S17.52 2 12 2zm1 5h-2v6l5.25 3.15.75-1.23-4-2.42V7z"/>
            </svg>
          </div>
          <div>
            <p class="font-switzer font-semibold text-[17px] text-[#030303] leading-none">Amortisationszeit</p>
            <p class="font-switzer text-[12px] text-[#8C8C8C] mt-0.5">bis die Anlage sich trägt</p>
          </div>
        </div>
        <p class="font-switzer font-semibold text-[36px] text-[#030303] leading-none tracking-[-1px]">
          {breakEvenYear>=0?breakEvenYear+1:'—'} <span class="text-[20px] font-normal text-[#8C8C8C]">Jahre</span>
        </p>
      </div>

      <!-- Investition -->
      <div class="rounded-[16px] px-6 py-5 flex flex-col gap-4" style="background:white; border:1px solid #e8e3db;">
        <div class="flex items-center gap-2.5">
          <div class="w-9 h-9 rounded-[10px] flex items-center justify-center shrink-0" style="background:#f5f5f5;">
            <svg class="w-5 h-5" viewBox="0 0 24 24" fill="#555">
              <path d="M10 20v-6h4v6h5v-8h3L12 3 2 12h3v8z"/>
            </svg>
          </div>
          <div>
            <p class="font-switzer font-semibold text-[17px] text-[#030303] leading-none">Investition</p>
            <div class="flex items-center gap-1 mt-0.5">
              <svg class="w-3 h-3 shrink-0" viewBox="0 0 24 24" fill="#02B597">
                <path d="M12 1L3 5v6c0 5.55 3.84 10.74 9 12 5.16-1.26 9-6.45 9-12V5l-9-4zm-2 16l-4-4 1.41-1.41L10 14.17l6.59-6.59L18 9l-8 8z"/>
              </svg>
              <p class="font-switzer text-[11px] text-[#02B597]">EIV CHF {fmt(solarIncentives)}</p>
            </div>
          </div>
        </div>
        <p class="font-switzer font-semibold text-[36px] text-[#030303] leading-none tracking-[-1px]">
          CHF {fmt(installationCostTotal-solarIncentives)}
        </p>
      </div>

    </div>

    <!-- ══ Charts Row 1: Bar + Line ══ -->
    <div class="grid grid-cols-2 gap-4">

      <!-- ── Bar chart: Tagesproduktion ── -->
      <div class="rounded-[16px] p-6 flex flex-col gap-5" style="background:white; border:1px solid #ede8e0;">
        <!-- Header -->
        <div class="flex items-start justify-between">
          <div>
            <h3 class="font-switzer font-semibold text-[17px] text-[#030303] tracking-[-0.4px]">Tagesproduktion</h3>
            <p class="font-switzer text-[13px] text-[#8C8C8C] mt-0.5">Typischer Tagesverlauf · {activeSeason === 'year' ? 'Jahresdurchschnitt' : seasons.find(s=>s.key===activeSeason)?.label}</p>
          </div>
          <div class="flex items-center gap-4">
            {#if localBattery}
              <div class="flex items-center gap-1.5">
                <div class="w-3 h-[3px] rounded-full" style="background:#FFE572"/>
                <span class="font-switzer text-[12px] text-[#8C8C8C]">Batterie</span>
              </div>
            {/if}
            <div class="flex items-center gap-1.5">
              <div class="w-3 h-3 rounded-sm" style="background:#0BDBB8; opacity:0.8"/>
              <span class="font-switzer text-[12px] text-[#8C8C8C]">Produktion</span>
            </div>
            <div class="flex items-center gap-1.5">
              <div class="w-3 h-3 rounded-sm" style="background:#66A3FF; opacity:0.4"/>
              <span class="font-switzer text-[12px] text-[#8C8C8C]">Verbrauch</span>
            </div>
          </div>
        </div>
        <!-- Chart -->
        <svg viewBox="0 0 {BW} {BH}" class="w-full" style="overflow:visible; cursor:crosshair;"
          on:mousemove={handleBarMove} on:mouseleave={() => bHoverHour=null}>
          <defs>
            <linearGradient id="gradProd" x1="0" y1="0" x2="0" y2="1">
              <stop offset="0%" stop-color="#0BDBB8" stop-opacity="1"/>
              <stop offset="100%" stop-color="#0BDBB8" stop-opacity="0.5"/>
            </linearGradient>
            <linearGradient id="gradCons" x1="0" y1="0" x2="0" y2="1">
              <stop offset="0%" stop-color="#66A3FF" stop-opacity="0.5"/>
              <stop offset="100%" stop-color="#66A3FF" stop-opacity="0.15"/>
            </linearGradient>
          </defs>
          <!-- Grid -->
          {#each [0.25,0.5,0.75,1] as f}
            <line x1={BP.l} y1={BP.t+bcH*(1-f)} x2={BW-BP.r} y2={BP.t+bcH*(1-f)}
              stroke="#f4f4f4" stroke-width="1"/>
            <text x={BP.l-5} y={BP.t+bcH*(1-f)+4} text-anchor="end" font-size="9" fill="#bbb" font-family="Switzer,sans-serif">
              {(maxHourly*f).toFixed(1)}
            </text>
          {/each}
          <!-- Bars -->
          {#each hourlyData as d}
            <rect x={bpx(d.h)} y={bpy(d.cons)} width={bBarW} height={bph(d.cons)}
              fill="url(#gradCons)" rx="3"
              opacity="{bHoverHour===null||bHoverHour===d.h?1:0.35}"/>
          {/each}
          {#each hourlyData as d}
            <rect x={bpx(d.h)} y={bpy(d.prod)} width={bBarW} height={bph(d.prod)}
              fill="url(#gradProd)" rx="3"
              opacity="{bHoverHour===null||bHoverHour===d.h?1:0.35}"/>
          {/each}
          {#if localBattery}
            <polyline
              points={hourlyData.map(d=>`${(bpx(d.h)+bBarW/2).toFixed(1)},${(BP.t+bcH-(d.bat/100)*bcH).toFixed(1)}`).join(' ')}
              fill="none" stroke="#FFE572" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round" opacity="0.9"/>
          {/if}
          <!-- X axis -->
          <line x1={BP.l} y1={BP.t+bcH} x2={BW-BP.r} y2={BP.t+bcH} stroke="#eeeeee" stroke-width="1"/>
          {#each [0,4,8,12,16,20,23] as h}
            <text x={bpx(h)+bBarW/2} y={BH-8} text-anchor="middle" font-size="9" fill="#bbb" font-family="Switzer,sans-serif">{h}h</text>
          {/each}
          <!-- Hover tooltip -->
          {#if bHoverHour !== null}
            {@const hd = hourlyData[bHoverHour]}
            {@const tx = bpx(bHoverHour)+bBarW/2}
            {@const tipLeft = tx > BW*0.65}
            {@const tipX = tipLeft ? tx-188 : tx+14}
            {@const tipW = 176}
            {@const tipH = localBattery ? 110 : 88}
            <line x1={tx} y1={BP.t} x2={tx} y2={BP.t+bcH} stroke="#030303" stroke-width="1" stroke-dasharray="4 3" opacity="0.15"/>
            <filter id="tipShadowB" x="-30%" y="-30%" width="160%" height="160%">
              <feDropShadow dx="0" dy="6" stdDeviation="10" flood-color="#000" flood-opacity="0.28"/>
            </filter>
            <g class="tip-enter">
              <!-- Tooltip box -->
              <rect x={tipX} y={BP.t-4} width={tipW} height={tipH} rx="14" fill="#0e0e0e" filter="url(#tipShadowB)"/>
              <rect x={tipX} y={BP.t-4} width={tipW} height={tipH} rx="14" fill="none" stroke="rgba(255,255,255,0.09)" stroke-width="1"/>
              <!-- Time header -->
              <text x={tipX+16} y={BP.t+16} font-size="13" font-weight="700" fill="white" letter-spacing="-0.3" font-family="Switzer,sans-serif">{bHoverHour}:00 Uhr</text>
              <!-- Divider -->
              <line x1={tipX+16} y1={BP.t+25} x2={tipX+tipW-16} y2={BP.t+25} stroke="#252525" stroke-width="1"/>
              <!-- Produktion row -->
              <circle cx={tipX+22} cy={BP.t+40} r="4" fill="#0BDBB8"/>
              <text x={tipX+32} y={BP.t+44} font-size="12" fill="#666" font-family="Switzer,sans-serif">Produktion</text>
              <text x={tipX+tipW-16} y={BP.t+44} text-anchor="end" font-size="13" font-weight="700" fill="white" font-family="Switzer,sans-serif">{hd.prod.toFixed(2)} kW</text>
              <!-- Verbrauch row -->
              <circle cx={tipX+22} cy={BP.t+62} r="4" fill="#66A3FF"/>
              <text x={tipX+32} y={BP.t+66} font-size="12" fill="#666" font-family="Switzer,sans-serif">Verbrauch</text>
              <text x={tipX+tipW-16} y={BP.t+66} text-anchor="end" font-size="13" font-weight="700" fill="white" font-family="Switzer,sans-serif">{hd.cons.toFixed(2)} kW</text>
              <!-- Batterie row -->
              {#if localBattery}
                <circle cx={tipX+22} cy={BP.t+84} r="4" fill="#FFE572"/>
                <text x={tipX+32} y={BP.t+88} font-size="12" fill="#666" font-family="Switzer,sans-serif">Batterie</text>
                <text x={tipX+tipW-16} y={BP.t+88} text-anchor="end" font-size="13" font-weight="700" fill="white" font-family="Switzer,sans-serif">{hd.bat.toFixed(0)}%</text>
              {/if}
            </g>
          {/if}
        </svg>
      </div>

      <!-- ── Line chart: Kostenvergleich ── -->
      <div class="rounded-[16px] p-6 flex flex-col gap-5" style="background:white; border:1px solid #ede8e0;">
        <!-- Header -->
        <div class="flex items-start justify-between">
          <div>
            <h3 class="font-switzer font-semibold text-[17px] text-[#030303] tracking-[-0.4px]">Kostenvergleich</h3>
            <p class="font-switzer text-[13px] text-[#8C8C8C] mt-0.5">Kumulierte Stromkosten über {installationLifeSpan} Jahre</p>
          </div>
          <div class="flex items-center gap-4">
            <div class="flex items-center gap-1.5">
              <div class="w-5 h-[2.5px] rounded-full" style="background:#FFE572"/>
              <span class="font-switzer text-[12px] text-[#8C8C8C]">Mit Solar</span>
            </div>
            <div class="flex items-center gap-1.5">
              <div class="w-5 h-[2.5px] rounded-full" style="background:#66A3FF"/>
              <span class="font-switzer text-[12px] text-[#8C8C8C]">Ohne Solar</span>
            </div>
          </div>
        </div>
        <!-- Chart -->
        <svg viewBox="0 0 {LW} {LH}" class="w-full" style="overflow:visible; cursor:crosshair;"
          on:mousemove={handleLineMove} on:mouseleave={() => lHoverYear=null}>
          <defs>
            <linearGradient id="gradSolar" x1="0" y1="0" x2="0" y2="1">
              <stop offset="0%" stop-color="#FFE572" stop-opacity="0.3"/>
              <stop offset="100%" stop-color="#FFE572" stop-opacity="0.02"/>
            </linearGradient>
            <linearGradient id="gradNoSolar" x1="0" y1="0" x2="0" y2="1">
              <stop offset="0%" stop-color="#66A3FF" stop-opacity="0.15"/>
              <stop offset="100%" stop-color="#66A3FF" stop-opacity="0.02"/>
            </linearGradient>
          </defs>
          <!-- Grid -->
          {#each lYLab as {val,y}}
            <line x1={LP.l} y1={y} x2={LW-LP.r} y2={y} stroke="#f4f4f4" stroke-width="1"/>
            <text x={LP.l-5} y={y+4} text-anchor="end" font-size="9" fill="#bbb" font-family="Switzer,sans-serif">
              {fmt(val)}
            </text>
          {/each}
          <!-- Fills -->
          <path d={lFillN} fill="url(#gradNoSolar)"/>
          <path d={lFillS} fill="url(#gradSolar)"/>
          <!-- Lines -->
          <path d={lPathN} fill="none" stroke="#66A3FF" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" opacity="0.7"/>
          <path d={lPathS} fill="none" stroke="#FFE572" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round"/>
          <!-- Amortisation marker -->
          {#if breakEvenYear>=0}
            {@const beCalYear = new Date().getFullYear() + breakEvenYear}
            {@const labelW = 88}
            {@const labelX = lBEX - labelW/2}
            {@const labelY = LP.t}
            <!-- Vertical dashed line full height -->
            <line x1={lBEX} y1={LP.t} x2={lBEX} y2={LP.t+lcH} stroke="#030303" stroke-width="1" stroke-dasharray="4 4" opacity="0.15"/>
            <!-- Dot always visible -->
            <circle cx={lBEX} cy={lBEY} r="6" fill="white" stroke="#FFE572" stroke-width="2.5"/>
            <circle cx={lBEX} cy={lBEY} r="2.5" fill="#FFE572"/>
            <!-- Year label — fades out while hovering -->
            <g style="opacity:{lHoverYear===null?1:0}; transition:opacity 0.25s ease;">
              <rect x={labelX} y={labelY} width={labelW} height="32" rx="16" fill="#030303"/>
              <text x={lBEX} y={labelY+21} text-anchor="middle" font-size="16" font-weight="700" fill="#FFE572" letter-spacing="-0.5" font-family="Switzer,sans-serif">{beCalYear}</text>
            </g>
          {/if}
          <!-- X axis labels -->
          <line x1={LP.l} y1={LP.t+lcH} x2={LW-LP.r} y2={LP.t+lcH} stroke="#eeeeee" stroke-width="1"/>
          {#each lXLab as {year,x}}
            <text x={x} y={LH-8} text-anchor="middle" font-size="9" fill="#bbb" font-family="Switzer,sans-serif">{year}</text>
          {/each}
          <!-- Hover crosshair + tooltip -->
          {#if lHoverYear !== null}
            {@const diff = lHoverValN - lHoverValS}
            {@const tipW = 210}
            {@const tipH = diff > 0 ? 122 : 96}
            <line x1={lHoverX} y1={LP.t} x2={lHoverX} y2={LP.t+lcH} stroke="#030303" stroke-width="1" stroke-dasharray="4 3" opacity="0.15"/>
            <circle cx={lHoverX} cy={lHoverYN} r="5" fill="white" stroke="#66A3FF" stroke-width="2.5"/>
            <circle cx={lHoverX} cy={lHoverYS} r="5" fill="white" stroke="#FFE572" stroke-width="2.5"/>
            <filter id="tipShadowL" x="-30%" y="-30%" width="160%" height="160%">
              <feDropShadow dx="0" dy="6" stdDeviation="10" flood-color="#000" flood-opacity="0.28"/>
            </filter>
            <g class="tip-move" style="transform:translate({lTipX}px, {LP.t-4}px)">
              <g class="tip-enter">
                <!-- Tooltip box -->
                <rect x="0" y="0" width={tipW} height={tipH} rx="14" fill="#0e0e0e" filter="url(#tipShadowL)"/>
                <rect x="0" y="0" width={tipW} height={tipH} rx="14" fill="none" stroke="rgba(255,255,255,0.09)" stroke-width="1"/>
                <!-- Header -->
                <text x="16" y="20" font-size="14" font-weight="700" fill="white" letter-spacing="-0.4" font-family="Switzer,sans-serif">{lHoverCalendarYear}</text>
                <text x={tipW-16} y="20" text-anchor="end" font-size="11" fill="#666" font-family="Switzer,sans-serif">Jahr {lHoverYear}</text>
                <!-- Divider -->
                <line x1="16" y1="29" x2={tipW-16} y2="29" stroke="#252525" stroke-width="1"/>
                <!-- Mit Solar -->
                <circle cx="24" cy="44" r="4" fill="#FFE572"/>
                <text x="34" y="48" font-size="12" fill="#666" font-family="Switzer,sans-serif">Mit Solar</text>
                <text x={tipW-16} y="48" text-anchor="end" font-size="13" font-weight="700" fill="white" font-family="Switzer,sans-serif">CHF {fmt(lHoverValS)}</text>
                <!-- Ohne Solar -->
                <circle cx="24" cy="66" r="4" fill="#66A3FF"/>
                <text x="34" y="70" font-size="12" fill="#666" font-family="Switzer,sans-serif">Ohne Solar</text>
                <text x={tipW-16} y="70" text-anchor="end" font-size="13" font-weight="700" fill="white" font-family="Switzer,sans-serif">CHF {fmt(lHoverValN)}</text>
                <!-- Ersparnis pill -->
                {#if diff > 0}
                  <rect x="16" y="82" width={tipW-32} height="26" rx="8" fill="rgba(2,181,151,0.12)"/>
                  <text x={tipW/2} y="99" text-anchor="middle" font-size="12" font-weight="700" fill="#02B597" font-family="Switzer,sans-serif">↓ CHF {fmt(diff)} gespart</text>
                {/if}
              </g>
            </g>
          {/if}
        </svg>
      </div>

    </div>

    <!-- ══ Charts Row 2: 3 Cards ══ -->
    <div class="grid grid-cols-3 gap-4">

      <!-- ── Monatliche Einsparung: Mini bar chart ── -->
      <div class="rounded-[16px] p-5 flex flex-col" style="background:white; border:1px solid #ede8e0; gap:12px;">
        <!-- Header -->
        <div class="shrink-0">
          <p class="font-switzer font-semibold text-[17px] text-[#030303] leading-none">Monatliche Einsparung</p>
          <p class="font-switzer text-[12px] text-[#8C8C8C] mt-0.5">weniger ans Netz im Ø</p>
        </div>

        <!-- Chart -->
        <svg viewBox="0 0 {MW} {MH}" class="w-full flex-1" style="overflow:visible; cursor:crosshair;"
          on:mousemove={(e) => { const r=e.currentTarget.getBoundingClientRect(); const x=(e.clientX-r.left)*(MW/r.width); mHoverIdx=Math.max(0,Math.min(11,Math.floor(x/mStep))); }}
          on:mouseleave={() => mHoverIdx=null}>
          <defs>
            <linearGradient id="gradMS" x1="0" y1="0" x2="0" y2="1">
              <stop offset="0%" stop-color="#02B597" stop-opacity="1"/>
              <stop offset="100%" stop-color="#02B597" stop-opacity="0.45"/>
            </linearGradient>
            <filter id="mTipShadow" x="-30%" y="-30%" width="160%" height="160%">
              <feDropShadow dx="0" dy="4" stdDeviation="8" flood-color="#000" flood-opacity="0.22"/>
            </filter>
          </defs>

          {#each monthlyData as d, i}
            {@const active = mHoverIdx === null || mHoverIdx === i}
            <!-- Saving bar -->
            <rect x={mpx(i)} y={mScaledY(d.saving, mMaxCost)} width={mBW} height={mScaledH(d.saving, mMaxCost)}
              fill="url(#gradMS)" rx="4"
              opacity={active ? 1 : 0.3}
              style="transition:opacity 0.2s;"/>
            <!-- Month label -->
            <text x={mpx(i)+mBW/2} y={MH-1} text-anchor="middle" font-size="8"
              fill={active ? '#999' : '#ddd'} font-family="Switzer,sans-serif"
              style="transition:fill 0.2s;">{d.label}</text>
          {/each}

          <!-- Hover tooltip -->
          {#if mHoverIdx !== null}
            {@const d = monthlyData[mHoverIdx]}
            {@const tx = mpx(mHoverIdx)+mBW/2}
            {@const tipW = 260}
            {@const tipH = 122}
            {@const tipX = Math.min(Math.max(tx - tipW/2, 0), MW - tipW)}
            {@const tipY = -tipH + 4 + mScaledY(d.saving, mMaxCost) * 0.15}
            {@const pctSaved = d.gridWithout > 0 ? Math.round((d.gridWithout - d.gridWith) / d.gridWithout * 100) : 0}
            <g class="tip-enter">
              <rect x={tipX} y={tipY} width={tipW} height={tipH} rx="14" fill="#0e0e0e" filter="url(#mTipShadow)"/>
              <rect x={tipX} y={tipY} width={tipW} height={tipH} rx="14" fill="none" stroke="rgba(255,255,255,0.09)" stroke-width="1"/>
              <!-- Header -->
              <text x={tipX+16} y={tipY+20} font-size="14" font-weight="700" fill="white" letter-spacing="-0.4" font-family="Switzer,sans-serif">{MONTH_FULL[mHoverIdx]}</text>
              <text x={tipX+tipW-16} y={tipY+20} text-anchor="end" font-size="11" fill="#666" font-family="Switzer,sans-serif">Strombezug aus dem Netz</text>
              <line x1={tipX+16} y1={tipY+29} x2={tipX+tipW-16} y2={tipY+29} stroke="#252525" stroke-width="1"/>
              <!-- Mit Solar -->
              <circle cx={tipX+24} cy={tipY+44} r="4" fill="#02B597"/>
              <text x={tipX+34} y={tipY+48} font-size="12" fill="#666" font-family="Switzer,sans-serif">Mit Solar</text>
              <text x={tipX+tipW-16} y={tipY+48} text-anchor="end" font-size="13" font-weight="700" fill="white" font-family="Switzer,sans-serif">{fmt(d.gridWith)} kWh</text>
              <!-- Ohne Solar -->
              <circle cx={tipX+24} cy={tipY+66} r="4" fill="#555"/>
              <text x={tipX+34} y={tipY+70} font-size="12" fill="#666" font-family="Switzer,sans-serif">Ohne Solar</text>
              <text x={tipX+tipW-16} y={tipY+70} text-anchor="end" font-size="13" font-weight="700" fill="white" font-family="Switzer,sans-serif">{fmt(d.gridWithout)} kWh</text>
              <!-- Pill -->
              <rect x={tipX+16} y={tipY+82} width={tipW-32} height="26" rx="8" fill="rgba(2,181,151,0.12)"/>
              <text x={tipX+tipW/2} y={tipY+99} text-anchor="middle" font-size="12" font-weight="700" fill="#02B597" font-family="Switzer,sans-serif">↓ {pctSaved}% weniger vom Netz</text>
            </g>
          {/if}
        </svg>

        <!-- Bottom: value + savings pill -->
        <div class="flex items-end justify-between pt-1 border-t border-[#f0ece6]">
          <div>
            <p class="font-switzer font-semibold text-[34px] text-[#030303] leading-none tracking-[-1.5px]">
              CHF {fmt(mHoverIdx !== null ? monthlyData[mHoverIdx].saving : avgMonthlySaving)}
            </p>
            <p class="font-switzer text-[11px] text-[#8C8C8C] mt-1">
              {mHoverIdx !== null ? MONTH_FULL[mHoverIdx] : 'Ø pro Monat'}
            </p>
          </div>
          <div class="px-3 py-1.5 rounded-full mb-1" style="background:rgba(2,181,151,0.1);">
            <p class="font-switzer text-[12px] font-semibold text-[#02B597]">↓ gespart</p>
          </div>
        </div>
      </div>

      <!-- ── Eigenverbrauch + Energieunabhängigkeit (combined) ── -->
      <div class="rounded-[16px] p-5 flex flex-col gap-4" style="background:white; border:1px solid #ede8e0;">
        <!-- Header -->
        <div>
          <p class="font-switzer font-semibold text-[17px] text-[#030303] leading-none">Eigenverbrauch & Unabhängigkeit</p>
          <p class="font-switzer text-[12px] text-[#8C8C8C] mt-0.5">Ihr Strom vom Dach — wieviel & wann</p>
        </div>

        <!-- Two-column: donut left, chart right -->
        <div class="flex gap-4 items-start">
          <!-- Left: donut gauge -->
          <div class="flex flex-col items-center gap-3 shrink-0" style="width:140px;">
            <svg viewBox="0 0 220 76" class="w-full" style="overflow:visible; cursor:default;"
              on:mouseenter={() => donutHovered = true}
              on:mouseleave={() => donutHovered = false}>
              <defs>
                <filter id="donutGlow" x="-20%" y="-20%" width="140%" height="140%">
                  <feDropShadow dx="0" dy="0" stdDeviation="4" flood-color="#FFE572" flood-opacity="0.5"/>
                </filter>
              </defs>
              <path d={arcBg} fill="none" stroke="#F0EDEA" stroke-width={DSW} stroke-linecap="round"/>
              {#if roofPct > 0}
                <path d={arcRoof} fill="none" stroke="#FFE572" stroke-width={DSW} stroke-linecap="round"
                  filter={donutHovered ? 'url(#donutGlow)' : undefined}
                  style="transition:filter 0.25s ease;"/>
              {/if}
              <text x={DCX} y={DCY - 12} text-anchor="middle" font-size="9" fill="#aaa" font-family="Switzer,sans-serif" letter-spacing="1">VOM DACH</text>
              <text x={DCX} y={DCY + 16} text-anchor="middle" font-size="32" font-weight="700" fill="#030303" font-family="Switzer,sans-serif" letter-spacing="-1">{roofPct}%</text>
            </svg>
            <!-- kWh stats -->
            <div class="w-full flex flex-col gap-2">
              <div class="flex items-center justify-between py-2 px-2.5 rounded-[9px]" style="background:#FEFCE8;">
                <div class="flex items-center gap-1.5">
                  <div class="w-2 h-2 rounded-full bg-[#FFE572] shrink-0"/>
                  <span class="font-switzer text-[10px] text-[#92400E]">Solar</span>
                </div>
                <span class="font-switzer font-semibold text-[11px] text-[#030303]">{fmt(solarUsed)} kWh</span>
              </div>
              <div class="flex items-center justify-between py-2 px-2.5 rounded-[9px]" style="background:#F0F4FF;">
                <div class="flex items-center gap-1.5">
                  <div class="w-2 h-2 rounded-full bg-[#66A3FF] shrink-0"/>
                  <span class="font-switzer text-[10px] text-[#1D4ED8]">Netz</span>
                </div>
                <span class="font-switzer font-semibold text-[11px] text-[#030303]">{fmt(gridImport)} kWh</span>
              </div>
            </div>
          </div>

          <!-- Divider -->
          <div class="self-stretch w-px shrink-0" style="background:#f0ece6;"></div>

          <!-- Right: 24h chart -->
          <div class="flex-1 flex flex-col gap-2 min-w-0">

        <!-- 24h chart -->
        <svg viewBox="0 0 {EW} {EH}" class="w-full" style="overflow:visible; cursor:crosshair;"
          on:mousemove={(e) => { const r=e.currentTarget.getBoundingClientRect(); const x=(e.clientX-r.left)*(EW/r.width); eHoverH=Math.max(0,Math.min(23,Math.round(x/(EW/23)))); }}
          on:mouseleave={() => eHoverH=null}>
          <defs>
            <linearGradient id="eGradSolar" x1="0" y1="0" x2="0" y2="1">
              <stop offset="0%" stop-color="#38BDF8" stop-opacity="1"/>
              <stop offset="100%" stop-color="#0EA5E9" stop-opacity="0.5"/>
            </linearGradient>
            <linearGradient id="eGradGrid" x1="0" y1="0" x2="0" y2="1">
              <stop offset="0%" stop-color="#B0A89E" stop-opacity="1"/>
              <stop offset="100%" stop-color="#B0A89E" stop-opacity="0.4"/>
            </linearGradient>
            <linearGradient id="eGradBat" x1="0" y1="0" x2="0" y2="1">
              <stop offset="0%" stop-color="#FBBF24" stop-opacity="1"/>
              <stop offset="100%" stop-color="#FBBF24" stop-opacity="0.4"/>
            </linearGradient>
            <filter id="eTipShadow" x="-30%" y="-30%" width="160%" height="160%">
              <feDropShadow dx="0" dy="4" stdDeviation="8" flood-color="#000" flood-opacity="0.22"/>
            </filter>
          </defs>

          <!-- Baseline -->
          <line x1={EP.l} y1={EP.t+ecH} x2={EW-EP.r} y2={EP.t+ecH} stroke="#EDE8E0" stroke-width="1"/>

          {#each hourlyBars as b}
            {@const x = ex(b.h)}
            {@const prodH  = (b.prod  / maxBarVal) * ecH}
            {@const gridH  = (b.grid  / maxBarVal) * ecH}
            {@const batH   = localBattery && b.batBonus > 0 ? (b.batBonus / maxBarVal) * ecH : 0}
            {@const active = eHoverH === null || eHoverH === b.h}
            <!-- Grey: grid import bar -->
            {#if gridH > 0.5}
              <rect x={x} y={EP.t + ecH - gridH} width={barW} height={gridH}
                fill="url(#eGradGrid)" rx="2"
                opacity={active ? 1 : 0.15} style="transition:opacity 0.15s;"/>
            {/if}
            <!-- Green: solar production bar -->
            {#if prodH > 0.5}
              <rect x={x} y={EP.t + ecH - prodH} width={barW} height={prodH}
                fill="url(#eGradSolar)" rx="2"
                opacity={active ? 1 : 0.15} style="transition:opacity 0.15s;"/>
            {/if}
            <!-- Yellow: battery bonus bar -->
            {#if batH > 0.5}
              <rect x={x} y={EP.t + ecH - prodH - batH} width={barW} height={batH}
                fill="url(#eGradBat)" rx="2"
                opacity={active ? 1 : 0.15} style="transition:opacity 0.15s;"/>
            {/if}
            <!-- Hour label every 6h -->
            {#if b.h % 6 === 0}
              <text x={x + barW/2} y={EH - 2} text-anchor="middle" font-size="7.5" fill="#C0BAB2" font-family="Switzer,sans-serif">{b.h}h</text>
            {/if}
          {/each}

          <!-- Consumption curve — dashed orange -->
          <polyline points={linePoints} fill="none" stroke="#F59E0B" stroke-width="2" stroke-linejoin="round" stroke-dasharray="4 3" opacity="0.9"/>

          <!-- Hover tooltip -->
          {#if eHoverH !== null}
            {@const b = hourlyBars[eHoverH]}
            {@const tx = ex(b.h) + barW/2}
            {@const tipW = 190}
            {@const tipH = localBattery ? 118 : 96}
            {@const tipX = Math.min(Math.max(tx - tipW/2, 0), EW - tipW)}
            {@const tipY = -tipH - 8}
            <line x1={tx} y1={EP.t} x2={tx} y2={EP.t+ecH} stroke="rgba(0,0,0,0.08)" stroke-width="1"/>
            <g class="tip-enter">
              <rect x={tipX} y={tipY} width={tipW} height={tipH} rx="13" fill="#0e0e0e" filter="url(#eTipShadow)"/>
              <rect x={tipX} y={tipY} width={tipW} height={tipH} rx="13" fill="none" stroke="rgba(255,255,255,0.08)" stroke-width="1"/>
              <text x={tipX+14} y={tipY+20} font-size="13" font-weight="700" fill="white" font-family="Switzer,sans-serif">{b.h}:00 – {b.h+1}:00 Uhr</text>
              <line x1={tipX+14} y1={tipY+28} x2={tipX+tipW-14} y2={tipY+28} stroke="#222" stroke-width="1"/>
              <circle cx={tipX+22} cy={tipY+42} r="4" fill="#02B597"/>
              <text x={tipX+32} y={tipY+46} font-size="12" fill="#888" font-family="Switzer,sans-serif">Solarproduktion</text>
              <text x={tipX+tipW-14} y={tipY+46} text-anchor="end" font-size="13" font-weight="700" fill="white" font-family="Switzer,sans-serif">{(b.prod * 1000).toFixed(0)} Wh</text>
              <circle cx={tipX+22} cy={tipY+62} r="4" fill="#B0A89E"/>
              <text x={tipX+32} y={tipY+66} font-size="12" fill="#888" font-family="Switzer,sans-serif">Netzbezug</text>
              <text x={tipX+tipW-14} y={tipY+66} text-anchor="end" font-size="13" font-weight="700" fill="white" font-family="Switzer,sans-serif">{(b.grid * 1000).toFixed(0)} Wh</text>
              {#if localBattery && b.batBonus > 0}
                <circle cx={tipX+22} cy={tipY+82} r="4" fill="#FBBF24"/>
                <text x={tipX+32} y={tipY+86} font-size="12" fill="#888" font-family="Switzer,sans-serif">Batterie</text>
                <text x={tipX+tipW-14} y={tipY+86} text-anchor="end" font-size="13" font-weight="700" fill="white" font-family="Switzer,sans-serif">{(b.batBonus * 1000).toFixed(0)} Wh</text>
                <rect x={tipX+14} y={tipY+100} width={tipW-28} height="12" rx="3" fill="rgba(251,191,36,0.1)"/>
                <text x={tipX+tipW/2} y={tipY+109} text-anchor="middle" font-size="9" fill="#FBBF24" font-family="Switzer,sans-serif">Speicher aktiv</text>
              {:else}
                <rect x={tipX+14} y={tipY+78} width={tipW-28} height="12" rx="3" fill="rgba(2,181,151,0.1)"/>
                <text x={tipX+tipW/2} y={tipY+87} text-anchor="middle" font-size="9" fill="#02B597" font-family="Switzer,sans-serif">{b.grid < 0.005 ? '✓ Netzunabhängig' : 'Teilweise gedeckt'}</text>
              {/if}
            </g>
          {/if}
        </svg>

        <!-- Legend -->
        <div class="flex items-center gap-4 px-1">
          <div class="flex items-center gap-1.5">
            <div class="w-2.5 h-2.5 rounded-sm" style="background:#38BDF8;"></div>
            <span class="font-switzer text-[10px] text-[#8C8C8C]">Solar</span>
          </div>
          <div class="flex items-center gap-1.5">
            <div class="w-2.5 h-2.5 rounded-sm" style="background:#B0A89E;"></div>
            <span class="font-switzer text-[10px] text-[#8C8C8C]">Netzbezug</span>
          </div>
          {#if localBattery}
          <div class="flex items-center gap-1.5">
            <div class="w-2.5 h-2.5 rounded-sm" style="background:#FBBF24;"></div>
            <span class="font-switzer text-[10px] text-[#8C8C8C]">Batterie</span>
          </div>
          {/if}
          <div class="flex items-center gap-1.5 ml-auto">
            <svg width="16" height="6" viewBox="0 0 16 6"><line x1="0" y1="3" x2="16" y2="3" stroke="#F59E0B" stroke-width="2" stroke-dasharray="4 3"/></svg>
            <span class="font-switzer text-[10px] text-[#8C8C8C]">Verbrauch</span>
          </div>
        </div>

        <!-- KPI + 24h timeline -->
        <div class="pt-2 border-t border-[#f0ece6]">
          <!-- Numbers row -->
          <div class="flex items-end justify-between">
            <div>
              <p class="font-switzer text-[10px] text-[#8C8C8C]">Eigendeckung</p>
              <p class="font-switzer font-semibold text-[32px] text-[#030303] leading-none tracking-[-1px] mt-1">
                {indepPct}<span class="text-[17px] font-normal text-[#8C8C8C] ml-0.5">%</span>
              </p>
            </div>
            <div class="text-right">
              <p class="font-switzer text-[10px] text-[#8C8C8C]">davon netzunabhängig</p>
              <p class="font-switzer font-semibold text-[32px] text-[#030303] leading-none tracking-[-1px] mt-1">
                {indepHours}<span class="text-[17px] font-normal text-[#8C8C8C] ml-0.5">h / Tag</span>
              </p>
            </div>
          </div>

          <!-- 24h annotated timeline -->
          <div class="mt-3">
            <!-- Bracket + label above -->
            <div class="relative h-6 mb-1">
              <div class="absolute flex flex-col items-center" style="left:{tlStartPct}%; width:{tlSpanPct}%;">
                <span class="font-switzer font-semibold text-[11px] whitespace-nowrap" style="color:#0EA5E9;">{indepHours}h netzunabhängig</span>
                <div class="w-full flex items-center mt-0.5">
                  <div class="h-[6px] w-px" style="background:#0EA5E9;"></div>
                  <div class="flex-1 h-px" style="background:#0EA5E9; opacity:0.4;"></div>
                  <div class="h-[6px] w-px" style="background:#0EA5E9;"></div>
                </div>
              </div>
            </div>
            <!-- Bar -->
            <div class="relative h-3 rounded-full overflow-hidden" style="background:#EDE8E0;">
              <div class="absolute top-0 h-full" style="left:{tlStartPct}%; width:{tlSpanPct}%; background:linear-gradient(90deg,#7DD3FC,#0EA5E9);"></div>
              {#if localBattery}
                <div class="absolute top-0 h-full" style="left:{(17/24)*100}%; width:{(5/24)*100}%; background:#FBBF24; opacity:0.7;"></div>
              {/if}
              {#each [6,12,18] as tick}
                <div class="absolute top-0 h-full w-px" style="left:{(tick/24)*100}%; background:rgba(255,255,255,0.6);"></div>
              {/each}
            </div>
            <!-- Time labels -->
            <div class="flex justify-between mt-1.5">
              {#each ['0h','6h','12h','18h','24h'] as t}
                <span class="font-switzer text-[9px] text-[#C0BAB2]">{t}</span>
              {/each}
            </div>
          </div>
        </div>
          </div><!-- /right column -->
        </div><!-- /flex row -->
      </div><!-- /combined card -->

      <!-- ── KPI: Installationsdauer ── -->
      <div class="rounded-[16px] px-6 py-5 flex flex-col justify-between" style="background:white; border:1px solid #ede8e0;">
        <!-- Header -->
        <div class="mb-5">
          <p class="font-switzer font-semibold text-[17px] text-[#030303] leading-none">Installationsdauer</p>
          <p class="font-switzer text-[12px] text-[#8C8C8C] mt-0.5">von Auftrag bis Inbetrieb</p>
        </div>

        <!-- Process steps: big cards, no arrows -->
        <div class="flex gap-2">
          {#each [
            { label: 'Offerte',       dur: '1–2 Tage',   desc: 'Individuelles Angebot mit Systemauslegung und Wirtschaftlichkeitsrechnung.',      bg: '#FEF9EE', color: '#B45309', icon: 'doc',    n: '01', barColor: '#D4EFE9' },
            { label: 'Bewilligungen', dur: '2–4 Wochen', desc: 'Baubewilligungen und Netzanmeldung beim lokalen Energieversorger.',                bg: '#EEF3FF', color: '#3B6FE8', icon: 'shield', n: '02', barColor: '#7EDACC' },
            { label: 'Installation',  dur: '2–3 Tage',   desc: 'Montage der Module, Wechselrichter und elektrischer Anschluss durch unser Team.',  bg: '#F5F0FF', color: '#7C3AED', icon: 'wrench', n: '03', barColor: '#1EC4A4' },
            { label: 'Abnahme',       dur: '1 Tag',      desc: 'Inbetriebnahme, Einweisung und offizielle Abnahme – ab sofort Strom vom Dach.',    bg: '#F0FDF4', color: '#16A34A', icon: 'check',  n: '04', barColor: '#02B597' },
          ] as step, si}
            <div class="relative group flex-1 min-w-0 flex flex-col gap-2.5">
              <!-- Card -->
              <div class="rounded-[14px] p-4 flex flex-col justify-between h-[130px] transition-all duration-200 group-hover:-translate-y-1"
                style="background:{step.bg}; box-shadow:0 2px 8px rgba(0,0,0,0.05);">
                <!-- Top: number + icon -->
                <div class="flex items-start justify-between">
                  <span class="font-switzer font-bold text-[26px] leading-none tracking-tight" style="color:{step.color}; opacity:0.15;">{step.n}</span>
                  <div class="w-9 h-9 rounded-[10px] flex items-center justify-center" style="background:rgba(255,255,255,0.75);">
                    {#if step.icon === 'doc'}
                      <svg class="w-5 h-5" viewBox="0 0 24 24" fill="none" stroke={step.color} stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round">
                        <path d="M14 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V8z"/><polyline points="14 2 14 8 20 8"/><line x1="16" y1="13" x2="8" y2="13"/><line x1="16" y1="17" x2="8" y2="17"/>
                      </svg>
                    {:else if step.icon === 'shield'}
                      <svg class="w-5 h-5" viewBox="0 0 24 24" fill="none" stroke={step.color} stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round">
                        <path d="M12 22s8-4 8-10V5l-8-3-8 3v7c0 6 8 10 8 10z"/><polyline points="9 12 11 14 15 10"/>
                      </svg>
                    {:else if step.icon === 'wrench'}
                      <svg class="w-5 h-5" viewBox="0 0 24 24" fill="none" stroke={step.color} stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round">
                        <path d="M14.7 6.3a1 1 0 0 0 0 1.4l1.6 1.6a1 1 0 0 0 1.4 0l3.77-3.77a6 6 0 0 1-7.94 7.94l-6.91 6.91a2.12 2.12 0 0 1-3-3l6.91-6.91a6 6 0 0 1 7.94-7.94l-3.76 3.76z"/>
                      </svg>
                    {:else}
                      <svg class="w-5 h-5" viewBox="0 0 24 24" fill="none" stroke={step.color} stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round">
                        <path d="M22 11.08V12a10 10 0 1 1-5.93-9.14"/><polyline points="22 4 12 14.01 9 11.01"/>
                      </svg>
                    {/if}
                  </div>
                </div>
                <!-- Bottom: label + dur -->
                <div>
                  <p class="font-switzer font-semibold text-[12px] text-[#1a1a1a] leading-tight">{step.label}</p>
                  <p class="font-switzer text-[11px] mt-0.5 font-medium" style="color:{step.color};">{step.dur}</p>
                </div>
              </div>
              <!-- Bar segment aligned under card -->
              <div class="h-[7px] rounded-full w-full" style="background:{step.barColor};"></div>

              <!-- Hover tooltip -->
              <!-- Hover tooltip -->
              <div class="absolute top-[calc(100%+14px)] left-1/2 -translate-x-1/2 w-[240px] rounded-[18px] px-5 py-5 pointer-events-none
                opacity-0 group-hover:opacity-100 transition-opacity duration-200"
                style="background:#0e0e0e; box-shadow:0 16px 40px rgba(0,0,0,0.35); z-index:50;">
                <div class="absolute bottom-full left-1/2 -translate-x-1/2 w-0 h-0" style="border-left:8px solid transparent; border-right:8px solid transparent; border-bottom:8px solid #0e0e0e;"></div>
                <p class="font-switzer font-semibold text-[17px] text-white leading-none">{step.label}</p>
                <p class="font-switzer text-[14px] font-semibold mt-2 mb-3" style="color:{step.color};">{step.dur}</p>
                <div style="height:1px; background:#222; margin-bottom:12px;"></div>
                <p class="font-switzer text-[13px] text-[#888] leading-[1.6]">{step.desc}</p>
              </div>
            </div>
          {/each}
        </div>

        <!-- Big KPI -->
        <div class="pt-3 border-t border-[#f0ece6]">
          <p class="font-switzer font-semibold text-[38px] text-[#030303] leading-none tracking-[0px]">
            4–6 <span class="text-[22px] font-normal text-[#8C8C8C]">Wochen</span>
          </p>
          <p class="font-switzer text-[12px] text-[#8C8C8C] mt-2">schlüsselfertig & betriebsbereit</p>
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
