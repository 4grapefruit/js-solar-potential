<script lang="ts">
  import { onMount } from 'svelte';
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
  type HeatingType = 'heatpump' | 'electric' | 'other';
  let heatingType: HeatingType = 'heatpump';
  const heatingTypes: { key: HeatingType; label: string }[] = [
    { key: 'heatpump', label: 'Wärmepumpe'    },
    { key: 'electric', label: 'Elektroheizung' },
    { key: 'other',    label: 'Andere'          },
  ];
  let flowPaused   = false;

  // ── Hover highlight ────────────────────────────────────────────────────────
  type FlowGroup = 'solar' | 'einspeisung' | 'netz' | 'batterie' | 'eigenverbrauch' | 'gesamtverbrauch' | 'ev';
  let hoveredFlow: FlowGroup | null = null;

  // Reactive derived opacities — Svelte tracks hoveredFlow directly here
  // Lines
  $: oSolarHouse   = !hoveredFlow || hoveredFlow==='solar' || hoveredFlow==='eigenverbrauch' || hoveredFlow==='gesamtverbrauch' ? 1 : 0.2;
  $: oSolarBat     = !hoveredFlow || hoveredFlow==='solar' || hoveredFlow==='batterie'                                         ? 1 : 0.2;
  $: oEinspeisung  = !hoveredFlow || hoveredFlow==='solar' || hoveredFlow==='einspeisung' || hoveredFlow==='netz'              ? 1 : 0.2;
  $: oNetz         = !hoveredFlow || hoveredFlow==='netz'  || hoveredFlow==='gesamtverbrauch' || hoveredFlow==='solar'         ? 1 : 0.2;
  $: oBatHouse     = !hoveredFlow || hoveredFlow==='batterie' || hoveredFlow==='gesamtverbrauch'                               ? 1 : 0.2;
  $: oEV           = !hoveredFlow || hoveredFlow==='ev' || hoveredFlow==='gesamtverbrauch'                                     ? 1 : 0.2;
  // Boxes
  $: bSolar          = !hoveredFlow || hoveredFlow==='solar' || hoveredFlow==='einspeisung' || hoveredFlow==='batterie' || hoveredFlow==='eigenverbrauch' || hoveredFlow==='gesamtverbrauch' || hoveredFlow==='netz' ? 1 : 0.25;
  $: bEinspeisung    = !hoveredFlow || hoveredFlow==='einspeisung' || hoveredFlow==='solar' || hoveredFlow==='netz'                                                                           ? 1 : 0.25;
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
  $: houseCons = yearlyKwhEnergyConsumption * sm.cons;                  // Hausverbrauch (ohne EV)
  $: sCons   = houseCons + evKwh;                                        // Gesamtverbrauch

  // EMS-Prioritäten: 1. Eigenverbrauch → 2. Batterie → 3. Ladestation → 4. Netzeinspeisung
  const BATTERY_KWH_YEAR = 3000; // ~10 kWh × 300 Zyklen/Jahr
  // Prio 1: Eigenverbrauch (direkter Solarstrom ins Haus)
  $: directKwh  = Math.min(sProd, houseCons);
  $: surplus1   = Math.max(0, sProd - directKwh);
  // Prio 2: Batterie laden
  $: batteryKwh = localBattery ? Math.min(surplus1, BATTERY_KWH_YEAR) : 0;
  $: surplus2   = Math.max(0, surplus1 - batteryKwh);
  // Prio 3: Ladestation E-Auto
  $: evFromSolar = evCharging ? Math.min(evKwh, surplus2) : 0;
  $: surplus3    = Math.max(0, surplus2 - evFromSolar);
  // Prio 4: Netzeinspeisung (max. 70% der Gesamtproduktion)
  $: einspeisKwh = Math.min(surplus3, sProd * 0.70);
  // Netz-Import: was Solar + Batterie nicht decken kann
  $: solarUsed  = directKwh + batteryKwh + evFromSolar;
  $: gridImport = Math.max(0, sCons - solarUsed);

  $: roofPct    = sCons > 0 ? Math.min(100, Math.round((solarUsed / sCons) * 100)) : 0;
  $: netGridPct = 100 - roofPct;

  // kW peak estimates for labels
  $: solarKwPeak = Math.max(0.1, sProd / 365 / 5.5 * 1.8);
  $: gridKwPeak  = Math.max(0,   gridImport / 365 / 14);
  $: battPct     = localBattery && sProd > 0
    ? Math.min(100, Math.round((batteryKwh / (sProd * 0.35)) * 100)) : 0;

  // ── Monthly savings chart ─────────────────────────────────────────────────
  const MONTHS = ['J','F','M','A','M','J','J','A','S','O','N','D'];
  const MONTHS_SHORT = ['Jan','Feb','Mär','Apr','Mai','Jun','Jul','Aug','Sep','Okt','Nov','Dez'];
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
    const solarPct    = mCons > 0 ? Math.round((mDirect / mCons) * 100) : 0;
    return { label, costWithout, costWith: Math.max(0, costWith), saving: costWithout - Math.max(0, costWith), gridWith: Math.round(mGrid), gridWithout: Math.round(mCons), solarPct };
  });
  $: maxMonthlyCost = Math.max(...monthlyData.map(d => d.costWithout), 1);
  $: maxMonthlySaving = Math.max(...monthlyData.map(d => d.saving), 1);
  $: avgMonthlySaving = monthlyData.reduce((s, d) => s + d.saving, 0) / 12;

  // ── Monthly chart dimensions ───────────────────────────────────────────────
  const MW=300, MH=110, MP={t:4,r:0,b:15,l:0};
  const mcH=MH-MP.t-MP.b;
  const mStep=MW/12, mBW=mStep*0.55;
  function mpx(i:number){return i*mStep+mStep/2-mBW/2;}
  $: mMaxCost = maxMonthlySaving;
  function mScaledH(v:number, max:number){return max>0?(v/max)*mcH:0;}
  function mScaledY(v:number, max:number){return MP.t+mcH-mScaledH(v,max);}
  let mHoverIdx: number | null = null;
  const MONTH_FULL = ['Januar','Februar','März','April','Mai','Juni','Juli','August','September','Oktober','November','Dezember'];

  // ── Financing calculator ──────────────────────────────────────────────────
  let loanType: 'hypothek' | 'kredit' = 'hypothek';
  let loanRate = 1.5;
  let loanYears = 20;
  function setLoanType(t: 'hypothek' | 'kredit') {
    loanType = t;
    if (t === 'hypothek') { loanRate = 1.5; loanYears = 20; }
    else { loanRate = 3.5; loanYears = 8; }
  }
  const loanTypeOptions: {k: 'hypothek' | 'kredit', l: string}[] = [{k:'hypothek', l:'Hypothek'}, {k:'kredit', l:'Kredit'}];
  const RATE_MIN = 0.5, RATE_MAX = 8;
  const YEAR_MIN = 5, YEAR_MAX = 25;
  $: rateFillPct = ((loanRate - RATE_MIN) / (RATE_MAX - RATE_MIN)) * 100;
  $: yearFillPct = ((loanYears - YEAR_MIN) / (YEAR_MAX - YEAR_MIN)) * 100;
  $: loanPrincipal = Math.max(0, installationCostTotal - solarIncentives);
  $: loanMonthlyRate = loanRate / 100 / 12;
  $: loanNMonths = loanYears * 12;
  $: monthlyPayment = loanMonthlyRate > 0
    ? loanPrincipal * loanMonthlyRate / (1 - Math.pow(1 + loanMonthlyRate, -loanNMonths))
    : loanPrincipal / loanNMonths;
  $: hoverSaving = mHoverIdx !== null ? monthlyData[mHoverIdx].saving : avgMonthlySaving;
  $: hoverNet = hoverSaving - monthlyPayment;
  $: paymentLinePct = mMaxCost > 0 ? Math.min((monthlyPayment / mMaxCost) * 75, 75) : 0;

  // ── 24h hourly chart ──────────────────────────────────────────────────────
  const HOURS = Array.from({ length: 24 }, (_, i) => i);
  const baseConsProfile = [
    0.20,0.15,0.12,0.12,0.15,0.20,
    0.35,0.50,0.40,0.35,0.30,0.30,
    0.45,0.35,0.30,0.30,0.35,0.50,
    0.60,0.55,0.50,0.45,0.35,0.25,
  ];
  const baseConsSum = baseConsProfile.reduce((a,b) => a+b, 0);
  $: peakKw      = installationSizeKw;
  // Saisonale Peak-Leistung (Nennleistung × saisonaler Faktor)
  const seasonPeakMult: Record<Season, number> = {
    year: 1.00, spring: 0.95, summer: 0.90, autumn: 0.80, winter: 0.45,
  };
  $: curvePeakKw = installationSizeKw * (seasonPeakMult[activeSeason] ?? 1.0);
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
  $: maxHourly = Math.max(...hourlyData.map(d => Math.max(d.prod, d.cons)), installationSizeKw, 1);

  // ── Tagesproduktion Chart ─────────────────────────────────────────────────
  const BW=520, BH=230, BP={t:14, r:16, b:28, l:44};
  const bcW=BW-BP.l-BP.r, bcH=BH-BP.t-BP.b;
  const chartHStart=6, chartHEnd=21, chartHSpan=chartHEnd-chartHStart;
  function bpx(h:number){return BP.l + (h-chartHStart)/chartHSpan*bcW;}
  function bpy(v:number){return BP.t + bcH - (v/maxHourly)*bcH;}

  // Jahreszeitabhängige Kurvenparameter
  const seasonCurve: Record<Season, {peak:number, wMorn:number, wEve:number, hStart:number, hEnd:number}> = {
    year:   { peak:13.0, wMorn:3.8, wEve:3.8, hStart:6.0, hEnd:20.0 },
    spring: { peak:13.0, wMorn:3.6, wEve:3.9, hStart:6.0, hEnd:20.5 },
    summer: { peak:13.0, wMorn:3.4, wEve:4.2, hStart:5.5, hEnd:21.0 },
    autumn: { peak:12.5, wMorn:3.2, wEve:3.0, hStart:7.0, hEnd:19.0 },
    winter: { peak:12.0, wMorn:2.4, wEve:2.2, hStart:8.0, hEnd:16.5 },
  };
  // Wolkenschatten-Ereignisse pro Saison: [Zeitpunkt (h), Tiefe (0–1), Breite (h)]
  // → sanfte gauss'sche Einbrüche statt harter Stufenfunktionen
  const seasonDips: Record<Season, [number, number, number][]> = {
    year:   [[9.5,  0.18, 1.1], [14.0, 0.13, 1.4]],
    summer: [[9.0,  0.14, 1.0], [15.0, 0.10, 1.8]],
    spring: [[10.0, 0.20, 1.1], [13.5, 0.15, 1.3]],
    autumn: [[9.5,  0.22, 1.0], [12.5, 0.17, 1.4]],
    winter: [[11.5, 0.20, 1.2]],
  };

  function applyDips(base: number, t: number, dips: [number, number, number][]): number {
    let v = base;
    for (const [tc, depth, width] of dips) {
      const dd = (t - tc) / width;
      v *= (1 - depth * Math.exp(-dd * dd * 2.5));
    }
    return Math.max(0, v);
  }

  $: sc = seasonCurve[activeSeason];
  $: solarCurvePts = (() => {
    const pts:{x:number,y:number}[] = [];
    const dips = seasonDips[activeSeason];
    for (let t = chartHStart; t <= chartHEnd; t += 0.25) {
      let v = 0;
      if (t >= sc.hStart && t <= sc.hEnd) {
        const dt = t < sc.peak ? (t - sc.peak) / sc.wMorn : (t - sc.peak) / sc.wEve;
        const base = curvePeakKw * Math.exp(-dt * dt * 0.7);
        // Fade to exactly 0 at edges over 0.5h window
        const fadeIn  = Math.min(1, (t - sc.hStart) / 0.5);
        const fadeOut = Math.min(1, (sc.hEnd - t) / 0.5);
        v = applyDips(base, t, dips) * fadeIn * fadeOut;
      }
      pts.push({ x: bpx(t), y: bpy(v) });
    }
    return pts;
  })();
  $: batCurvePts = (() => {
    const pts:{x:number,y:number}[] = [];
    for (let t = chartHStart; t <= chartHEnd; t += 0.25) {
      let prod = 0;
      if (t >= sc.hStart && t <= sc.hEnd) {
        const dt = t < sc.peak ? (t - sc.peak) / sc.wMorn : (t - sc.peak) / sc.wEve;
        prod = Math.max(0, curvePeakKw * Math.exp(-dt*dt*0.7));
      }
      const h = Math.round(t);
      const cons = hourlyData[Math.min(h,23)]?.cons ?? 0;
      const charge = Math.max(0, prod - cons) * 0.4;
      pts.push({ x: bpx(t), y: BP.t + bcH - (charge/maxHourly)*bcH });
    }
    return pts;
  })();
  $: peakY = bpy(installationSizeKw);
  $: solarLineD = solarCurvePts.length > 1
    ? solarCurvePts.map((p,i)=>`${i===0?'M':'L'} ${p.x.toFixed(1)} ${p.y.toFixed(1)}`).join(' ')
    : '';
  $: solarAreaD = solarCurvePts.length > 1
    ? `M ${solarCurvePts[0].x.toFixed(1)} ${BP.t+bcH} `
      + solarCurvePts.map(p=>`L ${p.x.toFixed(1)} ${p.y.toFixed(1)}`).join(' ')
      + ` L ${solarCurvePts[solarCurvePts.length-1].x.toFixed(1)} ${BP.t+bcH} Z`
    : '';
  $: batLineD = batCurvePts.length > 1
    ? batCurvePts.map((p,i)=>`${i===0?'M':'L'} ${p.x.toFixed(1)} ${p.y.toFixed(1)}`).join(' ')
    : '';

  // Hover für Tagesproduktions-Chart
  let bHoverT: number | null = null;
  $: bHoverProd = bHoverT !== null ? (() => {
    if (bHoverT! < sc.hStart || bHoverT! > sc.hEnd) return 0;
    const dt = bHoverT! < sc.peak ? (bHoverT! - sc.peak) / sc.wMorn : (bHoverT! - sc.peak) / sc.wEve;
    const base = curvePeakKw * Math.exp(-dt * dt * 0.7);
    return applyDips(base, bHoverT!, seasonDips[activeSeason]);
  })() : 0;

  const bStep=bcW/24, bBarW=bStep*0.55;
  function bph(v:number){return (v/maxHourly)*bcH;}
  $: prodPts = '';
  $: batPts  = '';

  function smoothPath(pts: {x:number,y:number}[]): string {
    if (pts.length < 2) return '';
    let d = `M ${pts[0].x.toFixed(1)} ${pts[0].y.toFixed(1)}`;
    for (let i = 1; i < pts.length; i++) {
      const p0 = pts[Math.max(0, i-2)];
      const p1 = pts[i-1];
      const p2 = pts[i];
      const p3 = pts[Math.min(pts.length-1, i+1)];
      const cp1x = p1.x + (p2.x - p0.x) / 6;
      const cp1y = p1.y + (p2.y - p0.y) / 6;
      const cp2x = p2.x - (p3.x - p1.x) / 6;
      const cp2y = p2.y - (p3.y - p1.y) / 6;
      d += ` C ${cp1x.toFixed(1)} ${cp1y.toFixed(1)} ${cp2x.toFixed(1)} ${cp2y.toFixed(1)} ${p2.x.toFixed(1)} ${p2.y.toFixed(1)}`;
    }
    return d;
  }
  $: prodPath = smoothPath(hourlyData.map(d => ({ x: BP.l + d.h * bStep + bStep/2, y: bpy(d.prod) })));
  $: batPath  = smoothPath(hourlyData.map(d => { const charge = Math.max(0, d.prod - d.cons) * 0.4; return { x: BP.l + d.h * bStep + bStep/2, y: BP.t + bcH - (charge/maxHourly)*bcH }; }));

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
  $: lMin=Math.min(...cumWithSolar.filter(v=>v>0), 0);
  $: lRange = lMax - lMin;
  $: lXS=lcW/Math.max(installationLifeSpan-1,1);
  function lpx(i:number){return LP.l+i*lXS;}
  $: lpy = (v:number) => LP.t+lcH-((v-lMin)/lRange)*lcH;

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
  $: lBEFrac = (() => {
    if (breakEvenYear < 0) return -1;
    if (breakEvenYear === 0) return 0;
    const y0 = breakEvenYear - 1;
    const s0 = cumWithSolar[y0],    s1 = cumWithSolar[breakEvenYear];
    const n0 = cumWithoutSolar[y0], n1 = cumWithoutSolar[breakEvenYear];
    const denom = (s1 - s0) - (n1 - n0);
    return y0 + (denom !== 0 ? Math.max(0, Math.min(1, (n0 - s0) / denom)) : 0.5);
  })();
  $: lBEX = lBEFrac >= 0 ? lpx(lBEFrac) : -999;
  $: lBEY = (() => {
    if (lBEFrac < 0) return -999;
    const y0 = Math.floor(lBEFrac);
    const y1 = Math.min(y0 + 1, cumWithSolar.length - 1);
    const t = lBEFrac - y0;
    return lpy(cumWithSolar[y0] + t * (cumWithSolar[y1] - cumWithSolar[y0]));
  })();
  $: lHoverNearBE = lHoverYear !== null && breakEvenYear >= 0 && Math.abs(lHoverX - lBEX) < 15;
  $: lYLab  =[0,0.333,0.667,1].map(f=>({val:lMin+lRange*f,y:lpy(lMin+lRange*f)}));
  $: lXLab  =Array.from({length:Math.floor(installationLifeSpan/5)+1},(_,i)=>i*5)
    .filter(y=>y<installationLifeSpan)
    .map(y=>({year:new Date().getFullYear()+y,x:lpx(y)}));

  // ── Semi-donut ────────────────────────────────────────────────────────────
  // Gauge goes left (9 o'clock) → top (12 o'clock) → right (3 o'clock)
  // In our coord system (0°=top, clockwise): left=270°, right=90°, through top
  const DR=58, DRI=36, DCX=110, DCY=72;
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
  const EW = 460, EH = 110, EP = { t: 8, r: 12, b: 24, l: 12 };
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
  $: maxBarVal = Math.max(...hourlyBars.map(b => Math.max(b.prod, b.cons)), 1);
  function ex(h: number) { return EP.l + (h / 23) * (EW - EP.l - EP.r); }
  function ePath(vals: number[]): string {
    const pts = vals.map((v, i) => ({ x: ex(i), y: EP.t + ecH - (v / maxBarVal) * ecH }));
    if (pts.length < 2) return '';
    let d = `M ${pts[0].x.toFixed(1)} ${pts[0].y.toFixed(1)}`;
    for (let i = 1; i < pts.length; i++) {
      const p0 = pts[Math.max(0, i-2)], p1 = pts[i-1], p2 = pts[i], p3 = pts[Math.min(pts.length-1, i+1)];
      const cp1x = p1.x+(p2.x-p0.x)/6, cp1y = p1.y+(p2.y-p0.y)/6;
      const cp2x = p2.x-(p3.x-p1.x)/6, cp2y = p2.y-(p3.y-p1.y)/6;
      d += ` C ${cp1x.toFixed(1)} ${cp1y.toFixed(1)} ${cp2x.toFixed(1)} ${cp2y.toFixed(1)} ${p2.x.toFixed(1)} ${p2.y.toFixed(1)}`;
    }
    return d;
  }
  $: solarVals  = hourlyBars.map(b => b.prod);
  $: consVals   = hourlyBars.map(b => b.cons);
  $: solarLinePath = ePath(solarVals);
  $: consLinePath  = ePath(consVals);
  // Closed solar area (to baseline)
  $: solarAreaPath = solarLinePath
    + ` L ${ex(23).toFixed(1)} ${(EP.t+ecH).toFixed(1)}`
    + ` L ${ex(0).toFixed(1)} ${(EP.t+ecH).toFixed(1)} Z`;
  // Gap zone: where consumption > solar — closed shape between cons line (top) and solar line (bottom)
  $: gapAreaPath = (() => {
    // Forward along cons line where cons > solar, then back along solar
    const pts: string[] = [];
    let inGap = false;
    let d = '';
    for (let i = 0; i < 24; i++) {
      const overload = consVals[i] > solarVals[i];
      const cx = ex(i), cy = EP.t + ecH - (consVals[i] / maxBarVal) * ecH;
      const sx = ex(i), sy = EP.t + ecH - (solarVals[i] / maxBarVal) * ecH;
      if (overload && !inGap) { d += `M ${cx.toFixed(1)} ${cy.toFixed(1)} `; inGap = true; }
      else if (overload) { d += `L ${cx.toFixed(1)} ${cy.toFixed(1)} `; }
      else if (!overload && inGap) {
        // close gap: go back to solar line
        for (let j = i - 1; j >= 0; j--) {
          if (consVals[j] <= solarVals[j]) break;
          d += `L ${ex(j).toFixed(1)} ${(EP.t + ecH - (solarVals[j] / maxBarVal) * ecH).toFixed(1)} `;
        }
        d += 'Z ';
        inGap = false;
      }
    }
    if (inGap) {
      for (let j = 23; j >= 0; j--) {
        if (consVals[j] <= solarVals[j]) break;
        d += `L ${ex(j).toFixed(1)} ${(EP.t + ecH - (solarVals[j] / maxBarVal) * ecH).toFixed(1)} `;
      }
      d += 'Z';
    }
    return d;
  })();
  const barW = 14;
  let eHoverH: number | null = null;
  $: solarStartH  = hourlyBars.findIndex(b => b.prod > 0.001);
  $: solarEndH    = 23 - [...hourlyBars].reverse().findIndex(b => b.prod > 0.001);

  // ── 24h Radial Clock ──────────────────────────────────────────────────────
  const CCX = 150, CCY = 150, COR = 112, CIR = 89;
  const CCAP = 5; // inner 3D cap width in px
  function clockArc(h: number): string {
    const gap = 1.8;
    const a1 = ((h / 24) * 360 - 90 + gap / 2) * (Math.PI / 180);
    const a2 = (((h + 1) / 24) * 360 - 90 - gap / 2) * (Math.PI / 180);
    const x1 = CCX + COR * Math.cos(a1), y1 = CCY + COR * Math.sin(a1);
    const x2 = CCX + COR * Math.cos(a2), y2 = CCY + COR * Math.sin(a2);
    const x3 = CCX + CIR * Math.cos(a2), y3 = CCY + CIR * Math.sin(a2);
    const x4 = CCX + CIR * Math.cos(a1), y4 = CCY + CIR * Math.sin(a1);
    return `M${x1.toFixed(2)},${y1.toFixed(2)} A${COR},${COR} 0 0 1 ${x2.toFixed(2)},${y2.toFixed(2)} L${x3.toFixed(2)},${y3.toFixed(2)} A${CIR},${CIR} 0 0 0 ${x4.toFixed(2)},${y4.toFixed(2)}Z`;
  }
  // Inner dark cap — thin strip on the inner edge of each block (3D inset shadow effect)
  function clockArcCap(h: number): string {
    const gap = 1.8;
    const innerR = CIR + CCAP;
    const a1 = ((h / 24) * 360 - 90 + gap / 2) * (Math.PI / 180);
    const a2 = (((h + 1) / 24) * 360 - 90 - gap / 2) * (Math.PI / 180);
    const x1 = CCX + innerR * Math.cos(a1), y1 = CCY + innerR * Math.sin(a1);
    const x2 = CCX + innerR * Math.cos(a2), y2 = CCY + innerR * Math.sin(a2);
    const x3 = CCX + CIR    * Math.cos(a2), y3 = CCY + CIR    * Math.sin(a2);
    const x4 = CCX + CIR    * Math.cos(a1), y4 = CCY + CIR    * Math.sin(a1);
    return `M${x1.toFixed(2)},${y1.toFixed(2)} A${innerR},${innerR} 0 0 1 ${x2.toFixed(2)},${y2.toFixed(2)} L${x3.toFixed(2)},${y3.toFixed(2)} A${CIR},${CIR} 0 0 0 ${x4.toFixed(2)},${y4.toFixed(2)}Z`;
  }
  $: peakSolarH = hourlyBars.reduce((best, b, h) => b.prod > hourlyBars[best].prod ? h : best, 12);
  $: peakClockAngle = (peakSolarH / 24 * 360 - 90) * Math.PI / 180;
  const clockMidR = (COR + CIR) / 2;
  let clockHoverH: number | null = null;
  let clockMounted = false;
  let clockDisplayH = 0;
  let clockMountAnimDone = false;
  onMount(() => {
    clockMounted = true;
    const target = indepHours;
    const dur = 900;
    setTimeout(() => {
      const t0 = performance.now();
      const step = (now: number) => {
        const p = Math.min((now - t0) / dur, 1);
        const e = 1 - Math.pow(1 - p, 3); // easeOutCubic
        clockDisplayH = Math.round(e * target);
        if (p < 1) requestAnimationFrame(step);
        else clockDisplayH = target;
      };
      requestAnimationFrame(step);
    }, 400);
    // After mount animation completes, switch to fast unhover fade
    setTimeout(() => { clockMountAnimDone = true; }, 1500);
  });
  // Solar window outer arc (sunrise → sunset indicator)
  $: solarWindowArcPath = (() => {
    const gap = 1.8;
    const r = COR + CCAP + 4;
    const a1 = (solarStartH / 24 * 360 - 90 + gap / 2) * (Math.PI / 180);
    const a2 = ((solarEndH + 1) / 24 * 360 - 90 - gap / 2) * (Math.PI / 180);
    const x1 = CCX + r * Math.cos(a1), y1 = CCY + r * Math.sin(a1);
    const x2 = CCX + r * Math.cos(a2), y2 = CCY + r * Math.sin(a2);
    const spanH = (solarEndH + 1) - solarStartH;
    return `M${x1.toFixed(2)},${y1.toFixed(2)} A${r},${r} 0 ${spanH > 12 ? 1 : 0} 1 ${x2.toFixed(2)},${y2.toFixed(2)}`;
  })();
  $: tlStartPct   = (solarStartH / 24) * 100;
  $: tlEndPct     = (solarEndH   / 24) * 100;
  $: tlSpanPct    = tlEndPct - tlStartPct;
</script>

<style>
  /* ── EV Hover-Karte ─────────────────────────────────────────────────────── */
  /* Position oben-links fixiert, wächst nach unten-rechts */
  .ev-float {
    transform-origin: top left;
  }

  /* Karte: kompakt im Normalzustand, expandiert beim Hover */
  .ev-card-hover {
    width: 130px;
    transition: width 0.35s ease;
    overflow: hidden;
  }

  .ev-card-hover:hover {
    width: 230px;
  }

  /* Solar-Info unten — klappt vertikal auf */
  .ev-expand {
    max-height: 0;
    opacity: 0;
    overflow: hidden;
    transition: max-height 0.35s ease, opacity 0.25s ease;
  }

  .ev-card-hover:hover .ev-expand {
    max-height: 120px;
    opacity: 1;
  }

  /* ── Clock reveal animations ────────────────────────────────────────────── */
  @keyframes segReveal {
    from { opacity: 0; transform: scale(0.68); }
    to   { opacity: 1; transform: scale(1);    }
  }
  @keyframes fadeUp {
    from { opacity: 0; transform: translateY(12px); }
    to   { opacity: 1; transform: translateY(0);    }
  }
  @keyframes fadeIn {
    from { opacity: 0; }
    to   { opacity: 1; }
  }
  .seg-reveal {
    transform-box: view-box;
    transform-origin: 150px 150px;
    animation: segReveal 0.55s cubic-bezier(0.34, 1.4, 0.64, 1) both;
  }
  .stats-left  { animation: fadeUp 0.5s ease-out 680ms both; }
  .stats-right { animation: fadeUp 0.5s ease-out 780ms both; }
  .clock-center-fade  { animation: fadeIn 0.4s ease-out 900ms both; }
  .clock-unhover-fade { animation: fadeIn 0.18s ease-out 0ms both; }
  .clock-svg-wrapper  { width: 100%; }
  /* Segment ring scales on hover — center stays stable */
  .seg-ring {
    transform-box: view-box;
    transform-origin: 150px 150px;
    transition: transform 0.38s cubic-bezier(0.34, 1.28, 0.64, 1);
  }
  .seg-ring-hovered { transform: scale(1.08); }
  /* Rim depth circles scale faster/further than segments — break away outward */
  .rim-depth-ring {
    transform-box: view-box;
    transform-origin: 150px 150px;
    transition: transform 0.38s cubic-bezier(0.34, 1.28, 0.64, 1);
  }
  .rim-depth-ring-hovered { transform: scale(1.10); }
  .progress-ring {
    transform-box: view-box;
    transform-origin: 150px 150px;
    transition: transform 0.38s cubic-bezier(0.34, 1.28, 0.64, 1);
  }
  .progress-ring-hovered { transform: scale(1.18); }

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


  /* Pause/Play toggle button */
  .flow-toggle-btn {
    display: flex;
    align-items: center;
    border-radius: 999px;
    overflow: hidden;
    background: rgba(3,3,3,0.28);
    backdrop-filter: blur(8px);
    border: 1px solid rgba(255,255,255,0.10);
    cursor: pointer;
    transition: background 0.25s ease;
  }
  .flow-toggle-btn:hover {
    background: rgba(3,3,3,0.55);
  }
  .flow-toggle-icon {
    width: 40px;
    height: 40px;
    display: flex;
    align-items: center;
    justify-content: center;
    flex-shrink: 0;
  }
  .flow-toggle-label {
    max-width: 0;
    overflow: hidden;
    white-space: nowrap;
    font-size: 12px;
    font-weight: 600;
    color: white;
    letter-spacing: -0.2px;
    transition: max-width 0.35s cubic-bezier(0.4,0,0.2,1), padding-right 0.35s cubic-bezier(0.4,0,0.2,1);
    padding-right: 0;
  }
  .flow-toggle-btn:hover .flow-toggle-label {
    max-width: 120px;
    padding-right: 14px;
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
            class="flow-toggle-btn absolute"
            style="left:1rem; top:1rem;">
            <!-- Icon circle -->
            <span class="flow-toggle-icon">
              {#if flowPaused}
                <svg width="18" height="18" viewBox="0 0 24 24" fill="white">
                  <path d="M8 5v14l11-7z"/>
                </svg>
              {:else}
                <svg width="18" height="18" viewBox="0 0 24 24" fill="white">
                  <circle cx="7" cy="12" r="2.5"/>
                  <circle cx="17" cy="12" r="2.5"/>
                  <rect x="5" y="6" width="4" height="12" rx="1.5"/>
                  <rect x="15" y="6" width="4" height="12" rx="1.5"/>
                </svg>
              {/if}
            </span>
            <!-- Label slides in on hover -->
            <span class="flow-toggle-label font-switzer">
              {flowPaused ? 'Starten' : 'Pausieren'}
            </span>
          </button>

          <!-- SVG flow overlay: coords map 1:1 to image percentage -->
          <svg class="flow-svg absolute inset-0 w-full h-full pointer-events-none {flowPaused ? 'paused' : ''}"
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
            <div class="absolute ev-float" role="region" aria-label="Ladestation" style="left:75%; top:70%; opacity:{bEV}; transition:opacity 0.5s;"
              on:mouseenter={() => hoveredFlow='ev'} on:mouseleave={() => hoveredFlow=null}>
              <div class="rounded-[10px] shadow-lg ev-card-hover" style="background:rgba(3,3,3,0.84); backdrop-filter:blur(8px);">

                <!-- Basis-Inhalt: oben (Label + Badge) + Hauptzahl -->
                <div class="px-3 py-2">
                  <div class="flex items-center justify-between gap-3 mb-0.5">
                    <div class="flex items-center gap-1.5">
                      <div class="w-2 h-2 rounded-full bg-[#A78BFA]" />
                      <span class="font-switzer text-[10px] text-[#A78BFA]">Ladestation</span>
                    </div>
                  </div>
                  <p class="font-switzer font-semibold text-[17px] text-white leading-none">{fmt(evKwh)} kWh</p>
                  <p class="font-switzer text-[10px] text-[#8C8C8C] mt-0.5">20'000 km / Jahr</p>
                </div>

                <!-- Hover-Erweiterung: Solar-Info unten, volle Breite -->
                <div class="ev-expand">
                  <div class="px-3 pt-2 pb-3" style="border-top:1px solid rgba(167,139,250,0.15)">
                    <div class="flex items-center justify-between mb-1">
                      <div class="flex items-center gap-1">
                        <svg class="w-3 h-3 shrink-0" viewBox="0 0 24 24" fill="#A78BFA"><path d="M7 2v11h3v9l7-12h-4l4-8z"/></svg>
                        <span class="font-switzer text-[11px] text-[#8C8C8C]">Solar lädt täglich</span>
                      </div>
                      <p class="font-switzer font-bold text-[15px] text-white leading-none">{fmt(evFromSolar / 365, 1)} kWh</p>
                    </div>
                    <div class="h-1.5 rounded-full mb-1" style="background:rgba(255,255,255,0.1);">
                      <div class="h-1.5 rounded-full"
                        style="width:{evKwh > 0 ? Math.min(100, Math.round((evFromSolar/evKwh)*100)) : 0}%; background:linear-gradient(90deg,#16A34A,#4ADE80)"/>
                    </div>
                    <div class="flex justify-between">
                      <span class="font-switzer text-[10px] text-[#8C8C8C]">Solar-Abdeckung</span>
                      <span class="font-switzer text-[10px] font-semibold text-[#A78BFA]">{evKwh > 0 ? Math.min(100, Math.round((evFromSolar/evKwh)*100)) : 0}%</span>
                    </div>
                  </div>
                </div>

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

          <!-- Ausstattung -->
          <div class="mt-2 pt-3 border-t border-[#e0dbd4]">
            <p class="font-switzer font-semibold text-[10px] text-[#8C8C8C] tracking-[0.5px] uppercase mb-2">Ausstattung</p>

            <!-- Batterie -->
            <button on:click={() => localBattery=!localBattery}
              class="flex items-center gap-2.5 pl-[5px] pr-4 py-[5px] rounded-[64px] transition-all duration-200 text-left w-full mb-1.5"
              style="background:{localBattery?'#F97316':'white'}">
              <div class="w-[32px] h-[32px] rounded-full flex items-center justify-center shrink-0"
                style="background:{localBattery?'rgba(255,255,255,0.25)':'#f2f2f2'}">
                {#if localBattery}
                  <svg class="w-[16px] h-[16px]" viewBox="0 0 24 24" fill="white">
                    <path d="M9 16.17L4.83 12l-1.42 1.41L9 19 21 7l-1.41-1.41z"/>
                  </svg>
                {:else}
                  <svg class="w-[18px] h-[18px]" viewBox="0 0 24 24" fill="#F97316">
                    <path d="M15.67 4H14V2h-4v2H8.33C7.6 4 7 4.6 7 5.33v15.33C7 21.4 7.6 22 8.33 22h7.33c.74 0 1.34-.6 1.34-1.33V5.33C17 4.6 16.4 4 15.67 4zm-1.67 9h-2v4h-2v-4H8l4-8 4 8h-2z"/>
                  </svg>
                {/if}
              </div>
              <span class="font-switzer font-semibold text-[13px] tracking-[-0.26px]"
                style="color:{localBattery?'white':'#030303'}">Batterie</span>
            </button>

            <!-- Ladestation -->
            <button on:click={() => evCharging=!evCharging}
              class="flex items-center gap-2.5 pl-[5px] pr-4 py-[5px] rounded-[64px] transition-all duration-200 text-left w-full"
              style="background:{evCharging?'#A78BFA':'white'}">
              <div class="w-[32px] h-[32px] rounded-full flex items-center justify-center shrink-0"
                style="background:{evCharging?'rgba(255,255,255,0.25)':'#f2f2f2'}">
                {#if evCharging}
                  <svg class="w-[16px] h-[16px]" viewBox="0 0 24 24" fill="white">
                    <path d="M9 16.17L4.83 12l-1.42 1.41L9 19 21 7l-1.41-1.41z"/>
                  </svg>
                {:else}
                  <svg class="w-[18px] h-[18px]" viewBox="0 0 24 24" fill="#A78BFA">
                    <path d="M18.92 6.01C18.72 5.42 18.16 5 17.5 5h-11c-.66 0-1.21.42-1.42 1.01L3 12v8c0 .55.45 1 1 1h1c.55 0 1-.45 1-1v-1h12v1c0 .55.45 1 1 1h1c.55 0 1-.45 1-1v-8l-2.08-5.99zM6.85 7h10.29l1.08 3.11H5.77L6.85 7zM19 17H5v-5h14v5z"/>
                    <circle cx="7.5" cy="14.5" r="1.5" fill="#A78BFA"/>
                    <circle cx="16.5" cy="14.5" r="1.5" fill="#A78BFA"/>
                  </svg>
                {/if}
              </div>
              <span class="font-switzer font-semibold text-[13px] tracking-[-0.26px]"
                style="color:{evCharging?'white':'#030303'}">Ladestation</span>
            </button>

          </div>

          <!-- Heizung -->
          <div class="pt-3 border-t border-[#e0dbd4]">
            <p class="font-switzer font-semibold text-[10px] text-[#8C8C8C] tracking-[0.5px] uppercase mb-2">Heizung</p>
            {#each heatingTypes as ht}
              <button
                on:click={() => heatingType = ht.key}
                class="flex items-center gap-2.5 pl-[5px] pr-4 py-[5px] rounded-[64px] transition-all duration-200 text-left w-full mb-1.5"
                style="background:{heatingType===ht.key?'#E8E3DB':'white'}; border:{heatingType===ht.key?'1px solid #C5BFB5':'1px solid transparent'};">
                <div class="w-[32px] h-[32px] rounded-full flex items-center justify-center shrink-0"
                  style="background:{heatingType===ht.key?'#C5BFB5':'#f2f2f2'}">
                  {#if heatingType===ht.key}
                    <svg class="w-[16px] h-[16px]" viewBox="0 0 24 24" fill="#3D3530">
                      <path d="M9 16.17L4.83 12l-1.42 1.41L9 19 21 7l-1.41-1.41z"/>
                    </svg>
                  {/if}
                </div>
                <span class="font-switzer font-semibold text-[13px] tracking-[-0.26px]"
                  style="color:#030303;">{ht.label}</span>
              </button>
            {/each}
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
            <p class="font-switzer font-semibold text-[17px] text-white leading-none">Rendite</p>
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
    <div class="grid grid-cols-5 gap-4">

      <!-- ── Tagesproduktion ── -->
      <div class="col-span-2 rounded-[16px] p-6 flex flex-col gap-4" style="background:white; border:1px solid #ede8e0;">
        <div>
          <h3 class="font-switzer font-semibold text-[17px] text-[#030303] tracking-[-0.4px]">Tagesproduktion</h3>
          <p class="font-switzer text-[13px] text-[#8C8C8C] mt-0.5">Typischer Tagesverlauf · {activeSeason === 'year' ? 'Jahresdurchschnitt' : seasons.find(s=>s.key===activeSeason)?.label}</p>
        </div>
        <svg viewBox="0 0 {BW} {BH}" class="w-full" style="overflow:visible; cursor:crosshair;"
          on:mousemove={(e) => { const r=e.currentTarget.getBoundingClientRect(); const x=(e.clientX-r.left)*(BW/r.width); bHoverT = Math.round((x-BP.l)/bcW*chartHSpan*4)/4 + chartHStart; }}
          on:mouseleave={() => bHoverT=null}>
          <defs>
            <filter id="tipShadowB" x="-30%" y="-30%" width="160%" height="160%">
              <feDropShadow dx="0" dy="4" stdDeviation="8" flood-color="#000" flood-opacity="0.25"/>
            </filter>
            <linearGradient id="solarFill" x1="0" y1="0" x2="0" y2="1">
              <stop offset="0%" stop-color="#FFD000" stop-opacity="0.15"/>
              <stop offset="100%" stop-color="#FFD000" stop-opacity="0"/>
            </linearGradient>
          </defs>

          <!-- Grid: Nennleistungs-Referenzlinie -->
          <line x1={BP.l} y1={peakY} x2={BW-BP.r} y2={peakY} stroke="#f0f0f0" stroke-width="1" stroke-dasharray="3 4"/>
          <text x={BP.l-4} y={peakY+4} text-anchor="end" font-size="9" fill="#ccc" font-family="Switzer,sans-serif">{installationSizeKw.toFixed(1)} kWp</text>

          <!-- Gefüllte Fläche -->
          <path d={solarAreaD} fill="url(#solarFill)" style="transition: d 0.5s cubic-bezier(0.4,0,0.2,1);"/>

          <!-- Batterieladung -->
          {#if localBattery}
            <path d={batLineD} fill="none" stroke="#F97316" stroke-width="2" stroke-linecap="round" opacity="0.8" style="transition: d 0.5s cubic-bezier(0.4,0,0.2,1);"/>
          {/if}

          <!-- Solar Linie -->
          <path d={solarLineD} fill="none" stroke="#FFD000" stroke-width="3.5" stroke-linecap="round" stroke-linejoin="round" style="transition: d 0.5s cubic-bezier(0.4,0,0.2,1);"/>

          <!-- Hover -->
          {#if bHoverT !== null && bHoverT >= chartHStart && bHoverT <= chartHEnd}
            {@const hx = bpx(bHoverT)}
            {@const hy = bpy(bHoverProd)}
            {@const tipLeft = hx > BW * 0.55}
            {@const tipX = tipLeft ? hx - 178 : hx + 14}
            {@const tipY = Math.min(hy - 28, BP.t + bcH - 60)}
            <line x1={hx} y1={BP.t} x2={hx} y2={BP.t+bcH} stroke="#030303" stroke-width="1" stroke-dasharray="4 3" opacity="0.12"/>
            <circle cx={hx} cy={hy} r="6" fill="#FFD000" stroke="white" stroke-width="2.5"/>
            <g filter="url(#tipShadowB)">
              <rect x={tipX} y={tipY} width="164" height="54" rx="12" fill="#111"/>
              <text x={tipX+14} y={tipY+22} font-size="15" font-weight="700" fill="white" font-family="Switzer,sans-serif" letter-spacing="-0.3">{Math.floor(bHoverT)}:00 Uhr</text>
              <text x={tipX+14} y={tipY+42} font-size="17" fill="#FFD000" font-weight="700" font-family="Switzer,sans-serif" letter-spacing="-0.5">{bHoverProd.toFixed(2)} kW</text>
            </g>
          {/if}

          <!-- X axis -->
          <line x1={BP.l} y1={BP.t+bcH} x2={BW-BP.r} y2={BP.t+bcH} stroke="#eee" stroke-width="1"/>
          {#each [6,9,12,15,18,21] as h}
            {#if h >= chartHStart && h <= chartHEnd}
              <text x={bpx(h)} y={BH-8} text-anchor="middle" font-size="9" fill="#bbb" font-family="Switzer,sans-serif">{h}h</text>
            {/if}
          {/each}
        </svg>

        <!-- Saison-Toggle Tabs -->
        <div class="flex gap-1 justify-center pt-1">
          <!-- Ganzes Jahr: immer als Text -->
          <button
            on:click={() => activeSeason = 'year'}
            class="px-4 py-2 rounded-full font-switzer font-semibold text-[13px] leading-none"
            style="background:{activeSeason==='year'?'#FFF3D0':'transparent'}; color:{activeSeason==='year'?'#8B6400':'#bbb'}; border:{activeSeason==='year'?'1.5px solid #E5C45A':'1.5px solid transparent'}; transition:all 0.25s;">
            Ganzes Jahr
          </button>
          <!-- Jahreszeiten: Icons -->
          {#each seasons.filter(s=>s.key!=='year') as s}
            <button
              on:click={() => activeSeason = s.key}
              class="w-10 h-10 rounded-full flex items-center justify-center"
              style="background:{activeSeason===s.key?'#FFF3D0':'transparent'}; color:{activeSeason===s.key?'#8B6400':'#bbb'}; border:{activeSeason===s.key?'1.5px solid #E5C45A':'1.5px solid transparent'}; transition:all 0.25s;"
              title={s.label}>
              {#if s.key === 'spring'}
                <svg viewBox="0 0 20 20" width="18" height="18" fill="none" stroke="currentColor" stroke-width="1.6" stroke-linecap="round" stroke-linejoin="round">
                  <path d="M10 18 C10 18 4 13.5 4 9 C4 6 6.5 3.5 10 3.5 C13.5 3.5 16 6 16 9 C16 13.5 10 18 10 18Z"/>
                  <line x1="10" y1="18" x2="10" y2="11"/>
                </svg>
              {:else if s.key === 'summer'}
                <svg viewBox="0 0 20 20" width="18" height="18" fill="none" stroke="currentColor" stroke-width="1.6" stroke-linecap="round">
                  <circle cx="10" cy="10" r="4"/>
                  <line x1="10" y1="1" x2="10" y2="3.5"/><line x1="10" y1="16.5" x2="10" y2="19"/>
                  <line x1="1" y1="10" x2="3.5" y2="10"/><line x1="16.5" y1="10" x2="19" y2="10"/>
                  <line x1="3.3" y1="3.3" x2="5.2" y2="5.2"/><line x1="14.8" y1="14.8" x2="16.7" y2="16.7"/>
                  <line x1="16.7" y1="3.3" x2="14.8" y2="5.2"/><line x1="5.2" y1="14.8" x2="3.3" y2="16.7"/>
                </svg>
              {:else if s.key === 'autumn'}
                <svg viewBox="0 0 20 20" width="18" height="18" fill="none" stroke="currentColor" stroke-width="1.6" stroke-linecap="round" stroke-linejoin="round">
                  <path d="M12.5 3 C17 4.5 17.5 11 12.5 14.5 C8 16.5 3.5 14 4 9.5 C4.5 5.5 9.5 2 12.5 3Z"/>
                  <line x1="4.5" y1="15.5" x2="12" y2="8"/>
                  <line x1="3" y1="17" x2="4.5" y2="15.5"/>
                </svg>
              {:else if s.key === 'winter'}
                <svg viewBox="0 0 20 20" width="18" height="18" fill="none" stroke="currentColor" stroke-width="1.6" stroke-linecap="round">
                  <line x1="10" y1="1.5" x2="10" y2="18.5"/>
                  <line x1="1.5" y1="10" x2="18.5" y2="10"/>
                  <line x1="3.8" y1="3.8" x2="16.2" y2="16.2"/>
                  <line x1="16.2" y1="3.8" x2="3.8" y2="16.2"/>
                  <line x1="7.5" y1="2.5" x2="10" y2="5"/><line x1="12.5" y1="2.5" x2="10" y2="5"/>
                  <line x1="7.5" y1="17.5" x2="10" y2="15"/><line x1="12.5" y1="17.5" x2="10" y2="15"/>
                </svg>
              {/if}
            </button>
          {/each}
        </div>
      </div>

      <!-- ── Eigenverbrauch + Energieunabhängigkeit (combined) ── -->
      <div class="rounded-[16px] p-6 flex flex-col gap-5 col-span-3" style="background:white; border:1px solid #ede8e0;">

        <!-- Header -->
        <div>
          <p class="font-switzer font-semibold text-[17px] text-[#030303] leading-none">Eigenverbrauch & Unabhängigkeit</p>
          <p class="font-switzer text-[12px] text-[#8C8C8C] mt-1.5">Wann das Haus vom Netz unabhängig ist — Stunde für Stunde</p>
        </div>

        <!-- Three-column layout: stats | clock hero | stats -->
        <div class="flex gap-6 items-center flex-1">

          <!-- LEFT: Eigenverbrauch + Jahresverbrauch -->
          <div class="stats-left flex-1 flex flex-col gap-6 justify-center self-stretch pl-1">

            <div class="flex flex-col gap-2">
              <p class="font-switzer text-[10px] font-medium text-[#B0AAA4] uppercase tracking-[1px]">Eigenverbrauch</p>
              <div class="flex items-baseline gap-1 leading-none">
                <span class="font-switzer font-bold text-[44px] text-[#111] leading-none tracking-[-2.5px]">{roofPct}</span>
                <span class="font-switzer font-semibold text-[18px] text-[#C8C2BA] leading-none">%</span>
              </div>
              <div class="h-[4px] rounded-full overflow-hidden" style="background:#EDE8E1;">
                <div class="h-full rounded-full" style="width:{roofPct}%; background:#FFD000; transition:width 0.8s cubic-bezier(0.4,0,0.2,1);"/>
              </div>
              <p class="font-switzer text-[11px] text-[#C8C2BA]">vom eigenen Dach</p>
            </div>

            <div class="h-px" style="background:#F0EDE8;"></div>

            <div class="flex flex-col gap-2">
              <p class="font-switzer text-[10px] font-medium text-[#B0AAA4] uppercase tracking-[1px]">Jahresverbrauch</p>
              {#if solarUsed + gridImport > 0}
                {@const sW = Math.round(solarUsed/(solarUsed+gridImport)*100)}
                <div class="flex flex-col gap-2.5">
                  <div class="flex flex-col gap-1">
                    <div class="flex items-center justify-between">
                      <div class="flex items-center gap-1.5">
                        <div class="w-1.5 h-1.5 rounded-full" style="background:#FFD000;"/>
                        <span class="font-switzer text-[11px] text-[#8C8C8C]">Solar</span>
                      </div>
                      <span class="font-switzer font-semibold text-[12px] text-[#111] tracking-[-0.5px]">{fmt(solarUsed)} kWh</span>
                    </div>
                    <div class="h-[3px] rounded-full overflow-hidden" style="background:#EDE8E1;">
                      <div class="h-full rounded-full" style="width:{sW}%; background:#FFD000;"/>
                    </div>
                  </div>
                  <div class="flex flex-col gap-1">
                    <div class="flex items-center justify-between">
                      <div class="flex items-center gap-1.5">
                        <div class="w-1.5 h-1.5 rounded-full" style="background:#C8C2BA;"/>
                        <span class="font-switzer text-[11px] text-[#8C8C8C]">Netz</span>
                      </div>
                      <span class="font-switzer font-semibold text-[12px] text-[#111] tracking-[-0.5px]">{fmt(gridImport)} kWh</span>
                    </div>
                    <div class="h-[3px] rounded-full overflow-hidden" style="background:#EDE8E1;">
                      <div class="h-full rounded-full" style="width:{100-sW}%; background:#C8C2BA;"/>
                    </div>
                  </div>
                </div>
              {/if}
            </div>

          </div><!-- /left stats -->

          <!-- CENTER: Clock hero -->
          <div class="flex-shrink-0 w-[320px] flex flex-col justify-center items-center gap-3">
            <div class="clock-svg-wrapper">
            <svg viewBox="0 0 300 300" style="width:100%; overflow:visible; cursor:crosshair;"
              on:mousemove={(e) => {
                const r = e.currentTarget.getBoundingClientRect();
                const mx = (e.clientX - r.left) * (300 / r.width) - CCX;
                const my = (e.clientY - r.top) * (300 / r.height) - CCY;
                const dist = Math.sqrt(mx * mx + my * my);
                if (dist >= CIR - 10 && dist <= COR + 36) {
                  const ang = ((Math.atan2(my, mx) * 180 / Math.PI) + 90 + 360) % 360;
                  clockHoverH = Math.floor(ang / 15);
                } else { clockHoverH = null; }
              }}
              on:mouseleave={() => clockHoverH = null}>
              <defs>
                <!-- Center circle inset shadow — ring casts shadow inward -->
                <filter id="clockCenterShadow" x="-20%" y="-20%" width="140%" height="140%">
                  <feDropShadow dx="0" dy="3" stdDeviation="9" flood-color="#000" flood-opacity="0.10"/>
                </filter>
              </defs>

              <!-- Subtle outer tick marks — static, outside scaling group -->
              {#each Array(24) as _, h}
                {@const ta = (h / 24 * 360 - 90) * Math.PI / 180}
                {@const isCard = h % 6 === 0}
                <line
                  x1={CCX + (COR + 5)  * Math.cos(ta)}
                  y1={CCY + (COR + 5)  * Math.sin(ta)}
                  x2={CCX + (COR + (isCard ? 13 : 9)) * Math.cos(ta)}
                  y2={CCY + (COR + (isCard ? 13 : 9)) * Math.sin(ta)}
                  stroke={isCard ? '#C8C2BA' : '#DDD8D2'}
                  stroke-width={isCard ? 1.5 : 1}
                  stroke-linecap="round"/>
              {/each}

              <!-- Segments + solar arc — scale together on hover -->
              <g class="seg-ring" class:seg-ring-hovered={clockHoverH !== null}>
              <!-- Solar window arc -->
              <path d={solarWindowArcPath}
                fill="none" stroke="#FFD000" stroke-width="2"
                stroke-linecap="round" opacity="0.3"/>
              {#each hourlyBars as b, h}
                {@const isIndep   = b.grid < 0.005}
                {@const isPartial = b.prod > 0.001 && !isIndep}
                {@const dimmed    = clockHoverH !== null && clockHoverH !== h}
                <g class="seg-reveal" style="animation-delay: {h * 28}ms;">
                  {#if isIndep}
                    {@const intensity = 0.35 + 0.65 * Math.min(1, b.prod / (maxBarVal * 0.55))}
                    <!-- main face -->
                    <path d={clockArc(h)} fill="#FFD000"
                      opacity={dimmed ? 0.18 : intensity}
                      style="transition:opacity 0.12s;"/>
                    <!-- inner 3D cap — subtle shadow on inner edge -->
                    <path d={clockArcCap(h)} fill="#A07800"
                      opacity={dimmed ? 0.02 : intensity * 0.07}
                      style="transition:opacity 0.12s;"/>
                  {:else if isPartial}
                    {@const softness = 0.18 + 0.22 * Math.min(1, b.prod / (maxBarVal * 0.4))}
                    <path d={clockArc(h)} fill="#FFD000"
                      opacity={dimmed ? 0.06 : softness}
                      style="transition:opacity 0.12s;"/>
                    <path d={clockArcCap(h)} fill="#A07800"
                      opacity={dimmed ? 0.01 : softness * 0.07}
                      style="transition:opacity 0.12s;"/>
                  {:else}
                    <!-- main face -->
                    <path d={clockArc(h)} fill="#EDE8E1"/>
                    <!-- inner 3D cap — subtle shadow on inner edge -->
                    <path d={clockArcCap(h)} fill="#B8B2AA" opacity="0.10"/>
                  {/if}
                </g>
              {/each}
              </g>

              <!-- Hovered segment highlight -->
              {#if clockHoverH !== null}
                <path d={clockArc(clockHoverH)} fill="none" stroke="rgba(0,0,0,0.1)" stroke-width="1.5"/>
              {/if}

              <!-- Cardinal labels -->
              <text x="150" y="9"   text-anchor="middle" font-size="10" fill="#C8C2BA" font-family="Switzer,sans-serif">0h</text>
              <text x="295" y="154" text-anchor="start"  font-size="10" fill="#C8C2BA" font-family="Switzer,sans-serif">6h</text>
              <text x="150" y="297" text-anchor="middle" font-size="10" fill="#C8C2BA" font-family="Switzer,sans-serif">12h</text>
              <text x="5"   y="154" text-anchor="end"    font-size="10" fill="#C8C2BA" font-family="Switzer,sans-serif">18h</text>



              <!-- Depth rim: bevel rings only — scale faster on hover -->
              <g class="rim-depth-ring" class:rim-depth-ring-hovered={clockHoverH !== null}>
                <circle cx="150" cy="150" r="83" fill="none" stroke={clockHoverH !== null ? 'rgba(0,0,0,0.02)' : 'rgba(0,0,0,0.03)'} stroke-width="1.5" style="transition: stroke 0.2s ease;"/>
                <circle cx="150" cy="150" r="84.5" fill="none" stroke="rgba(255,255,255,0.55)" stroke-width="1"/>
              </g>

              <!-- Center face (inset well) — rendered before progress arc so arc appears on top -->
              <circle cx="150" cy="150" r="82" fill="white" filter="url(#clockCenterShadow)"/>

              <!-- Progress track + yellow arc — own group, scales more than bevel rings -->
              <g class="progress-ring" class:progress-ring-hovered={clockHoverH !== null}>
                <circle cx="150" cy="150" r="68" fill="none" stroke={clockHoverH !== null ? '#F8F5F2' : '#F4F0EC'} stroke-width="5" style="transition: stroke 0.2s ease;"/>
                {#if indepHours > 0}
                  {@const ir  = 68}
                  {@const ia1 = -Math.PI / 2}
                  {@const ia2 = ia1 + (indepHours / 24) * 2 * Math.PI}
                  {@const circ = 2 * Math.PI * ir}
                  {@const arcLen = (indepHours / 24) * circ}
                  <path
                    d={`M ${CCX + ir * Math.cos(ia1)} ${CCY + ir * Math.sin(ia1)} A ${ir} ${ir} 0 ${indepHours >= 12 ? 1 : 0} 1 ${CCX + ir * Math.cos(ia2)} ${CCY + ir * Math.sin(ia2)}`}
                    fill="none" stroke="#FFD000" stroke-width="5" stroke-linecap="round"
                    stroke-dasharray={circ}
                    stroke-dashoffset={clockMounted ? circ - arcLen : circ}
                    opacity={clockHoverH !== null ? 0 : 1}
                    style="transition: stroke-dashoffset 1.1s cubic-bezier(0.4,0,0.2,1) 500ms, opacity 0.2s ease;"/>
                {/if}
              </g>

              <!-- Center content -->
              {#if clockHoverH === null}
                <g class={clockMountAnimDone ? 'clock-unhover-fade' : 'clock-center-fade'}>
                  <text x="150" y="154" text-anchor="middle" font-family="Switzer,sans-serif">
                    <tspan font-size="52" font-weight="700" fill="#030303" letter-spacing="-2.5">{clockDisplayH}</tspan><tspan font-size="20" font-weight="600" fill="#8C8C8C" dy="-15" dx="3">h</tspan>
                  </text>
                  <text x="150" y="170" text-anchor="middle" font-size="10" fill="#C8C2BA" font-family="Switzer,sans-serif">unabhängig / Tag</text>
                </g>
              {:else}
                {@const bh = hourlyBars[clockHoverH]}
                {@const isIndepH   = bh.grid < 0.005}
                {@const isPartialH = bh.prod > 0.001 && !isIndepH}
                <!-- Time -->
                <text x="150" y="105" text-anchor="middle" font-size="10.5" font-weight="600" fill="#8C8480" letter-spacing="0.4" font-family="Switzer,sans-serif">{clockHoverH}:00 – {clockHoverH + 1}:00 Uhr</text>
                <line x1="114" y1="113" x2="186" y2="113" stroke="#E8E2DC" stroke-width="1"/>
                <!-- Solar label + value -->
                <text x="150" y="126" text-anchor="middle" font-size="9.5" fill="#A09890" letter-spacing="0.6" font-family="Switzer,sans-serif">SOLAR</text>
                <text x="150" y="143" text-anchor="middle" font-size="18" font-weight="700" fill="#111" letter-spacing="-0.8" font-family="Switzer,sans-serif">{bh.prod.toFixed(1)} kWh</text>
                <line x1="114" y1="150" x2="186" y2="150" stroke="#E8E2DC" stroke-width="1"/>
                <!-- Netz label + value -->
                <text x="150" y="163" text-anchor="middle" font-size="9.5" fill="#A09890" letter-spacing="0.6" font-family="Switzer,sans-serif">NETZ</text>
                <text x="150" y="180" text-anchor="middle" font-size="18" font-weight="700" fill="#111" letter-spacing="-0.8" font-family="Switzer,sans-serif">{bh.grid.toFixed(1)} kWh</text>
                <!-- Status pill -->
                <rect x="108" y="193" width="84" height="17" rx="8.5"
                  fill={isIndepH ? 'rgba(255,208,0,0.25)' : isPartialH ? 'rgba(255,208,0,0.15)' : 'rgba(0,0,0,0.06)'}/>
                <text x="150" y="205.5" text-anchor="middle" font-size="9.5" font-weight="600" font-family="Switzer,sans-serif"
                  fill={isIndepH ? '#7A3500' : isPartialH ? '#B85C00' : '#807870'}>
                  {isIndepH ? '✓ Netzunabhängig' : isPartialH ? '~ Teils solar' : '↑ Netzbezug'}
                </text>
              {/if}
            </svg>


            </div><!-- /clock-svg-wrapper -->
          </div><!-- /clock hero -->

          <!-- RIGHT: Sonnenpeak + Einspeisung -->
          <div class="stats-right flex-1 flex flex-col gap-6 justify-center self-stretch pr-1">

            <div class="flex flex-col gap-2">
              <p class="font-switzer text-[10px] font-medium text-[#B0AAA4] uppercase tracking-[1px]">Sonnenpeak</p>
              <div class="flex items-baseline gap-1 leading-none">
                <span class="font-switzer font-bold text-[44px] text-[#111] leading-none tracking-[-2.5px]">{peakSolarH}</span>
                <span class="font-switzer font-semibold text-[16px] text-[#C8C2BA] leading-none">:00 Uhr</span>
              </div>
              <div class="relative h-[4px] rounded-full" style="background:linear-gradient(to right, #EDE8E1 0%, #FFD000 {peakSolarH/24*100}%, #EDE8E1 100%);">
                <div class="absolute top-1/2 w-[9px] h-[9px] rounded-full border-[2px] border-white"
                  style="left:{peakSolarH/24*100}%; transform:translate(-50%,-50%); background:#FFD000; box-shadow:0 0 0 2px rgba(255,208,0,0.3);"/>
              </div>
              <p class="font-switzer text-[11px] text-[#C8C2BA]">optimale Lastzeit</p>
            </div>

            <div class="h-px" style="background:#F0EDE8;"></div>

            <div class="flex flex-col gap-2">
              <p class="font-switzer text-[10px] font-medium text-[#B0AAA4] uppercase tracking-[1px]">Einspeisung</p>
              <div class="flex items-baseline gap-1 leading-none">
                <span class="font-switzer font-bold text-[44px] text-[#111] leading-none tracking-[-2.5px]">{fmt(einspeisKwh)}</span>
                <span class="font-switzer font-semibold text-[16px] text-[#C8C2BA] leading-none">kWh</span>
              </div>
              <p class="font-switzer text-[11px] text-[#C8C2BA]">zurück ins Netz / Jahr</p>
            </div>

          </div><!-- /right stats -->

        </div><!-- /three-column layout -->

      </div><!-- /combined card -->

    </div>

    <!-- ══ Charts Row 2: 3 Cards ══ -->
    <div class="grid grid-cols-3 gap-4">

      <!-- ── Finanzierungsrechner ── -->
      <div class="rounded-[16px] col-span-1 flex overflow-hidden" style="background:white; border:1px solid #ede8e0;">

        <!-- Left: KPI + Controls -->
        <div class="flex flex-col justify-between p-5 gap-5" style="width:46%; border-right:1px solid #f0ece6; min-width:0;">

          <!-- Top: title + KPI -->
          <div class="flex flex-col gap-5">
            <!-- Title + toggle -->
            <div>
              <p class="font-switzer font-semibold text-[17px] text-[#030303] leading-none mb-0.5">Finanzierung</p>
              <p class="font-switzer text-[12px] text-[#B0A89E]">nach Monatsrate</p>
              <div class="flex gap-1 mt-2">
                {#each loanTypeOptions as t}
                  <button class="font-switzer text-[12px] font-medium rounded-full px-3 py-1 transition-all duration-150"
                    style="background:{loanType === t.k ? '#f0ece6' : 'transparent'}; color:{loanType === t.k ? '#030303' : '#C0B8B0'}; border:1px solid {loanType === t.k ? '#ddd6cc' : 'transparent'}; cursor:pointer;"
                    on:click={() => setLoanType(t.k)}
                  >{t.l}</button>
                {/each}
              </div>
            </div>

            <!-- KPI number -->
            <div>
              <p class="font-switzer text-[10px] text-[#B0A89E] uppercase tracking-widest mb-1">
                {mHoverIdx !== null ? MONTH_FULL[mHoverIdx] : 'Ø Monat'}
              </p>
              <p class="font-switzer font-semibold leading-none tracking-[-2px] text-[#030303]" style="font-size:clamp(26px,3.5vw,36px);">
                {hoverNet >= 0 ? '+' : '−'}CHF {fmt(Math.abs(hoverNet))}
              </p>
              <span class="font-switzer text-[12px] font-semibold mt-1 inline-block" style="color:{hoverNet >= 0 ? '#02B597' : '#C0A898'};">
                {hoverNet >= 0 ? '↑ Einsparung' : '↓ Investition'}
              </span>
              <div class="flex flex-col gap-1.5 mt-3">
                <div class="flex items-center justify-between">
                  <div class="flex items-center gap-1.5">
                    <div class="w-[5px] h-[5px] rounded-full shrink-0" style="background:#02B597;"/>
                    <span class="font-switzer text-[11px] text-[#8C8C8C]">Strom gespart</span>
                  </div>
                  <span class="font-switzer text-[11px] font-semibold text-[#02B597]">+CHF {fmt(hoverSaving)}</span>
                </div>
                <div class="flex items-center justify-between">
                  <div class="flex items-center gap-1.5">
                    <div class="w-[5px] h-[5px] rounded-full shrink-0" style="background:#D4CEC8;"/>
                    <span class="font-switzer text-[11px] text-[#8C8C8C]">Monatsrate</span>
                  </div>
                  <span class="font-switzer text-[11px] font-semibold text-[#8C8C8C]">−CHF {fmt(monthlyPayment)}</span>
                </div>
              </div>
            </div>
          </div>

          <!-- Bottom: sliders -->
          <div class="flex flex-col gap-2.5 pt-4 border-t border-[#f4f0eb]">
            <div class="flex flex-col gap-1">
              <div class="flex items-center justify-between">
                <span class="font-switzer text-[10px] uppercase tracking-widest text-[#C0B8B0]">Zinssatz</span>
                <span class="font-switzer text-[12px] font-semibold text-[#030303]">{loanRate}%</span>
              </div>
              <input type="range" min={RATE_MIN} max={RATE_MAX} step="0.25" bind:value={loanRate}
                class="w-full h-[2px] rounded-full appearance-none cursor-pointer"
                style="background:linear-gradient(to right, #030303 {rateFillPct}%, #e5e1db {rateFillPct}%); accent-color:#030303;"/>
            </div>
            <div class="flex flex-col gap-1">
              <div class="flex items-center justify-between">
                <span class="font-switzer text-[10px] uppercase tracking-widest text-[#C0B8B0]">Laufzeit</span>
                <span class="font-switzer text-[12px] font-semibold text-[#030303]">{loanYears}J</span>
              </div>
              <input type="range" min={YEAR_MIN} max={YEAR_MAX} step="1" bind:value={loanYears}
                class="w-full h-[2px] rounded-full appearance-none cursor-pointer"
                style="background:linear-gradient(to right, #030303 {yearFillPct}%, #e5e1db {yearFillPct}%); accent-color:#030303;"/>
            </div>
          </div>
        </div>

        <!-- Right: Bar chart -->
        <div class="flex-1 flex flex-col p-4" style="background:#FAFAF8; min-width:0;">
          <div class="flex-1 relative flex items-end" style="gap:3px; min-height:0;">
            <!-- Payment reference line -->
            <div class="absolute left-0 right-0 pointer-events-none flex items-center gap-1"
              style="bottom:{paymentLinePct}%;">
              <div class="flex-1" style="border-top:1px dashed rgba(180,170,160,0.5);"/>
            </div>
            <!-- Bars -->
            {#each monthlyData as d, i}
              {@const hPct = mMaxCost > 0 ? Math.round((d.saving / mMaxCost) * 75) : 0}
              {@const covers = d.saving >= monthlyPayment}
              {@const isActive = mHoverIdx === null || mHoverIdx === i}
              <div class="flex-1 flex flex-col justify-end cursor-pointer" role="presentation"
                style="height:100%;"
                on:mouseenter={() => mHoverIdx = i}
                on:mouseleave={() => mHoverIdx = null}
              >
                <div class="w-full transition-all duration-200"
                  style="height:{hPct}%; border-radius:4px 4px 0 0; opacity:{isActive ? 1 : 0.15}; background:{covers
                    ? 'linear-gradient(to bottom, #02B597 0%, rgba(2,181,151,0.12) 100%)'
                    : 'linear-gradient(to bottom, rgba(180,168,156,0.7) 0%, rgba(180,168,156,0.05) 100%)'};"
                />
              </div>
            {/each}
          </div>
          <!-- Month labels -->
          <div class="flex mt-2" style="gap:3px;">
            {#each monthlyData as d, i}
              <div class="flex-1 text-center font-switzer transition-all duration-150"
                style="font-size:8px; color:{mHoverIdx === i ? '#030303' : '#C8C2BC'}; font-weight:{mHoverIdx === i ? 700 : 400};">
                {d.label}
              </div>
            {/each}
          </div>
        </div>
      </div>

      <!-- ── Line chart: Kostenvergleich ── -->
      <div class="col-span-2 rounded-[16px] p-6 flex flex-col gap-5" style="background:white; border:1px solid #ede8e0;">
        <!-- Header -->
        <div class="flex items-start justify-between">
          <div>
            <h3 class="font-switzer font-semibold text-[17px] text-[#030303] tracking-[-0.4px]">Kostenvergleich</h3>
            <p class="font-switzer text-[12px] text-[#8C8C8C] mt-0.5">Kumulierte Stromkosten über {installationLifeSpan} Jahre</p>
          </div>
          <div class="flex items-center gap-2">
            <div class="flex items-center gap-1.5 rounded-full px-3 py-1" style="background:rgba(2,181,151,0.08); border:1px solid rgba(2,181,151,0.3);">
              <div class="w-2 h-2 rounded-full" style="background:#02B597"/>
              <span class="font-switzer font-semibold text-[11px]" style="color:#047857;">Mit Solar</span>
            </div>
            <div class="flex items-center gap-1.5 rounded-full px-3 py-1" style="background:#fff1f1; border:1px solid #fca5a5;">
              <div class="w-2 h-2 rounded-full" style="background:#F87171"/>
              <span class="font-switzer font-semibold text-[11px]" style="color:#b91c1c;">Ohne Solar</span>
            </div>
          </div>
        </div>
        <!-- Chart -->
        <svg viewBox="0 0 {LW} {LH}" class="w-full" style="overflow:visible; cursor:crosshair;"
          on:mousemove={handleLineMove} on:mouseleave={() => lHoverYear=null}>
          <defs>
            <linearGradient id="gradSolar" x1="0" y1="0" x2="0" y2="1">
              <stop offset="0%" stop-color="#FFE572" stop-opacity="0.5"/>
              <stop offset="100%" stop-color="#FFE572" stop-opacity="0.01"/>
            </linearGradient>
            <linearGradient id="gradNoSolar" x1="0" y1="0" x2="0" y2="1">
              <stop offset="0%" stop-color="#F87171" stop-opacity="0.15"/>
              <stop offset="100%" stop-color="#F87171" stop-opacity="0.02"/>
            </linearGradient>
          </defs>
          <!-- Grid -->
          {#each lYLab as {val,y}}
            <line x1={LP.l} y1={y} x2={LW-LP.r} y2={y} stroke="#f4f4f4" stroke-width="1"/>
            <text x={LP.l-5} y={y+4} text-anchor="end" font-size="10" fill="#bbb" font-family="Switzer,sans-serif">
              {fmt(val)}
            </text>
          {/each}
          <!-- Fills -->
          <path d={lFillN} fill="url(#gradNoSolar)"/>
          <!-- Lines -->
          <path d={lPathN} fill="none" stroke="#F87171" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" opacity="0.8"/>
          <path d={lPathS} fill="none" stroke="#02B597" stroke-width="3" stroke-linecap="round" stroke-linejoin="round"/>
          <!-- Amortisation marker -->
          {#if breakEvenYear>=0}
            {@const beCalYear = new Date().getFullYear() + breakEvenYear}
            <!-- Connector line vom Dot zum Badge oben -->
            <line x1={lBEX} y1={lBEY - 8} x2={lBEX} y2={LP.t + 36} stroke="#ccc" stroke-width="1" stroke-dasharray="3 3" style="opacity:{lHoverYear===null?1:0}; transition:opacity 0.25s;"/>
            <!-- Dot — prominent -->
            <circle cx={lBEX} cy={lBEY} r="6" fill="white" stroke="#555" stroke-width="1.5"/>
            <circle cx={lBEX} cy={lBEY} r="2.5" fill="#333"/>
            <!-- Badge oben, über der Linie -->
            {@const labelW2 = 100}
            {@const labelX2 = Math.min(Math.max(lBEX - labelW2/2, LP.l), LW - LP.r - labelW2)}
            <g style="opacity:{lHoverYear===null?1:0}; transition:opacity 0.25s ease;">
              <rect x={labelX2} y={LP.t} width={labelW2} height="26" rx="13" fill="#222"/>
              <text x={labelX2 + labelW2/2} y={LP.t + 10} text-anchor="middle" font-size="7.5" fill="rgba(255,255,255,0.45)" font-family="Switzer,sans-serif" letter-spacing="0.6">AMORTISIERT</text>
              <text x={labelX2 + labelW2/2} y={LP.t + 21} text-anchor="middle" font-size="12" font-weight="700" fill="white" letter-spacing="-0.3" font-family="Switzer,sans-serif">Ab {beCalYear}</text>
            </g>
          {/if}
          <!-- End-of-chart Gap Annotation -->
          {#if cumWithSolar.length > 0 && cumWithoutSolar.length > 0}
            {@const endS = cumWithSolar[cumWithSolar.length-1]}
            {@const endN = cumWithoutSolar[cumWithoutSolar.length-1]}
            {@const endX = lpx(installationLifeSpan-1)}
            {@const endYS = lpy(endS)}
            {@const endYN = lpy(endN)}
            {@const gapMid = (endYS + endYN) / 2}
            {#if endN - endS > 1000}
              <line x1={endX-8} y1={endYS} x2={endX-8} y2={endYN} stroke="#ddd" stroke-width="1"/>
              <line x1={endX-13} y1={endYS} x2={endX-3} y2={endYS} stroke="#ddd" stroke-width="1"/>
              <line x1={endX-13} y1={endYN} x2={endX-3} y2={endYN} stroke="#ddd" stroke-width="1"/>
              <text x={endX-14} y={gapMid-5} text-anchor="end" font-size="11" font-weight="600" fill="#555" font-family="Switzer,sans-serif">CHF {fmt(endN-endS)}</text>
              <text x={endX-14} y={gapMid+8} text-anchor="end" font-size="10" fill="#aaa" font-family="Switzer,sans-serif">gespart</text>
            {/if}
          {/if}
          <!-- X axis labels -->
          <line x1={LP.l} y1={LP.t+lcH} x2={LW-LP.r} y2={LP.t+lcH} stroke="#eeeeee" stroke-width="1"/>
          {#each lXLab as {year,x}}
            <text x={x} y={LH-8} text-anchor="middle" font-size="10" fill="#bbb" font-family="Switzer,sans-serif">{year}</text>
          {/each}
          <!-- Hover crosshair + tooltip -->
          {#if lHoverYear !== null}
            {@const diff = lHoverValN - lHoverValS}
            {@const tipW = 210}
            {@const tipH = 96}
            <line x1={lHoverX} y1={LP.t} x2={lHoverX} y2={LP.t+lcH} stroke="#030303" stroke-width="1" stroke-dasharray="4 3" opacity="0.15"/>
            {#if !lHoverNearBE}
              <circle cx={lHoverX} cy={lHoverYN} r="5" fill="white" stroke="#F87171" stroke-width="2.5"/>
              <circle cx={lHoverX} cy={lHoverYS} r="5" fill="white" stroke="#02B597" stroke-width="2.5"/>
            {/if}
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
                <circle cx="24" cy="44" r="4" fill="#02B597"/>
                <text x="34" y="48" font-size="12" fill="#666" font-family="Switzer,sans-serif">Mit Solar</text>
                <text x={tipW-16} y="48" text-anchor="end" font-size="13" font-weight="700" fill="white" font-family="Switzer,sans-serif">CHF {fmt(lHoverValS)}</text>
                <!-- Ohne Solar -->
                <circle cx="24" cy="66" r="4" fill="#F87171"/>
                <text x="34" y="70" font-size="12" fill="#666" font-family="Switzer,sans-serif">Ohne Solar</text>
                <text x={tipW-16} y="70" text-anchor="end" font-size="13" font-weight="700" fill="white" font-family="Switzer,sans-serif">CHF {fmt(lHoverValN)}</text>
              </g>
            </g>
          {/if}
        </svg>
      </div>

      <!-- ── KPI: Installationsdauer ── -->
      <div class="col-span-2 w-full rounded-[16px] px-6 py-5 flex flex-col justify-between" style="background:white; border:1px solid #ede8e0;">
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
      <!-- ── PARTNER WIDGET ── -->
      <div class="col-span-1 w-full rounded-[16px] p-6 flex flex-col justify-between relative overflow-hidden" style="background:#ffffff; border:1px solid #e8e2da;">

        <div class="flex flex-col h-full justify-between gap-5">
          <!-- Header -->
          <div>
            <div class="inline-flex items-center gap-1.5 rounded-full px-3 py-1 mb-4" style="background:#f0fdf4; border:1px solid #bbf7d0;">
              <div class="w-1.5 h-1.5 rounded-full bg-[#16a34a]"></div>
              <p class="font-switzer font-semibold text-[11px] text-[#15803d] uppercase tracking-wider">Ihr Partner auf 25 Jahre</p>
            </div>

            <p class="font-switzer font-extrabold text-[26px] text-[#1a1a1a] leading-[1.1] tracking-tight mb-2">Wir sind auch<br>in Zukunft für<br>Sie da.</p>
            <p class="font-switzer text-[13px] text-[#6b7280] leading-[1.6]">Ihre Anlage wächst mit Ihren Bedürfnissen — wir begleiten Sie dabei.</p>
          </div>

          <!-- Services -->
          <div class="flex flex-col gap-2.5">
            <div class="flex items-center gap-3 rounded-[12px] px-3.5 py-3" style="background:#f6f1eb; border:1px solid #e8e2da;">
              <div class="w-7 h-7 rounded-[8px] bg-white flex items-center justify-center shrink-0" style="border:1px solid #e8e2da;">
                <svg class="w-3.5 h-3.5 text-[#02B597]" fill="none" viewBox="0 0 24 24" stroke="currentColor"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9.663 17h4.673M12 3v1m6.364 1.636l-.707.707M21 12h-1M4 12H3m3.343-5.657l-.707-.707m2.828 9.9a5 5 0 117.072 0l-.548.547A3.374 3.374 0 0014 18.469V19a2 2 0 11-4 0v-.531c0-.895-.356-1.754-.988-2.386l-.548-.547z"/></svg>
              </div>
              <div>
                <p class="font-switzer font-semibold text-[13px] text-[#1a1a1a]">Energieberatung</p>
                <p class="font-switzer text-[11px] text-[#9ca3af]">Optimierung & Erweiterung</p>
              </div>
            </div>

            <div class="flex items-center gap-3 rounded-[12px] px-3.5 py-3" style="background:#f6f1eb; border:1px solid #e8e2da;">
              <div class="w-7 h-7 rounded-[8px] bg-white flex items-center justify-center shrink-0" style="border:1px solid #e8e2da;">
                <svg class="w-3.5 h-3.5 text-[#02B597]" fill="none" viewBox="0 0 24 24" stroke="currentColor"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 6V4m0 2a2 2 0 100 4m0-4a2 2 0 110 4m-6 8a2 2 0 100-4m0 4a2 2 0 110-4m0 4v2m0-6V4m6 6v10m6-2a2 2 0 100-4m0 4a2 2 0 110-4m0 4v2m0-6V4"/></svg>
              </div>
              <div>
                <p class="font-switzer font-semibold text-[13px] text-[#1a1a1a]">Smart Management</p>
                <p class="font-switzer text-[11px] text-[#9ca3af]">Monitoring & Steuerung</p>
              </div>
            </div>

            <div class="flex items-center gap-3 rounded-[12px] px-3.5 py-3" style="background:#f6f1eb; border:1px solid #e8e2da;">
              <div class="w-7 h-7 rounded-[8px] bg-white flex items-center justify-center shrink-0" style="border:1px solid #e8e2da;">
                <svg class="w-3.5 h-3.5 text-[#02B597]" fill="none" viewBox="0 0 24 24" stroke="currentColor"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M14.7 6.3a1 1 0 0 0 0 1.4l1.6 1.6a1 1 0 0 0 1.4 0l3.77-3.77a6 6 0 0 1-7.94 7.94l-6.91 6.91a2.12 2.12 0 0 1-3-3l6.91-6.91a6 6 0 0 1 7.94-7.94l-3.76 3.76z"/></svg>
              </div>
              <div>
                <p class="font-switzer font-semibold text-[13px] text-[#1a1a1a]">Service & Wartung</p>
                <p class="font-switzer text-[11px] text-[#9ca3af]">Langfristig zuverlässig</p>
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
