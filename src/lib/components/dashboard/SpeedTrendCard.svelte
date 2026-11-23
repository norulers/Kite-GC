<!--
  SPDX-License-Identifier: GPL-3.0-or-later
  Copyright (C) 2026 Marc Hoffmann (b14ckyy)
-->

<!-- Ground-speed trend card of the chart plotter dashboard (Kip-inspired): the current speed in
     large digits above a one-minute line chart with the window average as a dashed reference. -->
<script lang="ts">
  import { t } from 'svelte-i18n';
  import type { InterfaceSettings } from '$lib/stores/settings';
  import { convertSpeed, speedDigits } from '$lib/utils/units';

  let {
    samples,
    now,
    currentMs,
    interfaceSettings,
  }: {
    /** Ground-speed samples (raw m/s) over roughly the last minute, oldest first. */
    samples: { t: number; v: number }[];
    /** Timestamp the window is anchored to (the newest telemetry tick). */
    now: number;
    /** Live ground speed, raw m/s. */
    currentMs: number;
    interfaceSettings: InterfaceSettings;
  } = $props();

  const WINDOW_MS = 60_000;

  let digits = $derived(speedDigits(interfaceSettings.speedUnit));
  let current = $derived(convertSpeed(currentMs, interfaceSettings.speedUnit));

  /** Chart geometry in a 0–100 user space (`preserveAspectRatio="none"` stretches it to the box). */
  let chart = $derived.by(() => {
    const end = now > 0 ? now : samples.length ? samples[samples.length - 1].t : 0;
    if (!end) return { line: '', avgY: 50, yMin: 0, yMax: 0, avg: 0, count: 0, flat: false };

    const start = end - WINDOW_MS;
    const pts = samples
      .filter((s) => s.t >= start)
      .map((s) => ({ x: ((s.t - start) / WINDOW_MS) * 100, v: convertSpeed(s.v, interfaceSettings.speedUnit).value }));

    const values = pts.map((p) => p.v);
    let lo = Math.min(...values);
    let hi = Math.max(...values);
    const avg = values.reduce((a, b) => a + b, 0) / values.length;

    // A near-flat trace would collapse to a hairline: open the range to at least one unit.
    const flat = hi - lo < 1;
    if (flat) {
      const mid = (hi + lo) / 2;
      lo = mid - 0.5;
      hi = mid + 0.5;
    }
    const pad = (hi - lo) * 0.12;
    lo = Math.max(0, lo - pad);
    hi += pad;
    const span = hi - lo || 1;

    const toY = (v: number) => 100 - ((v - lo) / span) * 100;
    return {
      line: pts.map((p) => `${p.x.toFixed(2)},${toY(p.v).toFixed(2)}`).join(' '),
      avgY: toY(avg),
      yMin: lo,
      yMax: hi,
      avg,
      count: pts.length,
      flat,
    };
  });
</script>

<div class="card">
  <div class="head">
    <span class="label">{$t('chartPlotter.speed')}</span>
    <span class="big">
      {now > 0 ? current.value.toFixed(digits) : '—'}<span class="unit">{current.unit}</span>
    </span>
  </div>

  <div class="plot-row">
    <div class="plot">
      <svg viewBox="0 0 100 100" preserveAspectRatio="none" aria-hidden="true">
        <!-- Horizontal reference lines at 25/50/75 % of the range. -->
        <line x1="0" y1="25" x2="100" y2="25" class="grid" vector-effect="non-scaling-stroke" />
        <line x1="0" y1="50" x2="100" y2="50" class="grid" vector-effect="non-scaling-stroke" />
        <line x1="0" y1="75" x2="100" y2="75" class="grid" vector-effect="non-scaling-stroke" />

        {#if chart.count > 1}
          <polyline points={chart.line} class="trace" vector-effect="non-scaling-stroke"
                    fill="none" stroke-linejoin="round" stroke-linecap="round" />
          <line x1="0" y1={chart.avgY} x2="100" y2={chart.avgY} class="avg-line"
                vector-effect="non-scaling-stroke" />
        {/if}
      </svg>
    </div>

    <div class="yscale">
      <span>{chart.count > 0 ? chart.yMax.toFixed(digits === 1 ? 1 : 0) : '—'}</span>
      <span>{chart.count > 0 ? chart.yMin.toFixed(digits === 1 ? 1 : 0) : '—'}</span>
    </div>
  </div>

  <div class="foot">
    <span class="avg">
      {#if chart.count > 0}
        {$t('chartPlotter.average')} {chart.avg.toFixed(digits)} {current.unit}
      {:else}
        —
      {/if}
    </span>
    <span class="span">{$t('chartPlotter.lastMinute')}</span>
  </div>
</div>

<style>
  .card {
    flex: 0.95 1 0;
    min-height: 0;
    display: flex;
    flex-direction: column;
    gap: 4px;
    padding: 6px 10px;
    box-sizing: border-box;
    overflow: hidden;
    background: #101113;
    border: 1px solid #242628;
    border-radius: 8px;
  }

  .head {
    flex: none;
    display: flex;
    align-items: baseline;
    justify-content: space-between;
    gap: 8px;
  }
  .label {
    font-size: 12px;
    font-weight: 700;
    letter-spacing: 0.08em;
    color: #7f8790;
  }
  .big {
    font-size: clamp(22px, 3.4vh, 42px);
    font-weight: 700;
    color: #f5a623;
    font-variant-numeric: tabular-nums;
    line-height: 1;
    white-space: nowrap;
  }
  .big .unit {
    margin-left: 5px;
    font-size: 0.45em;
    font-weight: 600;
    color: #b78525;
  }

  .plot-row {
    flex: 1 1 auto;
    min-height: 0;
    display: flex;
    gap: 6px;
  }
  .plot {
    flex: 1 1 auto;
    min-width: 0;
    position: relative;
  }
  .plot svg {
    position: absolute;
    inset: 0;
    width: 100%;
    height: 100%;
  }
  .yscale {
    flex: none;
    width: 30px;
    display: flex;
    flex-direction: column;
    justify-content: space-between;
    align-items: flex-end;
    font-size: 10px;
    color: #8a9099;
    font-variant-numeric: tabular-nums;
  }

  .grid {
    stroke: rgba(255, 255, 255, 0.08);
    stroke-width: 1;
  }
  .trace {
    stroke: #f5a623;
    stroke-width: 2;
  }
  .avg-line {
    stroke: #f5a623;
    stroke-width: 1;
    stroke-dasharray: 4 4;
    opacity: 0.55;
  }

  .foot {
    flex: none;
    display: flex;
    justify-content: space-between;
    gap: 8px;
    font-size: 11px;
    color: #7f8790;
    font-variant-numeric: tabular-nums;
  }
  .span {
    text-align: right;
  }
</style>
