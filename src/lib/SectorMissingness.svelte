<script>
  import { onMount } from 'svelte';
  import * as d3 from 'd3';

  let svgEl;
  let containerEl;
  let legendItems = [];
  let tooltip = { visible: false, x: 0, y: 0, sector: '', year: 0, rate: 0, n: 0 };

  const HEIGHT = 360;
  const M = { top: 24, right: 28, bottom: 46, left: 72 };

  onMount(async () => {
    const data  = await fetch('/data/sector_missingness.json').then(r => r.json());
    const color = d3.scaleOrdinal(d3.schemeTableau10).domain(data.map(d => d.sector));
    legendItems = data.map(d => ({ sector: d.sector, color: color(d.sector), n: d.n_facilities }));

    let animating = false;
    let animated  = false;
    let io;

    draw(data, color, false);

    function triggerAnimation() {
      if (animated) return;
      animated  = true;
      animating = true;
      draw(data, color, true);
      // 8 sectors × 140ms stagger + 1100ms line duration ≈ 2.2s
      setTimeout(() => { animating = false; }, 2500);
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

    const ro = new ResizeObserver(() => { if (!animating) draw(data, color, false); });
    ro.observe(containerEl);

    return () => { io?.disconnect(); ro.disconnect(); };
  });

  function draw(data, color, animate = false) {
    if (!svgEl || !containerEl) return;
    const W  = containerEl.clientWidth;
    const iW = W - M.left - M.right;
    const iH = HEIGHT - M.top - M.bottom;

    const svg = d3.select(svgEl).attr('width', W).attr('height', HEIGHT);
    svg.selectAll('*').remove();
    const g = svg.append('g').attr('transform', `translate(${M.left},${M.top})`);

    const allYears = data[0].years.map(d => d.year);
    const x = d3.scaleLinear().domain(d3.extent(allYears)).range([0, iW]);

    const maxRate = d3.max(data, s => d3.max(s.years, d => d.missing_rate));
    const y = d3.scaleLinear()
      .domain([0, Math.min(maxRate * 1.12, 1)]).range([iH, 0]).nice();

    g.append('g')
      .call(d3.axisLeft(y).ticks(5).tickSize(-iW).tickFormat(''))
      .call(gg => gg.select('.domain').remove())
      .call(gg => gg.selectAll('line').attr('stroke', '#f0f0f0'));

    const lineGen = d3.line()
      .x(d => x(d.year)).y(d => y(d.missing_rate)).curve(d3.curveMonotoneX);

    const STAGGER  = 140;   // ms between each sector's animation start
    const LINE_DUR = 1100;  // ms to draw each line

    // Each sector's line draws itself left→right; dots pop in as the line passes
    data.forEach((sectorData, si) => {
      const c         = color(sectorData.sector);
      const lineDelay = si * STAGGER;

      const path = g.append('path').datum(sectorData.years)
        .attr('fill', 'none').attr('stroke', c)
        .attr('stroke-width', 2).attr('opacity', 0.9).attr('d', lineGen);

      if (animate) {
        const len = path.node().getTotalLength();
        path.attr('stroke-dasharray', `${len} ${len}`)
          .attr('stroke-dashoffset', len)
          .transition()
          .delay(lineDelay)
          .duration(LINE_DUR)
          .ease(d3.easeQuadInOut)
          .attr('stroke-dashoffset', 0);
      }

      const dots = g.selectAll(null).data(sectorData.years).join('circle')
        .attr('cx', d => x(d.year)).attr('cy', d => y(d.missing_rate))
        .attr('r', 4).attr('fill', c)
        .attr('stroke', '#ffffff').attr('stroke-width', 1.5)
        .attr('cursor', 'pointer')
        .on('mousemove', function (event, d) {
          d3.select(this).transition().duration(80).attr('r', 6);
          const [px, py] = d3.pointer(event, containerEl);
          tooltip = { visible: true, x: px, y: py, sector: sectorData.sector, year: d.year, rate: d.missing_rate, n: sectorData.n_facilities };
        })
        .on('mouseleave', function () {
          d3.select(this).transition().duration(80).attr('r', 4);
          tooltip = { ...tooltip, visible: false };
        });

      if (animate) {
        dots.attr('opacity', 0)
          .transition()
          .delay(lineDelay + LINE_DUR * 0.75)
          .duration(300)
          .attr('opacity', 1);
      }
    });

    g.append('g')
      .attr('transform', `translate(0,${iH})`)
      .call(d3.axisBottom(x).tickFormat(d3.format('d')).ticks(allYears.length))
      .call(gg => gg.select('.domain').attr('stroke', '#d2d2d7'))
      .call(gg => gg.selectAll('text').attr('fill', '#86868b').attr('font-size', 12))
      .call(gg => gg.selectAll('.tick line').attr('stroke', '#d2d2d7'));

    g.append('g')
      .call(d3.axisLeft(y).ticks(5).tickFormat(d => `${(d * 100).toFixed(0)}%`))
      .call(gg => gg.select('.domain').attr('stroke', '#d2d2d7'))
      .call(gg => gg.selectAll('text').attr('fill', '#86868b').attr('font-size', 12))
      .call(gg => gg.selectAll('.tick line').attr('stroke', '#d2d2d7'));

    g.append('text')
      .attr('transform', 'rotate(-90)').attr('y', -58).attr('x', -iH / 2)
      .attr('text-anchor', 'middle').attr('fill', '#86868b').attr('font-size', 12)
      .text('Missing Report Rate');
  }
</script>

<div class="wrap" bind:this={containerEl}>
  <svg bind:this={svgEl}></svg>

  <div class="legend">
    {#each legendItems as item}
      <div class="legend-item">
        <span class="swatch" style="background:{item.color}"></span>
        <span class="label">{item.sector}</span>
        <span class="count">({item.n.toLocaleString()})</span>
      </div>
    {/each}
  </div>

  {#if tooltip.visible}
    <div
      class="tt"
      style="left:{Math.min(tooltip.x + 16, containerEl.clientWidth - 215)}px; top:{tooltip.y - 12}px"
    >
      <div class="tt-sector">{tooltip.sector}</div>
      <div class="tt-year">{tooltip.year}</div>
      <div class="tt-rate">{(tooltip.rate * 100).toFixed(1)}% of reports missing</div>
      <div class="tt-n">{tooltip.n.toLocaleString()} facilities in sector</div>
    </div>
  {/if}
</div>

<style>
  .wrap { position: relative; width: 100%; }
  svg { display: block; width: 100%; }

  .legend { display: flex; flex-wrap: wrap; gap: 8px 20px; padding: 10px 0 2px; }
  .legend-item { display: flex; align-items: center; gap: 6px; }
  .swatch { display: inline-block; width: 20px; height: 3px; border-radius: 2px; flex-shrink: 0; }
  .label { font-size: 12px; color: #6e6e73; }
  .count { font-size: 11px; color: #86868b; }

  .tt {
    position: absolute; background: #ffffff; border: 1px solid #e8e8ed;
    border-radius: 12px; padding: 11px 15px; pointer-events: none;
    font-size: 13px; color: #1d1d1f; width: 202px;
    box-shadow: 0 4px 24px rgba(0,0,0,.10), 0 1px 4px rgba(0,0,0,.06); z-index: 20;
  }
  .tt-sector { font-weight: 600; font-size: 14px; margin-bottom: 3px; }
  .tt-year { color: #0071e3; font-size: 12px; margin-bottom: 7px; font-weight: 500; }
  .tt-rate { color: #ff3b30; font-weight: 600; font-size: 13px; margin-bottom: 4px; }
  .tt-n { color: #86868b; font-size: 11px; }
</style>
