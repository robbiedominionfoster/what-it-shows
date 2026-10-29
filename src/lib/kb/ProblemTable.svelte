<script lang="ts">
  import reports from '#lib/data/reports-4471.json';
  import notes from '#lib/data/desk-notes.json';
  import { groupByCause, uniqueById, type CauseGroup, type Report } from '#lib/kb/causes';
  import { tenantOf } from '#lib/kb/locales';

  type Problem = CauseGroup['cause'] | 'oneDayMove';

  // tickets: where each problem is filed now, by problem.
  let { ticket, tickets }: { ticket: number; tickets: Partial<Record<Problem, string>> } = $props();

  const rows = $derived((reports as Report[]).filter((r) => r.mergedInto === ticket));
  const merged = $derived(uniqueById(rows));
  // A one-day move is not a day/month swap, whatever the tenant: it leaves its cause before grouping.
  const moved = $derived(merged.filter((r) => notes.oneDayMove.includes(r.id)));
  const groups = $derived.by(() => {
    const byCause: { problem: Problem; g: CauseGroup }[] = groupByCause(
      merged.filter((r) => !notes.oneDayMove.includes(r.id)),
      tenantOf,
    ).map((g) => ({ problem: g.cause, g }));
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
    <span class="dupes">({rows.length} rows in the export: {rows.length - merged.length} came twice, counted once)</span>
  {/if}
</p>

<table>
  <thead>
    <tr><th>problem</th><th>reports</th><th>intake tagged as</th><th>tenants</th><th>seen</th><th>ticket</th></tr>
  </thead>
  <tbody>
    {#each groups as { problem, g } (problem)}
      <tr class:open={!tickets[problem]}>
        <td>{LABEL[problem]}</td>
        <td>{g.reports.length}</td>
        <td class="tags">{tags(g)}</td>
        <td>{g.tenants}</td>
        <td>{g.firstSeen} → {g.lastSeen}</td>
        <td>
          {#if tickets[problem]}<a href={tickets[problem]}>{issue(tickets[problem])}</a>{:else}none yet{/if}
        </td>
      </tr>
    {/each}
  </tbody>
</table>

<style>
  .sum { font-size: 1.1rem; }
  .dupes { color: var(--muted); font-size: 0.85rem; }
  .tags { color: var(--muted); font-size: 0.85em; }
  .open td { background: var(--warn-bg); }
  .open td:last-child { color: var(--warn); font-weight: 600; }
</style>
