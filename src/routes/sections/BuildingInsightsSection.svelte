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

  import type { MdDialog } from '@material/web/dialog/dialog';
  import Expandable from '../components/Expandable.svelte';
  import {
    type BuildingInsightsResponse,
    type RequestError,
    findClosestBuilding,
    getDataLayerUrls,
    type SolarPanelConfig,
    type DataLayersResponse,
    type LayerId,
  } from '../solar';
  import { getLayer, type Layer } from '../layer';
  import Show from '../components/Show.svelte';
  import SummaryCard from '../components/SummaryCard.svelte';
  import { createPalette, normalize, rgbToColor } from '../visualize';
  import { panelsPalette } from '../colors';
  import InputBool from '../components/InputBool.svelte';
  import InputPanelsCount from '../components/InputPanelsCount.svelte';
  import { showNumber } from '../utils';
  import NumberInput from '../components/InputNumber.svelte';
  import Gauge from '../components/Gauge.svelte';

  export let expandedSection: string;
  export let buildingInsights: BuildingInsightsResponse | undefined;
  export let configId: number | undefined;
  export let panelCapacityWatts: number;
  export let showPanels: boolean;

  export let googleMapsApiKey: string;
  export let geometryLibrary: google.maps.GeometryLibrary;
  export let location: google.maps.LatLng;
  export let map: google.maps.Map;

  const icon = 'home';
  const title = 'Gebäude Parameter';

  let requestSent = false;
  let requestError: RequestError | undefined;
  let apiResponseDialog: MdDialog;

  let panelConfig: SolarPanelConfig | undefined;
  $: if (buildingInsights && configId !== undefined) {
    panelConfig = buildingInsights.solarPotential.solarPanelConfigs[configId];
  }

  let solarPanels: google.maps.Polygon[] = [];
  $: solarPanels.map((panel, i) =>
    panel.setMap(showPanels && panelConfig && i < panelConfig.panelsCount ? map : null),
  );

  let panelCapacityRatio = 1.0;
  $: if (buildingInsights) {
    const defaultPanelCapacity = buildingInsights.solarPotential.panelCapacityWatts;
    panelCapacityRatio = panelCapacityWatts / defaultPanelCapacity;
  }

  export async function showSolarPotential(location: google.maps.LatLng) {
    if (requestSent) {
      return;
    }

    console.log('showSolarPotential');
    buildingInsights = undefined;
    requestError = undefined;

    solarPanels.map((panel) => panel.setMap(null));
    solarPanels = [];

    requestSent = true;
    try {
      buildingInsights = await findClosestBuilding(location, googleMapsApiKey);
    } catch (e) {
      requestError = e as RequestError;
      return;
    } finally {
      requestSent = false;
    }

    // Create the solar panels on the map.
    const solarPotential = buildingInsights.solarPotential;
    const palette = createPalette(panelsPalette).map(rgbToColor);
    const minEnergy = solarPotential.solarPanels.slice(-1)[0].yearlyEnergyDcKwh;
    const maxEnergy = solarPotential.solarPanels[0].yearlyEnergyDcKwh;
    solarPanels = solarPotential.solarPanels.map((panel) => {
      const [w, h] = [solarPotential.panelWidthMeters / 2, solarPotential.panelHeightMeters / 2];
      const points = [
        { x: +w, y: +h }, // top right
        { x: +w, y: -h }, // bottom right
        { x: -w, y: -h }, // bottom left
        { x: -w, y: +h }, // top left
        { x: +w, y: +h }, //  top right
      ];
      const orientation = panel.orientation == 'PORTRAIT' ? 90 : 0;
      const azimuth = solarPotential.roofSegmentStats[panel.segmentIndex].azimuthDegrees;
      const colorIndex = Math.round(normalize(panel.yearlyEnergyDcKwh, maxEnergy, minEnergy) * 255);
      return new google.maps.Polygon({
        paths: points.map(({ x, y }) =>
          geometryLibrary.spherical.computeOffset(
            { lat: panel.center.latitude, lng: panel.center.longitude },
            Math.sqrt(x * x + y * y),
            Math.atan2(y, x) * (180 / Math.PI) + orientation + azimuth,
          ),
        ),
        strokeColor: '#B0BEC5',
        strokeOpacity: 0.9,
        strokeWeight: 1,
        fillColor: palette[colorIndex],
        fillOpacity: 0.9,
      });
    });
  }

  // DataLayer
  const dataLayerOptions: Record<LayerId | 'none', string> = {
    none: 'No layer',
    mask: 'Roof mask',
    dsm: 'Digital Surface Model',
    rgb: 'Aerial image',
    annualFlux: 'Jährliche Sonnenstunden',
    monthlyFlux: 'Monatliche Sonnenstunden',
    hourlyShade: 'Stündlicher Schatten',
  };

  const monthNames = [
    'Jan',
    'Feb',
    'Mar',
    'Apr',
    'May',
    'Jun',
    'Jul',
    'Aug',
    'Sep',
    'Oct',
    'Nov',
    'Dec',
  ];

  let dataLayersResponse: DataLayersResponse | undefined;
  let layerId: LayerId | 'none' = 'monthlyFlux';
  let layer: Layer | undefined;
  let imageryQuality: 'HIGH' | 'MEDIUM' | 'LOW';

  let playAnimation = false;
  let tick = 0;
  let month = 0;
  let day = 14;
  let hour = 0;

  let overlays: google.maps.GroundOverlay[] = [];
  let showRoofOnly = false;
  async function showDataLayer(reset = false) {
    if (reset) {
      dataLayersResponse = undefined;
      requestError = undefined;
      layer = undefined;

      // Default values per layer.
      showRoofOnly = ['annualFlux', 'monthlyFlux', 'hourlyShade'].includes(layerId);
      map.setMapTypeId(layerId == 'rgb' ? 'roadmap' : 'satellite');
      overlays.map((overlay) => overlay.setMap(null));
      month = layerId == 'hourlyShade' ? 3 : 0;
      day = 14;
      hour = 5;
      playAnimation = ['monthlyFlux', 'hourlyShade'].includes(layerId);
    }
    if (layerId == 'none') {
      return;
    }

    if (!layer) {
      const center = buildingInsights.center;
      const ne = buildingInsights.boundingBox.ne;
      const sw = buildingInsights.boundingBox.sw;
      const diameter = geometryLibrary.spherical.computeDistanceBetween(
        new google.maps.LatLng(ne.latitude, ne.longitude),
        new google.maps.LatLng(sw.latitude, sw.longitude),
      );
      const radius = Math.ceil(diameter / 2);
      try {
        dataLayersResponse = await getDataLayerUrls(center, radius, googleMapsApiKey);
      } catch (e) {
        requestError = e as RequestError;
        return;
      }

      imageryQuality = dataLayersResponse.imageryQuality;

      try {
        layer = await getLayer(layerId, dataLayersResponse, googleMapsApiKey);
      } catch (e) {
        requestError = e as RequestError;
        return;
      }
    }

    const bounds = layer.bounds;
    console.log('Render layer:', {
      layerId: layer.id,
      showRoofOnly: showRoofOnly,
      month: month,
      day: day,
    });
    overlays.map((overlay) => overlay.setMap(null));
    overlays = layer
      .render(showRoofOnly, month, day)
      .map((canvas) => new google.maps.GroundOverlay(canvas.toDataURL(), bounds));

    if (!['monthlyFlux', 'hourlyShade'].includes(layer.id)) {
      overlays[0].setMap(map);
    }
  }

  $: if (layer?.id == 'monthlyFlux') {
    overlays.map((overlay, i) => overlay.setMap(i == month ? map : null));
  } else if (layer?.id == 'hourlyShade') {
    overlays.map((overlay, i) => overlay.setMap(i == hour ? map : null));
  }

  function onSliderChange(event: Event) {
    const target = event.target as MdSlider;
    if (layer?.id == 'monthlyFlux') {
      if (target.valueStart != month) {
        month = target.valueStart ?? 0;
      } else if (target.valueEnd != month) {
        month = target.valueEnd ?? 0;
      }
      tick = month;
    } else if (layer?.id == 'hourlyShade') {
      if (target.valueStart != hour) {
        hour = target.valueStart ?? 0;
      } else if (target.valueEnd != hour) {
        hour = target.valueEnd ?? 0;
      }
      tick = hour;
    }
  }

  $: if (layer?.id == 'monthlyFlux') {
    if (playAnimation) {
      month = tick % 12;
    } else {
      tick = month;
    }
  } else if (layer?.id == 'hourlyShade') {
    if (playAnimation) {
      hour = tick % 24;
    } else {
      tick = hour;
    }
  }

  onMount(() => {
    showDataLayer(true);

    setInterval(() => {
      tick++;
    }, 1000);
  });

  $: showSolarPotential(location);
</script>

{#if requestError}
  <div class="error-container on-error-container-text">
    <Expandable section={title} icon="error" {title} subtitle={requestError.error.status}>
      <div class="grid place-items-center py-2 space-y-4">
        <div class="grid place-items-center">
          <p class="body-medium">
            Error on <code>buildingInsights</code> request
          </p>
          <p class="title-large">ERROR {requestError.error.code}</p>
          <p class="body-medium"><code>{requestError.error.status}</code></p>
          <p class="label-medium">{requestError.error.message}</p>
        </div>
        <md-filled-button role={undefined} on:click={() => showSolarPotential(location)}>
          Retry
          <md-icon slot="icon">refresh</md-icon>
        </md-filled-button>
      </div>
    </Expandable>
  </div>
{:else if !buildingInsights}
  <div class="grid py-8 place-items-center">
    <md-circular-progress four-color indeterminate />
  </div>
{:else if configId !== undefined && panelConfig}
  <Expandable
    bind:section={expandedSection}
    {icon}
    {title}
    subtitle={`Energie pro Jahr: ${(
      (panelConfig.yearlyEnergyDcKwh * panelCapacityRatio) /
      1000
    ).toFixed(2)} MWh`}
  >
    <div class="flex flex-col space-y-2 px-2">
      <span class="outline-text label-medium">
        <b>{title}</b> Gibt Auskunft über Standort, Dachfläche und Eignung für Solaranlagen.
      </span>

      {#if layer.palette}
        <div>
          <div
            class="h-2 outline rounded-sm"
            style={`background: linear-gradient(to right, ${layer.palette.colors.map(
              (hex) => '#' + hex,
            )})`}
          />
          <div class="flex justify-between pt-1 label-small">
            <span>{layer.palette.min}</span>
            <span>{layer.palette.max}</span>
          </div>
        </div>
      {/if}

      <InputPanelsCount
        bind:configId
        solarPanelConfigs={buildingInsights.solarPotential.solarPanelConfigs}
      />
      <NumberInput
        bind:value={panelCapacityWatts}
        icon="bolt"
        label="Panel Kapazität"
        suffix="Watt"
      />
      <InputBool bind:value={showPanels} label="Solar panels" />

      <div class="grid justify-items-end">
        <md-filled-tonal-button role={undefined} on:click={() => apiResponseDialog.show()}>
          API response
        </md-filled-tonal-button>
      </div>

      <md-dialog bind:this={apiResponseDialog}>
        <div slot="headline">
          <div class="flex items-center primary-text">
            <md-icon>{icon}</md-icon>
            <b>&nbsp;{title}</b>
          </div>
        </div>
        <div slot="content">
          <Show value={buildingInsights} label="buildingInsightsResponse" />
        </div>
        <div slot="actions">
          <md-text-button role={undefined} on:click={() => apiResponseDialog.close()}>
            Close
          </md-text-button>
        </div>
      </md-dialog>
    </div>
  </Expandable>

  {#if expandedSection == title}
    <div class="absolute top-0 left-0 w-72">
      <div class="flex flex-col space-y-2 m-2">
        <SummaryCard
          {icon}
          {title}
          rows={[
            {
              icon: 'wb_sunny',
              name: 'Jährliche Sonnenstunden',
              value: showNumber(buildingInsights.solarPotential.maxSunshineHoursPerYear),
              units: 'h',
            },
            {
              icon: 'square_foot',
              name: 'Dachfläche',
              value: showNumber(buildingInsights.solarPotential.wholeRoofStats.areaMeters2),
              units: 'm²',
            },
            {
              icon: 'solar_power',
              name: 'Max Panel Anzahl',
              value: showNumber(buildingInsights.solarPotential.solarPanels.length),
              units: '',
            },
            {
              icon: 'co2',
              name: 'CO₂ Einsparung',
              value: showNumber(buildingInsights.solarPotential.carbonOffsetFactorKgPerMwh),
              units: 'Kg/MWh',
            },
          ]}
        />

        <div class="p-4 w-full surface on-surface-text rounded-lg shadow-md">
          <div class="flex justify-around">
            <Gauge
              icon="solar_power"
              title="Panel Anzahl"
              label={showNumber(panelConfig.panelsCount)}
              labelSuffix={`/ ${showNumber(solarPanels.length)}`}
              max={solarPanels.length}
              value={panelConfig.panelsCount}
            />

            <Gauge
              icon="energy_savings_leaf"
              title="KWh pro Jahr"
              label={showNumber((panelConfig?.yearlyEnergyDcKwh ?? 0) * panelCapacityRatio)}
              labelSuffix="KWh"
              max={buildingInsights.solarPotential.solarPanelConfigs.slice(-1)[0]
                .yearlyEnergyDcKwh * panelCapacityRatio}
              value={panelConfig.yearlyEnergyDcKwh * panelCapacityRatio}
            />
          </div>
        </div>
      </div>
    </div>
  {/if}
{/if}
