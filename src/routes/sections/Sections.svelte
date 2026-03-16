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

  import type { BuildingInsightsResponse } from '../solar';
  import { findSolarConfig } from '../utils';
  import BuildingInsightsSection from './BuildingInsightsSection.svelte';
  import DataLayersSection from './DataLayersSection.svelte';
  import SolarPotentialSection from './SolarPotentialSection.svelte';

  export let location: google.maps.LatLng;
  export let map: google.maps.Map;
  export let geometryLibrary: google.maps.GeometryLibrary;
  export let googleMapsApiKey: string;

  export let buildingInsights: BuildingInsightsResponse | undefined = undefined;

  // State
  let expandedSection: string = '';
  export let showPanels = true;
  export let showHeatmap = true;

  // User settings
  export let monthlyAverageEnergyBillInput = 300;
  export let panelCapacityWattsInput = 250;
  export let energyCostPerKwhInput = 0.31;
  export let dcToAcDerateInput = 0.85;

  // Find the config that covers the yearly energy consumption.
  let yearlyKwhEnergyConsumption: number;
  $: yearlyKwhEnergyConsumption = (monthlyAverageEnergyBillInput / energyCostPerKwhInput) * 12;

  export let configId: number | undefined = undefined;
  $: if (configId === undefined && buildingInsights) {
    const defaultPanelCapacity = buildingInsights.solarPotential.panelCapacityWatts;
    const panelCapacityRatio = panelCapacityWattsInput / defaultPanelCapacity;
    configId = findSolarConfig(
      buildingInsights.solarPotential.solarPanelConfigs,
      yearlyKwhEnergyConsumption,
      panelCapacityRatio,
      dcToAcDerateInput,
    );
  }
</script>
<div class="flex flex-col rounded-md shadow-md">
  {#if geometryLibrary && map}
    <BuildingInsightsSection
      bind:expandedSection
      bind:buildingInsights
      bind:configId
      bind:showPanels
      bind:showHeatmap
      bind:panelCapacityWatts={panelCapacityWattsInput}
      {googleMapsApiKey}
      {geometryLibrary}
      {location}
      {map}
    />
  {/if}

  {#if buildingInsights && configId !== undefined}
    <md-divider inset />
    <DataLayersSection
      bind:expandedSection
      bind:showPanels
      bind:showHeatmap
      {googleMapsApiKey}
      {buildingInsights}
      {geometryLibrary}
      {map}
    />

    <md-divider inset />
    <SolarPotentialSection
      bind:expandedSection
      bind:configId
      bind:monthlyAverageEnergyBillInput
      bind:energyCostPerKwhInput
      bind:panelCapacityWattsInput
      bind:dcToAcDerateInput
      solarPanelConfigs={buildingInsights.solarPotential.solarPanelConfigs}
      defaultPanelCapacityWatts={buildingInsights.solarPotential.panelCapacityWatts}
    />
  {/if}

</div>
