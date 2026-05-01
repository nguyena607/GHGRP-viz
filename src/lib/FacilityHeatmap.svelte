<script>
  import { onMount } from 'svelte';
  import * as d3 from 'd3';

  let svgEl;
  let containerEl;
  let legendItems = [];
  let tooltip = { visible: false, x: 0, y: 0, facility: null };

  // Sort state — shared with the template
  let sortBy = 'sector'; // 'sector' | 'missing'

  // Stored after load so the sort toggle can redraw without re-fetching
  let _years      = [];
  let _facilities = [];
  let _color      = null;
  let _animating  = false;

  const ROW_H = 6;
  const M = { top: 8, right: 8, bottom: 54, left: 8 };

  function sorted(facilities) {
    if (sortBy === 'missing') {
      return [...facilities].sort((a, b) => b.n_missing - a.n_missing);
    }
    return facilities; // original: sector-grouped, then by n_missing within sector
  }

  function setSort(value) {
    if (sortBy === value || !_facilities.length) return;
    sortBy = value;
    draw(_years, sorted(_facilities), _color, false);
  }

  onMount(async () => {
    const { years, facilities } = await fetch('/data/facility_heatmap.json').then(r => r.json());

    const sectors = [...new Set(facilities.map(f => f.sector))].sort();
    const color   = d3.scaleOrdinal(d3.schemeTableau10).domain(sectors);
    legendItems   = sectors.map(s => ({ sector: s, color: color(s) }));

    _years      = years;
    _facilities = facilities;
    _color      = color;

    let animated = false;
    let io;

    draw(years, sorted(facilities), color, false);

    function triggerAnimation() {
      if (animated) return;
      animated   = true;
      _animating = true;
      draw(years, sorted(facilities), color, true);
      setTimeout(() => { _animating = false; }, 2000);
    }

    await new Promise(r => requestAnimationFrame(r));
    if (!containerEl) return;

    const rect = containerEl.getBoundingClientRect();
    if (rect.top < window.innerHeight * 0.92 && rect.bottom > 0) {
      triggerAnimation();
    } else {
      io = new IntersectionObserver(entries => {
        if (entries[0].isIntersecting) { io.disconnect(); triggerAnimation(); }
      }, { threshold: 0.1 });
      io.observe(containerEl);
    }

    const ro = new ResizeObserver(() => {
      if (!_animating) draw(_years, sorted(_facilities), _color, false);
    });
    ro.observe(containerEl);

    return () => { io?.disconnect(); ro.disconnect(); };
  });

  function draw(years, facilities, color, animate = false) {
    if (!svgEl || !containerEl) return;
    const W      = containerEl.clientWidth;
    const iW     = W - M.left - M.right;
    const totalH = M.top + facilities.length * ROW_H + M.bottom;

    const svg = d3.select(svgEl).attr('width', W).attr('height', totalH);
    svg.selectAll('*').remove();
    const g = svg.append('g').attr('transform', `translate(${M.left},${M.top})`);

    const x = d3.scaleBand().domain(years).range([0, iW]).padding(0.04);

    // Sector separators — only in sector mode
    if (sortBy === 'sector') {
      let prevSector = null;
      facilities.forEach((facility, fi) => {
        if (facility.sector !== prevSector && fi > 0) {
          g.append('line')
            .attr('x1', 0).attr('x2', iW)
            .attr('y1', fi * ROW_H).attr('y2', fi * ROW_H)
            .attr('stroke', '#eeeeee').attr('stroke-width', 2);
        }
        prevSector = facility.sector;
      });
    }

    // Cells drawn year-first so each column fades in together
    years.forEach((yr, yi) => {
      facilities.forEach((facility, fi) => {
        const cell = g.append('rect')
          .attr('x', x(yr)).attr('y', fi * ROW_H)
          .attr('width', x.bandwidth()).attr('height', ROW_H - 1)
          .attr('fill', facility.reporting[yi] ? color(facility.sector) : '#eeeeee')
          .attr('rx', 0.5);

        if (animate) {
          cell.attr('opacity', 0)
            .transition()
            .delay(yi * 90)
            .duration(200)
            .ease(d3.easeQuadOut)
            .attr('opacity', 1);
        }
      });
    });

    // Invisible hover rows
    facilities.forEach((facility, fi) => {
      g.append('rect')
        .attr('x', 0).attr('y', fi * ROW_H)
        .attr('width', iW).attr('height', ROW_H)
        .attr('fill', 'transparent').attr('cursor', 'pointer')
        .on('mousemove', function (event) {
          const [px, py] = d3.pointer(event, containerEl);
          const missingYears = years.filter((_, i) => !facility.reporting[i]);
          tooltip = { visible: true, x: px, y: py, facility: { ...facility, missingYears } };
        })
        .on('mouseleave', () => { tooltip = { ...tooltip, visible: false }; });
    });

    const iH = facilities.length * ROW_H;
    g.append('g')
      .attr('transform', `translate(0,${iH + 4})`)
      .call(d3.axisBottom(x))
      .call(gg => gg.select('.domain').attr('stroke', '#d2d2d7'))
      .call(gg => gg.selectAll('text')
        .attr('fill', '#86868b').attr('font-size', 11)
        .attr('transform', 'rotate(-45)')
        .attr('text-anchor', 'end').attr('dy', '0.35em').attr('dx', '-0.5em'))
      .call(gg => gg.selectAll('.tick line').attr('stroke', '#d2d2d7'));
  }
</script>

<div class="wrap" bind:this={containerEl}>
  <div class="toolbar">
    <div class="legend">
      {#each legendItems as item}
        <div class="legend-item">
          <span class="swatch" style="background:{item.color}"></span>
          <span class="label">{item.sector}</span>
        </div>
      {/each}
      <div class="legend-item">
        <span class="swatch missing-swatch"></span>
        <span class="label muted">Missing</span>
      </div>
    </div>

    <div class="seg" role="group" aria-label="Sort order">
      <button class:active={sortBy === 'sector'}  on:click={() => setSort('sector')}>By Sector</button>
      <button class:active={sortBy === 'missing'} on:click={() => setSort('missing')}>Most Missing</button>
    </div>
  </div>

  <svg bind:this={svgEl}></svg>

  {#if tooltip.visible && tooltip.facility}
    {@const f = tooltip.facility}
    <div
      class="tt"
      style="left:{Math.min(tooltip.x + 16, containerEl.clientWidth - 250)}px; top:{tooltip.y}px; transform: translateY(calc(-100% - 12px))"
    >
      <div class="tt-name">{f.facility_name}</div>
      <div class="tt-meta">{f.sector}{f.state ? ' · ' + f.state : ''}</div>
      <div class="tt-stat" class:all-good={f.n_missing === 0} class:some-missing={f.n_missing > 0}>
        {#if f.n_missing === 0}
          Reported all {f.reporting.length} years
        {:else}
          {f.n_missing} of {f.reporting.length} years missing
        {/if}
      </div>
      {#if f.missingYears.length > 0 && f.missingYears.length <= 8}
        <div class="tt-years">Missing: {f.missingYears.join(', ')}</div>
      {/if}
    </div>
  {/if}
</div>

<style>
  .wrap { position: relative; width: 100%; min-width: 480px; }
  svg { display: block; width: 100%; }

  /* ── Toolbar: legend + sort control ─────────────────────────────────── */
  .toolbar {
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    flex-wrap: wrap;
    gap: 10px;
    margin-bottom: 12px;
  }

  .legend { display: flex; flex-wrap: wrap; gap: 6px 16px; }
  .legend-item { display: flex; align-items: center; gap: 5px; }
  .swatch { display: inline-block; width: 11px; height: 11px; border-radius: 3px; flex-shrink: 0; }
  .missing-swatch { background: #eeeeee; border: 1px solid #d2d2d7; }
  .label { font-size: 11px; color: #6e6e73; }
  .muted { color: #86868b; }

  /* ── Segmented control ───────────────────────────────────────────────── */
  .seg {
    display: flex;
    background: #f5f5f7;
    border-radius: 8px;
    padding: 2px;
    gap: 2px;
    flex-shrink: 0;
  }

  .seg button {
    font-family: inherit;
    font-size: 11px;
    font-weight: 500;
    color: #6e6e73;
    background: transparent;
    border: none;
    border-radius: 6px;
    padding: 4px 10px;
    cursor: pointer;
    transition: background 0.15s, color 0.15s;
    white-space: nowrap;
  }

  .seg button:hover:not(.active) {
    background: rgba(0,0,0,0.05);
    color: #1d1d1f;
  }

  .seg button.active {
    background: #ffffff;
    color: #1d1d1f;
    box-shadow: 0 1px 3px rgba(0,0,0,0.12);
  }

  /* ── Tooltip ─────────────────────────────────────────────────────────── */
  .tt {
    position: absolute; background: #ffffff; border: 1px solid #e8e8ed;
    border-radius: 12px; padding: 11px 15px; pointer-events: none;
    font-size: 13px; color: #1d1d1f; max-width: 238px;
    box-shadow: 0 4px 24px rgba(0,0,0,.10), 0 1px 4px rgba(0,0,0,.06); z-index: 20;
  }
  .tt-name { font-weight: 600; font-size: 13px; margin-bottom: 3px; line-height: 1.3; }
  .tt-meta { color: #86868b; font-size: 11px; margin-bottom: 7px; }
  .tt-stat { font-weight: 600; font-size: 12px; margin-bottom: 3px; }
  .all-good { color: #34c759; }
  .some-missing { color: #ff3b30; }
  .tt-years { color: #86868b; font-size: 11px; margin-top: 3px; }
</style>
