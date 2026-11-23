<!--
  SPDX-License-Identifier: GPL-3.0-or-later
  Copyright (C) 2026 Marc Hoffmann (b14ckyy)
-->

<!-- Mission card of the chart plotter dashboard (replaces the old altitude/GS tile).
     Two-step workflow: LOAD brings a mission file into the planner (INAV `.mission`, ArduPilot/PX4
     `.plan` / `.waypoints`), EXECUTE uploads the loaded mission to the FC. The pilot then switches
     the vehicle to its mission mode on the RC (INAV NAV WP, ArduPilot/PX4 AUTO) to run it — the card
     deliberately does not start a mission on its own. -->
<script lang="ts">
  import { t } from 'svelte-i18n';
  import { get } from 'svelte/store';
  import { open } from '@tauri-apps/plugin-dialog';
  import { invoke } from '@tauri-apps/api/core';
  import { autopilotSystem } from '$lib/stores/autopilotContext';
  import { hasMsp, isArduPilotLink } from '$lib/stores/connection';
  import { mission, missionFlags, missionLoadFile, missionUpload, onMissionUploadProgress } from '$lib/stores/mission';
  import {
    arduMission, arduMissionFcSynced, loadArduMissionFromFile, markArduMissionSynced,
    parsePlanFile, parseWaypoints, framesForUpload, type ArduWaypoint,
  } from '$lib/stores/missionArdupilot';
  import { activeVehicleId } from '$lib/stores/vehicles';
  import { frameMissionOnMap } from '$lib/stores/mapCamera';
  import { anyCase } from '$lib/helpers/fileFilters';

  type StatusKind = 'muted' | 'ok' | 'warn' | 'error';

  const inav = $derived($autopilotSystem === 'inav');
  const wpCount = $derived(inav ? $mission.waypoints.length : $arduMission.length);
  const synced = $derived(inav ? $missionFlags.fc : $arduMissionFcSynced);
  const canTransfer = $derived(inav ? $hasMsp : $isArduPilotLink);

  let busy = $state<'load' | 'upload' | null>(null);
  let actionStatus = $state<{ text: string; kind: StatusKind } | null>(null);

  const status = $derived.by(() => {
    if (actionStatus) return actionStatus;
    if (wpCount === 0) return { text: $t('chartPlotter.noMission'), kind: 'muted' as StatusKind };
    if (!synced) return { text: $t('chartPlotter.notSynced'), kind: 'warn' as StatusKind };
    return { text: $t('chartPlotter.onFc'), kind: 'ok' as StatusKind };
  });

  function setStatus(text: string, kind: StatusKind): void {
    actionStatus = { text, kind };
  }

  /** File picker for the active stack: INAV `.mission` XML, ArduPilot/PX4 QGC `.plan` / `.waypoints`. */
  async function pickFile(): Promise<string | null> {
    const path = await open({
      title: $t('mission.openMissionTitle'),
      multiple: false,
      filters: [{
        name: 'Missions',
        extensions: inav ? anyCase(['mission']) : anyCase(['waypoints', 'txt', 'plan']),
      }],
    });
    if (typeof path === 'string') return path;
    return Array.isArray(path) ? path[0] ?? null : null;
  }

  /** Live "x of n" during a transfer, for both stacks (the backend emits the same progress event). */
  async function withProgress<T>(run: () => Promise<T>): Promise<T> {
    const un = await onMissionUploadProgress(({ current, total }) => {
      setStatus($t('mission.uploadingProgress', { values: { current, total } }), 'muted');
    });
    try {
      return await run();
    } finally {
      un();
    }
  }

  async function uploadInav(): Promise<void> {
    await withProgress(() => missionUpload(false));
  }

  async function uploadArdu(wps: ArduWaypoint[]): Promise<void> {
    await withProgress(async () => {
      await invoke<void>('ardu_mission_upload', {
        waypoints: framesForUpload(wps),
        vehicleId: get(activeVehicleId),
      });
      markArduMissionSynced('fc', wps);
    });
  }

  async function handleLoad(): Promise<void> {
    if (busy) return;
    let path: string | null;
    try {
      path = await pickFile();
    } catch (e) {
      setStatus($t('mission.openFailed', { values: { error: String(e) } }), 'error');
      return;
    }
    if (!path) return;

    busy = 'load';
    actionStatus = null;
    try {
      let count: number;
      if (inav) {
        const m = await missionLoadFile(path);
        count = m.waypoints.length;
      } else {
        const content = await invoke<string>('read_text_file', { path });
        const wps = /\.plan$/i.test(path) ? parsePlanFile(content) : parseWaypoints(content);
        loadArduMissionFromFile(wps);
        frameMissionOnMap();
        count = wps.length;
      }
      setStatus(
        count > 0
          ? $t('chartPlotter.loadedHint', { values: { count } })
          : $t('chartPlotter.noMission'),
        count > 0 ? 'ok' : 'muted',
      );
    } catch (e) {
      setStatus($t('mission.openFailed', { values: { error: String(e) } }), 'error');
    } finally {
      busy = null;
    }
  }

  /** EXECUTE = apply: push the loaded mission to the FC (the pilot then selects AUTO / NAV WP). */
  async function handleExecute(): Promise<void> {
    if (busy || !canTransfer || wpCount === 0) return;
    const count = wpCount;
    busy = 'upload';
    actionStatus = null;
    try {
      if (inav) {
        await uploadInav();
      } else {
        await uploadArdu(get(arduMission));
      }
      setStatus($t('chartPlotter.uploadedHint', { values: { count } }), 'ok');
    } catch (e) {
      setStatus($t('mission.uploadFailed', { values: { error: String(e) } }), 'error');
    } finally {
      busy = null;
    }
  }
</script>

<div class="card">
  <div class="head">
    <span class="label">{$t('chartPlotter.mission')}</span>
    <span class="count">{$t('chartPlotter.wpCount', { values: { count: wpCount } })}</span>
  </div>

  <div class="actions">
    <button class="act load" onclick={handleLoad} disabled={busy !== null}
            title={$t('chartPlotter.loadHint')}>
      <svg viewBox="0 0 24 24" width="18" height="18" aria-hidden="true">
        <path d="M3 7.5A1.5 1.5 0 0 1 4.5 6h4l2 2.2h7.5A1.5 1.5 0 0 1 19.5 9.7v7.8A1.5 1.5 0 0 1 18 19H4.5A1.5 1.5 0 0 1 3 17.5z"
              fill="none" stroke="currentColor" stroke-width="1.7" stroke-linejoin="round" />
        <path d="M12 16v-4M10 13.2l2-2 2 2" fill="none" stroke="currentColor" stroke-width="1.7"
              stroke-linecap="round" stroke-linejoin="round" />
      </svg>
      <span>{busy === 'load' ? '…' : $t('chartPlotter.load')}</span>
    </button>

    <button class="act execute" class:warn={wpCount > 0 && !synced}
            onclick={handleExecute}
            disabled={busy !== null || !canTransfer || wpCount === 0}
            title={canTransfer ? $t('chartPlotter.executeHint') : $t('chartPlotter.executeNeedsLink')}>
      <svg viewBox="0 0 24 24" width="18" height="18" aria-hidden="true">
        <path d="M12 4v10.2M7.8 10.4l4.2 4.2 4.2-4.2" fill="none" stroke="currentColor"
              stroke-width="2" stroke-linecap="round" stroke-linejoin="round" />
        <path d="M5 17.5h14" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" />
      </svg>
      <span>{busy === 'upload' ? '…' : $t('chartPlotter.execute')}</span>
    </button>
  </div>

  <span class="status {status.kind}" title={status.text}>{status.text}</span>
</div>

<style>
  .card {
    flex: 1 1 0;
    min-width: 0;
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
  .count {
    font-size: 11px;
    color: #8a9099;
    font-variant-numeric: tabular-nums;
  }

  .actions {
    flex: 1 1 auto;
    min-height: 0;
    display: flex;
    gap: 6px;
  }

  .act {
    flex: 1 1 0;
    min-width: 0;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    gap: 4px;
    padding: 4px;
    border-radius: 8px;
    font-size: clamp(11px, 1.5vh, 14px);
    font-weight: 700;
    letter-spacing: 0.08em;
    cursor: pointer;
    transition: background-color 0.15s, border-color 0.15s, color 0.15s;
  }
  .act.load {
    border: 1px solid #2f566b;
    background: rgba(55, 168, 219, 0.12);
    color: #7fd4ff;
  }
  .act.load:hover:not(:disabled) {
    background: rgba(55, 168, 219, 0.24);
  }
  .act.execute {
    border: 1px solid #3f6b33;
    background: rgba(89, 170, 41, 0.14);
    color: #8fd66a;
  }
  .act.execute:hover:not(:disabled) {
    background: rgba(89, 170, 41, 0.26);
  }
  /* Not uploaded to the FC yet — the start button reads as a warning, like the control panel's. */
  .act.execute.warn {
    border-color: #8a6a20;
    background: rgba(245, 166, 35, 0.14);
    color: #f5a623;
  }
  .act.execute.warn:hover:not(:disabled) {
    background: rgba(245, 166, 35, 0.26);
  }
  .act:disabled {
    opacity: 0.42;
    cursor: default;
  }

  .status {
    flex: none;
    font-size: 11px;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
  }
  .status.muted { color: #7f8790; }
  .status.ok { color: #59aa29; }
  .status.warn { color: #f5a623; }
  .status.error { color: #e05555; }
</style>
