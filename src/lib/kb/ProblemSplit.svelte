<script lang="ts">
  // #4471 split into the problems it holds. Built from the export at render time, never typed.
  // causes.ts places each report by tenant and channel; the desk's one-day-move reads
  // (desk-notes.json) are a separate problem, so they come out of whatever group they were filed in.
  import reports from '#lib/data/reports-4471.json';
  import notes from '#lib/data/desk-notes.json';
  import { groupByCause, uniqueById, type Report } from '#lib/kb/causes';
  import { tenantOf } from '#lib/kb/locales';

  type Key = 'parser' | 'hint' | 'one-day' | 'unplaced';
  let { ticket, issues = {} }: { ticket: number; issues?: Partial<Record<Key, string>> } = $props();

  const LABEL: Record<Key, string> = {
    parser: 'Shared parser reads day-first dates month first',
    hint: 'booking-v3 form hints MM/DD/YYYY, reads day first',
    'one-day': 'Date shifts by one day (time zone), not a swap',
    unplaced: 'No tenant on file: cause unknown',
  };

  const merged = $derived(uniqueById(reports as Report[]).filter((r) => r.mergedInto === ticket));
  const oneDay = $derived(merged.filter((r) => notes.oneDayMove.includes(r.id)));
  const rows = $derived.by(() => {
    const rest = merged.filter((r) => !notes.oneDayMove.includes(r.id));
    const groups = groupByCause(rest, tenantOf).map((g) => ({ key: g.cause as Key, reports: g.reports }));
    if (oneDay.length) groups.push({ key: 'one-day', reports: oneDay });
    return groups
      .map((g) => {
        const days = g.reports.map((r) => r.receivedAt.slice(0, 10)).sort();
        return {
          ...g,
          tenants: new Set(g.reports.map((r) => r.tenant)).size,
          channels: [...new Set(g.reports.map((r) => r.channel))].sort().join(', '),
          seen: `${days[0]} → ${days[days.length - 1]}`,
        };
      })
      .sort((a, b) => b.reports.length - a.reports.length);
  });
  const total = $derived(rows.reduce((n, r) => n + r.reports.length, 0));
</script>

<p class="sum">
  <strong>{merged.length}</strong> unique reports merged into #{ticket} ·
  <strong>{rows.length}</strong> problems · {total === merged.length ? 'every report placed once' : `${merged.length - total} not placed`}
</p>

<table>
  <thead><tr><th>problem</th><th>reports</th><th>tenants</th><th>channels</th><th>seen</th><th>ticket</th></tr></thead>
  <tbody>
    {#each rows as r (r.key)}
      <tr>
        <td>{LABEL[r.key]} <code>{r.key}</code></td>
        <td><strong>{r.reports.length}</strong></td>
        <td>{r.tenants}</td>
        <td>{r.channels}</td>
        <td>{r.seen}</td>
        <td>{#if issues[r.key]}<a href={issues[r.key]}>{issues[r.key].replace(/^.*\/issues\//, '#')}</a>{:else}none yet{/if}</td>
      </tr>
    {/each}
  </tbody>
</table>

<style>
  .sum { font-size: 1.1rem; }
</style>
