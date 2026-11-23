<!--
  SPDX-License-Identifier: GPL-3.0-or-later
  Copyright (C) 2026 Marc Hoffmann (b14ckyy)
-->

<!-- Wind rose card of the chart plotter dashboard — dial geometry and palette after Kip's
     wind-steer instrument (thin silver tick ring, dark number band, black card, heading badge).
     Heading-up card with the yellow "T" true-wind arrow, the pink course-over-ground marker, the
     blue downwind (drift) arrow, the green crab sector (bow → track), the wind angle relative to the
     bow in the centre, and the AIR/GS (orange) + WIND (yellow) readouts in the top corners. -->
<script lang="ts">
  import { t } from 'svelte-i18n';
  import type { TelemetryData } from '$lib/stores/telemetry';
  import type { InterfaceSettings } from '$lib/stores/settings';
  import { convertSpeed, speedDigits } from '$lib/utils/units';

  let {
    telem,
    interfaceSettings,
  }: {
    telem: TelemetryData;
    interfaceSettings: InterfaceSettings;
  } = $props();

  // Kip's dial proportions (1000-unit geometry divided by 5): a thin silver tick ring outside a
  // dark number band, black card inside.
  const TICKS = Array.from({ length: 72 }, (_, i) => {
    const deg = i * 5;
    const major = deg % 30 === 0;
    const rad = ((deg - 90) * Math.PI) / 180;
    const outer = 97.5;
    const inner = outer - (major ? 8 : deg % 10 === 0 ? 5 : 3);
    return {
      major,
      x1: 100 + outer * Math.cos(rad),
      y1: 100 + outer * Math.sin(rad),
      x2: 100 + inner * Math.cos(rad),
      y2: 100 + inner * Math.sin(rad),
    };
  });

  const DEG_LABELS = Array.from({ length: 12 }, (_, i) => i * 30).filter((deg) => deg % 90 !== 0);

  const CARDINALS = [
    { deg: 0, key: 'compass.n' },
    { deg: 90, key: 'compass.e' },
    { deg: 180, key: 'compass.s' },
    { deg: 270, key: 'compass.w' },
  ];

  /** Signed angle in −180…+180 (positive = clockwise / starboard). */
  function norm180(deg: number): number {
    return ((deg + 540) % 360) - 180;
  }

  const WIND_MIN_MS = 0.3;
  const COG_MIN_SPEED = 1.5; // m/s

  let live = $derived(telem.lastUpdate > 0);
  let heading = $derived(((telem.yaw % 360) + 360) % 360);
  let digits = $derived(speedDigits(interfaceSettings.speedUnit));

  // AIR = the airflow the airframe sees: indicated airspeed when a pitot reports, else ground speed.
  let airMs = $derived(telem.airspeed > 0 ? telem.airspeed : telem.groundSpeed);
  let airConv = $derived(convertSpeed(airMs, interfaceSettings.speedUnit));
  let windConv = $derived(convertSpeed(telem.windSpeedMs, interfaceSettings.speedUnit));

  let windShown = $derived(live && telem.windSpeedMs > WIND_MIN_MS);
  let cogShown = $derived(live && telem.groundSpeed > COG_MIN_SPEED);
  let airLabel = $derived(telem.airspeed > 0 ? $t('chartPlotter.air') : $t('chartPlotter.gs'));

  /** Wind angle relative to the bow (TWA), −180…+180; null without a wind estimate. */
  let twa = $derived(windShown ? norm180(telem.windDirFrom - heading) : null);
  /** Crab angle: course over ground relative to the heading (0 while standing still). */
  let crab = $derived(cogShown ? norm180(telem.course - heading) : 0);

  // Green sector between the bow (card top) and the actual track.
  let crabPath = $derived.by(() => {
    if (!cogShown || Math.abs(crab) < 1) return '';
    const r = 62;
    const rad = (crab * Math.PI) / 180;
    const x = 100 + r * Math.sin(rad);
    const y = 100 - r * Math.cos(rad);
    return `M100,100 L100,${100 - r} A${r},${r} 0 0 ${crab > 0 ? 1 : 0} ${x.toFixed(2)},${y.toFixed(2)} Z`;
  });
  let crabEdge = $derived.by(() => {
    const rad = (crab * Math.PI) / 180;
    const r = 62;
    return { x: 100 + r * Math.sin(rad), y: 100 - r * Math.cos(rad) };
  });
</script>

<div class="card">
  <div class="readouts">
    <div class="ro orange">
      <span class="ro-label">{airLabel}</span>
      <span class="ro-value" class:muted={!live}>{live ? airConv.value.toFixed(digits) : '—'}</span>
      <span class="ro-unit">{airConv.unit}</span>
    </div>
    <div class="ro right yellow">
      <span class="ro-label">{$t('chartPlotter.wind')}</span>
      <span class="ro-value" class:muted={!windShown}>{windShown ? windConv.value.toFixed(digits) : '—'}</span>
      <span class="ro-unit">{windConv.unit}</span>
    </div>
  </div>

  <div class="rose">
    <svg viewBox="0 0 200 200" preserveAspectRatio="xMidYMid meet" role="img"
         aria-label={$t('chartPlotter.windRose')}>
      <!-- Kip's dial: thin silver tick ring, dark number band, black card disc. -->
      <circle cx="100" cy="100" r="94.1" class="silver-ring" />
      <path d="M 18.6,53 A 94,94 0 0 1 67.9,11.7" class="wind-sector-port" />
      <path d="M 132.1,11.7 A 94,94 0 0 1 181.4,53" class="wind-sector-stbd" />
      <circle cx="100" cy="100" r="80.8" class="number-ring" />
      <circle cx="100" cy="100" r="71.3" class="inner-disc" />

      <!-- Rotating card (heading-up): ticks and labels turn with the heading. -->
      <g transform="rotate({-heading}, 100, 100)">
        {#each TICKS as tk}
          <line x1={tk.x1} y1={tk.y1} x2={tk.x2} y2={tk.y2} class:major={tk.major} />
        {/each}

        {#each DEG_LABELS as deg}
          <g transform="rotate({deg}, 100, 100)">
            <text x="100" y="23" text-anchor="middle" dominant-baseline="middle" class="deg-label">{deg}</text>
          </g>
        {/each}

        {#each CARDINALS as c}
          <g transform="rotate({c.deg}, 100, 100)">
            <text x="100" y="23" text-anchor="middle" dominant-baseline="middle" class="cardinal">{$t(c.key)}</text>
          </g>
        {/each}

        <!-- Crab sector (bow → track) in the waypoint green. -->
        {#if crabPath}
          <path d={crabPath} class="crab-sector" />
          <line x1="100" y1="100" x2={crabEdge.x} y2={crabEdge.y} class="crab-edge" />
        {/if}

        <!-- Blue downwind (drift) arrow. -->
        {#if windShown}
          <g transform="rotate({telem.windDirFrom + 180}, 100, 100)" class="downwind">
            <line x1="100" y1="92" x2="100" y2="76" />
            <polygon points="100,44 84,74 116,74" />
          </g>
        {/if}

        <!-- Pink course-over-ground marker on the rim. -->
        {#if cogShown}
          <g transform="rotate({telem.course}, 100, 100)" class="cog-marker">
            <rect x="97.6" y="8" width="4.8" height="34" rx="2.4" />
            <circle cx="100" cy="11" r="4.2" />
          </g>
        {/if}

        <!-- Yellow "T" true-wind arrow (the bearing the wind comes FROM). -->
        {#if windShown}
          <g transform="rotate({telem.windDirFrom}, 100, 100)" class="wind-arrow">
            <path d="M100,18 L112,44 L104,44 L104,78 L96,78 L96,44 L88,44 Z" />
            <circle cx="100" cy="56" r="9" />
            <text x="100" y="56.5" text-anchor="middle" dominant-baseline="middle">T</text>
          </g>
        {/if}
      </g>

      <!-- Fixed heading badge; the centre shows the wind angle relative to the bow. -->
      <g class="heading-badge">
        <rect x="74.5" y="2" width="51" height="21.5" rx="5" />
        <text x="100" y="13.2" text-anchor="middle" dominant-baseline="middle">
          {live ? Math.round(heading) + '°' : '—'}
        </text>
      </g>

      <text x="100" y="92" text-anchor="middle" dominant-baseline="middle" class="twa-label">{$t('chartPlotter.twa')}</text>
      <text x="100" y="105" text-anchor="middle" dominant-baseline="middle" class="twa-value">
        {twa !== null ? twa.toFixed(1) + '°' : '—'}
      </text>
    </svg>
  </div>

  <div class="foot">
    {#if windShown}
      <span>{$t('chartPlotter.twd')} {Math.round(telem.windDirFrom)}°</span>
    {/if}
    {#if cogShown}
      <span>{$t('chartPlotter.cog')} {Math.round(telem.course)}°</span>
    {/if}
    {#if !windShown && !cogShown}
      <span>—</span>
    {/if}
  </div>
</div>

<style>
  .card {
    flex: 1.35 1 0;
    min-height: 0;
    display: flex;
    flex-direction: column;
    gap: 2px;
    padding: 6px 10px 4px;
    box-sizing: border-box;
    overflow: hidden;
    background: #101113;
    border: 1px solid #242628;
    border-radius: 8px;
  }

  .readouts {
    display: flex;
    justify-content: space-between;
    flex: none;
    position: relative;
    z-index: 1;
  }
  .ro {
    display: flex;
    flex-direction: column;
    align-items: flex-start;
    line-height: 1.05;
  }
  .ro.right {
    align-items: flex-end;
  }
  .ro-label {
    font-size: 11px;
    font-weight: 700;
    letter-spacing: 0.1em;
  }
  .ro-value {
    font-size: clamp(18px, 2.5vh, 26px);
    font-weight: 800;
    font-variant-numeric: tabular-nums;
  }
  .ro-unit {
    font-size: 10px;
  }
  /* Kip palette: apparent wind (AIR/GS) orange, true wind (WIND) yellow — labels/units dimmed. */
  .ro.orange .ro-value { color: #ff9100; }
  .ro.orange .ro-label,
  .ro.orange .ro-unit { color: #ff9100b6; }
  .ro.yellow .ro-value { color: #ffdf00; }
  .ro.yellow .ro-label,
  .ro.yellow .ro-unit { color: #ffdf00b6; }
  .ro-value.muted { color: #8f8f8f; }

  .rose {
    flex: 1 1 auto;
    min-height: 0;
    display: flex;
    align-items: center;
    justify-content: center;
  }
  .rose svg {
    width: 100%;
    height: 100%;
    max-height: 100%;
  }

  /* Kip dial colours: silver tick ring outside a near-black number band, black card inside. */
  .silver-ring {
    fill: none;
    stroke: #b5b5b5;
    stroke-width: 7.6;
  }
  /* Port (red) and starboard (green) ring sectors — fixed on the dial, like Kip's. */
  .wind-sector-port {
    fill: none;
    stroke: #8f0000;
    stroke-width: 8;
  }
  .wind-sector-stbd {
    fill: none;
    stroke: #008700;
    stroke-width: 8;
  }
  .number-ring {
    fill: none;
    stroke: #191c1d;
    stroke-width: 19;
  }
  .inner-disc {
    fill: #000;
  }

  /* Tick marks on the silver ring (white, long and thick every 30°). */
  line {
    stroke: #fff;
    stroke-width: 1.1;
  }
  line.major {
    stroke-width: 3.2;
  }

  .deg-label {
    fill: #fff;
    font-size: 11px;
    font-weight: 600;
    font-family: 'Segoe UI', Tahoma, sans-serif;
  }
  .cardinal {
    fill: #fff;
    font-size: 12.5px;
    font-weight: 700;
    font-family: 'Segoe UI', Tahoma, sans-serif;
  }

  .crab-sector {
    fill: rgba(0, 208, 0, 0.22);
  }
  .crab-edge {
    stroke: #00d000;
    stroke-width: 1.2;
    stroke-dasharray: 3 3;
  }

  .downwind line {
    stroke: #3298ff;
    stroke-width: 6;
    stroke-linecap: round;
  }
  .downwind polygon {
    fill: #3298ff;
  }

  .cog-marker rect,
  .cog-marker circle {
    fill: #fc0fc0;
  }

  .wind-arrow path {
    fill: #ffdf00;
  }
  .wind-arrow circle {
    fill: #000;
  }
  .wind-arrow text {
    fill: #ffdf00;
    font-size: 11px;
    font-weight: 700;
    font-family: 'Segoe UI', Tahoma, sans-serif;
  }

  .heading-badge rect {
    fill: #000;
    stroke: #b5b5b5;
    stroke-opacity: 0.5;
    stroke-width: 0.9;
  }
  .heading-badge text {
    fill: #fff;
    font-size: 18px;
    font-weight: 700;
    font-family: 'Segoe UI', Tahoma, sans-serif;
    font-variant-numeric: tabular-nums;
  }

  .twa-label {
    fill: #b5b5b5;
    font-size: 7.5px;
    font-weight: 700;
    letter-spacing: 0.12em;
    font-family: 'Segoe UI', Tahoma, sans-serif;
  }
  .twa-value {
    fill: #fff;
    font-size: 14px;
    font-weight: 700;
    font-family: 'Segoe UI', Tahoma, sans-serif;
    font-variant-numeric: tabular-nums;
    paint-order: stroke;
    stroke: #000;
    stroke-width: 2.5;
    stroke-linejoin: round;
  }

  .foot {
    flex: none;
    height: 14px;
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 12px;
    font-size: 10px;
    font-weight: 600;
    letter-spacing: 0.05em;
    color: #6f767e;
    font-variant-numeric: tabular-nums;
  }
</style>
