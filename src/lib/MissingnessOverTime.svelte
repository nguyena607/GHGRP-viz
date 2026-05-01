<script>
  import { onMount } from 'svelte';
  import * as d3 from 'd3';

  let svgEl;
  let containerEl;
  let tooltip = { visible: false, x: 0, y: 0, year: 0, reported: 0, missing: 0, rate: 0 };

  const HEIGHT = 320;
  const M = { top: 24, right: 28, bottom: 46, left: 68 };
  const C_REPORTED = '#34c759';
  const C_MISSING  = '#ff3b30';

  onMount(async () => {
    const data = await fetch('/data/missingness_by_year.json').then(r => r.json());

    let animating = false;
    let animated  = false;
    let io;

    draw(data, false);

    function triggerAnimation() {
      if (animated) return;
      animated  = true;
      animating = true;
      draw(data, true);
      setTimeout(() => { animating = false; }, 2000);
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

    const ro = new ResizeObserver(() => { if (!animating) draw(data, false); });
    ro.observe(containerEl);

    return () => { io?.disconnect(); ro.disconnect(); };
  });

  function draw(data, animate = false) {
    if (!svgEl || !containerEl) return;
    const W  = containerEl.clientWidth;
    const iW = W - M.left - M.right;
    const iH = HEIGHT - M.top - M.bottom;

    const svg = d3.select(svgEl).attr('width', W).attr('height', HEIGHT);
    svg.selectAll('*').remove();

    // Clip path — width animates 0 → full, revealing chart left-to-right
    const clipRect = svg.append('defs')
      .append('clipPath').attr('id', 'mot-clip')
      .append('rect')
      .attr('x', -4).attr('y', -M.top)
      .attr('height', HEIGHT + M.top)
      .attr('width', animate ? 0 : iW + M.right + 4);

    if (animate) {
      clipRect.transition()
        .duration(1500)
        .ease(d3.easeQuadOut)
        .attr('width', iW + M.right + 4);
    }

    const g = svg.append('g').attr('transform', `translate(${M.left},${M.top})`);

    const x = d3.scaleLinear().domain(d3.extent(data, d => d.year)).range([0, iW]);
    const y = d3.scaleLinear().domain([0, d3.max(data, d => d.total)]).range([iH, 0]).nice();

    // Grid
    g.append('g')
      .call(d3.axisLeft(y).ticks(5).tickSize(-iW).tickFormat(''))
      .call(gg => gg.select('.domain').remove())
      .call(gg => gg.selectAll('line').attr('stroke', '#f0f0f0'));

    // Axes
    g.append('g')
      .attr('transform', `translate(0,${iH})`)
      .call(d3.axisBottom(x).tickFormat(d3.format('d')).ticks(data.length))
      .call(gg => gg.select('.domain').attr('stroke', '#d2d2d7'))
      .call(gg => gg.selectAll('text').attr('fill', '#86868b').attr('font-size', 12))
      .call(gg => gg.selectAll('.tick line').attr('stroke', '#d2d2d7'));

    g.append('g')
      .call(d3.axisLeft(y).ticks(5).tickFormat(d => d3.format(',')(d)))
      .call(gg => gg.select('.domain').attr('stroke', '#d2d2d7'))
      .call(gg => gg.selectAll('text').attr('fill', '#86868b').attr('font-size', 12))
      .call(gg => gg.selectAll('.tick line').attr('stroke', '#d2d2d7'));

    g.append('text')
      .attr('transform', 'rotate(-90)').attr('y', -54).attr('x', -iH / 2)
      .attr('text-anchor', 'middle').attr('fill', '#86868b').attr('font-size', 12)
      .text('Facilities');

    // Clipped group — areas + strokes sweep left-to-right inside the clip
    const cg = g.append('g').attr('clip-path', 'url(#mot-clip)');

    const stack  = d3.stack().keys(['reported', 'missing']);
    const layers = stack(data);
    const colors = [C_REPORTED, C_MISSING];

    const area = d3.area()
      .x(d => x(d.data.year)).y0(d => y(d[0])).y1(d => y(d[1]))
      .curve(d3.curveMonotoneX);

    cg.selectAll('.layer').data(layers).join('path')
      .attr('class', 'layer').attr('d', area)
      .attr('fill', (d, i) => colors[i])
      .attr('opacity', (d, i) => i === 0 ? 0.16 : 0.12);

    const lineTop = d3.line()
      .x(d => x(d.data.year)).y(d => y(d[1])).curve(d3.curveMonotoneX);

    layers.forEach((layer, i) => {
      cg.append('path').datum(layer)
        .attr('fill', 'none').attr('stroke', colors[i])
        .attr('stroke-width', 2).attr('d', lineTop);
    });

    // Crosshair line
    const vLine = g.append('line')
      .attr('stroke', '#0071e3').attr('stroke-width', 1)
      .attr('stroke-dasharray', '4,3')
      .attr('y1', 0).attr('y2', iH).attr('opacity', 0)
      .attr('pointer-events', 'none');

    // Transparent overlay rect to capture mouse events for the crosshair
    const bisect = d3.bisector(d => d.year).center;

    g.append('rect')
      .attr('x', 0).attr('y', 0).attr('width', iW).attr('height', iH)
      .attr('fill', 'none').attr('pointer-events', 'all')
      .attr('cursor', 'crosshair')
      .on('mousemove', function (event) {
        const [mx] = d3.pointer(event);
        const idx  = bisect(data, x.invert(mx));
        const d    = data[Math.max(0, Math.min(idx, data.length - 1))];
        vLine.attr('x1', x(d.year)).attr('x2', x(d.year)).attr('opacity', 1);
        const [px, py] = d3.pointer(event, containerEl);
        tooltip = { visible: true, x: px, y: py, year: d.year, reported: d.reported, missing: d.missing, rate: d.missing_rate };
      })
      .on('mouseleave', function () {
        vLine.attr('opacity', 0);
        tooltip = { ...tooltip, visible: false };
      });
  }
</script>

<div class="wrap" bind:this={containerEl}>
  <div class="legend">
    <div class="legend-item">
      <span class="swatch" style="background:{C_REPORTED}"></span>
      <span class="label">Reported</span>
    </div>
    <div class="legend-item">
      <span class="swatch" style="background:{C_MISSING}"></span>
      <span class="label">Missing</span>
    </div>
  </div>

  <svg bind:this={svgEl}></svg>

  {#if tooltip.visible}
    <div
      class="tt"
      style="left:{Math.min(tooltip.x + 16, containerEl.clientWidth - 185)}px; top:{tooltip.y - 12}px"
    >
      <div class="tt-year">{tooltip.year}</div>
      <div class="tt-row">
        <span class="dot" style="background:#34c759"></span>
        Reported <strong>{tooltip.reported.toLocaleString()}</strong>
      </div>
      <div class="tt-row">
        <span class="dot" style="background:#ff3b30"></span>
        Missing <strong>{tooltip.missing.toLocaleString()}</strong>
      </div>
      <div class="tt-rate">{(tooltip.rate * 100).toFixed(1)}% missing</div>
    </div>
  {/if}

</div>

<style>
  .wrap { position: relative; width: 100%; }
  svg { display: block; width: 100%; }

  .legend { display: flex; align-items: center; gap: 16px; margin-bottom: 10px; flex-wrap: wrap; }
  .legend-item { display: flex; align-items: center; gap: 6px; }
  .swatch { display: inline-block; width: 11px; height: 11px; border-radius: 3px; flex-shrink: 0; }
  .label { font-size: 12px; color: #6e6e73; }

  /* ── Tooltip ──────────────────────────────────────────────────────── */
  .tt {
    position: absolute; background: #ffffff; border: 1px solid #e8e8ed;
    border-radius: 12px; padding: 11px 15px; pointer-events: none;
    font-size: 13px; color: #1d1d1f; width: 172px;
    box-shadow: 0 4px 24px rgba(0,0,0,.10), 0 1px 4px rgba(0,0,0,.06); z-index: 20;
  }
  .tt-year { font-weight: 600; font-size: 14px; color: #0071e3; margin-bottom: 8px; }
  .tt-row { display: flex; align-items: center; gap: 6px; color: #6e6e73; margin-bottom: 4px; font-size: 12px; }
  .tt-row strong { margin-left: auto; color: #1d1d1f; font-weight: 500; }
  .dot { width: 8px; height: 8px; border-radius: 2px; flex-shrink: 0; }
  .tt-rate { margin-top: 9px; padding-top: 9px; border-top: 1px solid #e8e8ed; color: #ff3b30; font-weight: 600; font-size: 12px; }

</style>
