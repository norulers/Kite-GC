<!--
  SPDX-License-Identifier: GPL-3.0-or-later
  Copyright (C) 2026 Marc Hoffmann (b14ckyy)
-->

<!-- Chart plotter dashboard — Kip-inspired instrument column beside the map while the chart plotter
     mode is on (toolbar toggle). Wind rose on top, a one-minute speed trend in the middle, and a
     bottom row with the mission card (load + execute) next to the square Speed widget
     (throttle/acceleration bars). Owns the one-minute history so the trend chart has its data. -->
<script lang="ts">
  import { t } from 'svelte-i18n';
  import type { TelemetryData } from '$lib/stores/telemetry';
  import type { InterfaceSettings } from '$lib/stores/settings';
  import SpeedWidget from '$lib/components/widgets/SpeedWidget.svelte';
  import WindRoseCard from './WindRoseCard.svelte';
  import SpeedTrendCard from './SpeedTrendCard.svelte';
  import MissionCard from './MissionCard.svelte';

  let {
    telem,
    interfaceSettings,
  }: {
    telem: TelemetryData;
    interfaceSettings: InterfaceSettings;
  } = $props();

  const HISTORY_MS = 60_000;
  const STALE_GAP_MS = 5_000;
  const PUBLISH_MS = 400;

  // Plain (non-reactive) ring buffer — the effect only publishes snapshots of it, so it never reacts
  // to its own writes. The 400 ms publish cadence is plenty for a 1-minute trace.
  let samples: { t: number; v: number }[] = [];
  let history = $state<{ t: number; v: number }[]>([]);
  let lastPublish = 0;

  $effect(() => {
    const ts = telem.lastUpdate;
    if (!ts) {
      if (samples.length) {
        samples = [];
        history = [];
      }
      return;
    }
    const last = samples[samples.length - 1];
    if (last && ts <= last.t) return;
    if (last && ts - last.t > STALE_GAP_MS) samples = []; // reconnect / replay jump
    samples.push({ t: ts, v: telem.groundSpeed });
    const cutoff = ts - HISTORY_MS;
    while (samples.length > 1 && samples[0].t < cutoff) samples.shift();
    if (ts - lastPublish >= PUBLISH_MS) {
      lastPublish = ts;
      history = samples.slice();
    }
  });

  // The Speed widget is square (`--ws`): size it from the measured cell edge.
  let speedCellW = $state(0);
  let speedCellH = $state(0);
  let speedSize = $derived(Math.max(0, Math.min(speedCellW, speedCellH)));
</script>

<div class="cp-root" role="region" aria-label={$t('chartPlotter.title')}>
  <WindRoseCard {telem} {interfaceSettings} />

  <SpeedTrendCard
    samples={history}
    now={telem.lastUpdate}
    currentMs={telem.groundSpeed}
    {interfaceSettings}
  />

  <div class="tiles">
    <MissionCard />
    <div class="speed-cell" bind:clientWidth={speedCellW} bind:clientHeight={speedCellH}>
      <SpeedWidget {telem} size={speedSize} {interfaceSettings} />
    </div>
  </div>
</div>

<style>
  .cp-root {
    width: 100%;
    height: 100%;
    min-height: 0;
    box-sizing: border-box;
    display: flex;
    flex-direction: column;
    gap: 6px;
    padding: 6px;
    overflow: hidden;
    background: #0a0a0a;
    border: 1px solid #202020;
    border-radius: 10px;
  }

  .tiles {
    flex: 0.72 1 0;
    min-height: 0;
    display: flex;
    gap: 6px;
  }

  /* The Speed widget is square and brings its own card, so this cell only reserves a square slot
     next to the wide speed readout (like the reference's wide + narrow tile pair). */
  .speed-cell {
    flex: 0 0 auto;
    align-self: stretch;
    aspect-ratio: 1;
    height: 100%;
    min-height: 0;
    display: flex;
    align-items: center;
    justify-content: center;
  }
</style>
