<script lang="ts">
  import reports from '#lib/data/reports-4471.json';
  import notes from '#lib/data/desk-notes.json';
  import { groupByCause, readAs, uniqueById, type CauseGroup, type Report } from '#lib/kb/causes';
  import { tenantOf } from '#lib/kb/locales';

  type Problem = CauseGroup['cause'] | 'oneDayMove';
  type Filed = { url: string; state: string };

  // tickets: where each problem is filed now, and how it stands.
  let { ticket, tickets }: { ticket: number; tickets: Partial<Record<Problem, Filed>> } = $props();

  const rows = $derived((reports as Report[]).filter((r) => r.mergedInto === ticket));
  const merged = $derived(uniqueById(rows));
  // Counted the desk's way: a report stays under the cause its tenant files it under.
  const groups = $derived.by(() => {
    const byCause: { problem: Problem; g: CauseGroup }[] = groupByCause(merged, tenantOf).map((g) => ({ problem: g.cause, g }));
    const moved = merged.filter((r) => notes.oneDayMove.includes(r.id));
    if (moved.length) byCause.push({ problem: 'oneDayMove', g: groupByCause(moved, () => undefined)[0] });
    return byCause;
  });

  const LABEL: Record<Problem, string> = {
    parser: 'shared parser reads day-first dates month first',
    hint: 'booking-v3 form hint says MM/DD, form reads day first',
    oneDayMove: 'date moves by one day (time zone)',
    unplaced: 'no cause on file',
  };
  const tags = (g: CauseGroup) =>
    Object.entries(g.taggedAs)
      .sort((a, b) => b[1] - a[1])
      .map(([tag, n]) => `${tag} ×${n}`)
      .join(' · ');
  const issue = (url: string) => `#${url.split('/').pop()}`;
</script>

<p class="sum">
  <strong>{merged.length}</strong> reports in #{ticket} ·
  <strong>{groups.length}</strong>
  {groups.length === 1 ? 'problem' : 'problems'}
  {#if rows.length !== merged.length}
    <span class="muted">({rows.length} rows in the export: {rows.length - merged.length} came twice, counted once)</span>
  {/if}
</p>

<table>
  <thead>
    <tr><th>problem</th><th>reports</th><th>intake tagged as</th><th>tenants</th><th>seen</th><th>ticket</th><th>state</th></tr>
  </thead>
  <tbody>
    {#each groups as { problem, g } (problem)}
      {@const filed = tickets[problem]}
      <tr class:open={!filed || filed.state === 'open'}>
        <td>{LABEL[problem]}</td>
        <td>{g.reports.length}</td>
        <td class="muted">{tags(g)}</td>
        <td>{g.tenants}</td>
        <td>{g.firstSeen} → {g.lastSeen}</td>
        <td>{#if filed}<a href={filed.url}>{issue(filed.url)}</a>{:else}none yet{/if}</td>
        <td>{filed?.state ?? 'open'}</td>
      </tr>
    {/each}
  </tbody>
</table>

{#each groups as { problem, g } (problem)}
  {@const moved = problem === 'oneDayMove' ? 0 : readAs(g, notes.oneDayMove)}
  {#if moved}
    <p class="muted">
      {moved} of the <code>{problem}</code> reports read as a one-day move. The desk counts them under <code>{problem}</code> by
      tenant; they are also the one-day move row.
    </p>
  {/if}
{/each}

<style>
  .sum { font-size: 1.1rem; }
  .muted { color: var(--muted); font-size: 0.85em; }
  .open td { background: var(--warn-bg); }
  .open td:last-child { color: var(--warn); font-weight: 600; }
</style>
