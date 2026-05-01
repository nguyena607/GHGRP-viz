<script>
  import { onMount } from 'svelte';
  import * as d3 from 'd3';

  let svgEl;
  let containerEl;
  let tooltip = { visible: false, x: 0, y: 0, year: 0, entrants: 0, exits: 0, net: 0 };

  const HEIGHT = 320;
  const M = { top: 24, right: 28, bottom: 46, left: 68 };
  const COLORS = { entrants: '#007aff', exits: '#ff9500' };

  onMount(async () => {
    const data = await fetch('/data/entrants_exits.json').then(r => r.json());

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
    const g = svg.append('g').attr('transform', `translate(${M.left},${M.top})`);

    const keys = ['entrants', 'exits'];

    const x0 = d3.scaleBand()
      .domain(data.map(d => d.year)).range([0, iW])
      .paddingInner(0.28).paddingOuter(0.1);

    const x1 = d3.scaleBand().domain(keys).range([0, x0.bandwidth()]).padding(0.1);

    const y = d3.scaleLinear()
      .domain([0, d3.max(data, d => Math.max(d.entrants, d.exits)) * 1.1])
      .range([iH, 0]).nice();

    g.append('g')
      .call(d3.axisLeft(y).ticks(5).tickSize(-iW).tickFormat(''))
      .call(gg => gg.select('.domain').remove())
      .call(gg => gg.selectAll('line').attr('stroke', '#f0f0f0'));

    const yearGroups = g.selectAll('.yg').data(data).join('g')
      .attr('class', 'yg')
      .attr('transform', d => `translate(${x0(d.year)},0)`);

    // Bars grow up from the baseline, staggered left-to-right by year
    const bars = yearGroups.selectAll('rect')
      .data((d, yi) => keys.map(k => ({ key: k, value: d[k], parent: d, yi })))
      .join('rect')
      .attr('x', d => x1(d.key))
      .attr('width', x1.bandwidth())
      .attr('fill', d => COLORS[d.key])
      .attr('rx', 3)
      .attr('opacity', 0.88)
      .on('mousemove', function (event, d) {
        d3.select(this).transition().duration(80).attr('opacity', 1);
        const [px, py] = d3.pointer(event, containerEl);
        tooltip = { visible: true, x: px, y: py, year: d.parent.year, entrants: d.parent.entrants, exits: d.parent.exits, net: d.parent.net };
      })
      .on('mouseleave', function () {
        d3.select(this).transition().duration(80).attr('opacity', 0.88);
        tooltip = { ...tooltip, visible: false };
      });

    if (animate) {
      bars.attr('y', iH).attr('height', 0)
        .transition()
        .delay(d => d.yi * 55)
        .duration(650)
        .ease(d3.easeCubicOut)
        .attr('y', d => y(d.value))
        .attr('height', d => iH - y(d.value));
    } else {
      bars.attr('y', d => y(d.value)).attr('height', d => iH - y(d.value));
    }

    g.append('line')
      .attr('x1', 0).attr('x2', iW)
      .attr('y1', y(0)).attr('y2', y(0))
      .attr('stroke', '#d2d2d7').attr('stroke-width', 1);

    g.append('g')
      .attr('transform', `translate(0,${iH})`)
      .call(d3.axisBottom(x0).tickFormat(d3.format('d')))
      .call(gg => gg.select('.domain').attr('stroke', '#d2d2d7'))
      .call(gg => gg.selectAll('text').attr('fill', '#86868b').attr('font-size', 12))
      .call(gg => gg.selectAll('.tick line').attr('stroke', '#d2d2d7'));

    g.append('g')
      .call(d3.axisLeft(y).ticks(5).tickFormat(d3.format(',d')))
      .call(gg => gg.select('.domain').attr('stroke', '#d2d2d7'))
      .call(gg => gg.selectAll('text').attr('fill', '#86868b').attr('font-size', 12))
      .call(gg => gg.selectAll('.tick line').attr('stroke', '#d2d2d7'));

    g.append('text')
      .attr('transform', 'rotate(-90)').attr('y', -54).attr('x', -iH / 2)
      .attr('text-anchor', 'middle').attr('fill', '#86868b').attr('font-size', 12)
      .text('Facilities');

    const legend = g.append('g').attr('transform', `translate(${iW - 168}, 4)`);
    [['New Entrants', 'entrants'], ['Last Reporters', 'exits']].forEach(([label, key], i) => {
      const row = legend.append('g').attr('transform', `translate(0,${i * 22})`);
      row.append('rect').attr('width', 12).attr('height', 12)
        .attr('fill', COLORS[key]).attr('opacity', 0.88).attr('rx', 3);
      row.append('text').attr('x', 18).attr('y', 10)
        .attr('fill', '#6e6e73').attr('font-size', 12).text(label);
    });
  }
</script>

<div class="wrap" bind:this={containerEl}>
  <svg bind:this={svgEl}></svg>

  {#if tooltip.visible}
    <div
      class="tt"
      style="left:{Math.min(tooltip.x + 16, containerEl.clientWidth - 185)}px; top:{tooltip.y}px; transform: translateY(calc(-100% - 12px))"
    >
      <div class="tt-year">{tooltip.year}</div>
      <div class="tt-row">
        <span class="dot" style="background:#007aff"></span>
        Entrants <strong>{tooltip.entrants.toLocaleString()}</strong>
      </div>
      <div class="tt-row">
        <span class="dot" style="background:#ff9500"></span>
        Exits <strong>{tooltip.exits.toLocaleString()}</strong>
      </div>
      <div class="tt-net" class:pos={tooltip.net >= 0} class:neg={tooltip.net < 0}>
        Net: {tooltip.net >= 0 ? '+' : ''}{tooltip.net.toLocaleString()}
      </div>
    </div>
  {/if}
</div>

<style>
  .wrap { position: relative; width: 100%; }
  svg { display: block; width: 100%; }

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
  .tt-net { margin-top: 9px; padding-top: 9px; border-top: 1px solid #e8e8ed; font-weight: 600; font-size: 13px; }
  .pos { color: #34c759; }
  .neg { color: #ff3b30; }
</style>
