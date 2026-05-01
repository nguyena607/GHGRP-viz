<script>
  import MissingnessOverTime from './lib/MissingnessOverTime.svelte';
  import EntrantsExits from './lib/EntrantsExits.svelte';
  import FacilityHeatmap from './lib/FacilityHeatmap.svelte';
  import SectorMissingness from './lib/SectorMissingness.svelte';
</script>

<main>
  <header>
    <div class="header-inner">
      <div class="eyebrow">EPA GHGRP &middot; 2011&ndash;2023</div>
      <h1>Who Stopped Reporting?</h1>
      <p class="subtitle">
        The EPA's Greenhouse Gas Reporting Program requires roughly 8,700 large
        industrial facilities to disclose their emissions every year. For a few years,
        compliance improved. Then, starting in 2015, hundreds of facilities went silent
        — and stayed that way. By 2023, more than 1 in 4 tracked facilities filed no
        report at all. Now the EPA has proposed eliminating the program entirely.
      </p>
      <p class="subtitle-follow">
        This is the story of how a dataset began disappearing before it was cancelled.
      </p>
    </div>
  </header>

  <div class="sections">

    <!-- ── Part I: Guided narrative ──────────────────────────────────── -->

    <section class="narrative">
      <div class="sec-head">
        <div class="step-label">01</div>
        <h2>A gap that keeps growing</h2>
        <p>
          Reporting compliance actually <em>improved</em> in the program's first
          three years, reaching its best point in 2014 when fewer than 16% of
          facilities went unreported. Then, in 2016, nearly a quarter fell silent
          — and the rate has barely budged since. Watch the red band: it narrows,
          then widens, and never recovers. By 2023, more than 2,200 facilities
          a year are producing no data at all.
        </p>
      </div>
      <div class="card">
        <MissingnessOverTime />
      </div>
    </section>

    <section class="narrative">
      <div class="sec-head">
        <div class="step-label">02</div>
        <h2>Something broke in 2015</h2>
        <p>
          Year to year, some facilities enter the program for the first time while
          others report for the last. For most of the program's history, those flows
          were roughly balanced. Then 2015 shattered the equilibrium: 756 facilities
          filed their final report — more than four times any prior year. Entrants
          have been declining ever since. The program has been contracting in slow
          motion for nearly a decade.
        </p>
      </div>
      <div class="card">
        <EntrantsExits />
      </div>
    </section>

    <!-- ── Pivot ──────────────────────────────────────────────────────── -->

    <div class="pivot">
      <div class="pivot-line"></div>
      <span class="pivot-label">Explore the data</span>
      <div class="pivot-line"></div>
    </div>

    <p class="pivot-intro">
      The charts above trace the broad arc of the problem. Below, you can dig
      into the detail — which specific facilities have gone silent, for how long,
      and which industries are driving the trend.
    </p>

    <!-- ── Part II: Open / exploratory ───────────────────────────────── -->

    <section>
      <div class="sec-head">
        <h2>Every facility, every year</h2>
        <p>
          Each row is one of the 8,737 tracked facilities; each column is one
          reporting year. A colored cell means a report was filed; a faded cell
          means silence. Facilities are grouped by sector and sorted by the number
          of missing years — the worst offenders rise to the top of each group.
          Scroll down to see just how many facilities have been dark for years
          on end.
        </p>
      </div>
      <div class="card heatmap-card">
        <FacilityHeatmap />
      </div>
    </section>

    <section>
      <div class="sec-head">
        <h2>Which sectors are failing?</h2>
        <p>
          Zoom out from individual facilities to compare industries. Oil &amp; Gas
          stands apart: nearly half of its facilities are missing in any given year.
          Power plants hold comparatively steady. Some sectors show slow, steady
          deterioration; others spiked after 2015 and never recovered. Hover over
          the lines to track a specific sector's trajectory.
        </p>
      </div>
      <div class="card">
        <SectorMissingness />
      </div>
    </section>

  </div>

  <footer>
    <p>
      Data source: EPA Greenhouse Gas Reporting Program &middot;
      <em>ghgp_data_by_year_2023.xlsx</em>
    </p>
  </footer>
</main>

<style>
  :global(*, *::before, *::after) {
    box-sizing: border-box;
    margin: 0;
    padding: 0;
  }

  :global(body) {
    background: #ffffff;
    color: #1d1d1f;
    font-family: -apple-system, 'Helvetica Neue', Helvetica, Arial, sans-serif;
    line-height: 1.6;
    -webkit-font-smoothing: antialiased;
  }

  :global(svg text) {
    font-family: -apple-system, 'Helvetica Neue', Helvetica, Arial, sans-serif;
  }

  /* ── Header ──────────────────────────────────────────────────────────── */
  header {
    background: #ffffff;
    border-bottom: 1px solid #e8e8ed;
    padding: 5rem 1.5rem 4.5rem;
    text-align: center;
  }

  .header-inner {
    max-width: 640px;
    margin: 0 auto;
  }

  .eyebrow {
    font-size: 0.8rem;
    font-weight: 600;
    letter-spacing: 0.04em;
    color: #86868b;
    text-transform: uppercase;
    margin-bottom: 1rem;
  }

  h1 {
    font-size: clamp(2.4rem, 6vw, 4rem);
    font-weight: 700;
    line-height: 1.05;
    letter-spacing: -0.025em;
    color: #1d1d1f;
    margin-bottom: 1.25rem;
  }

  .subtitle {
    font-size: 1.05rem;
    color: #6e6e73;
    line-height: 1.75;
    font-weight: 400;
    margin-bottom: 0.9rem;
  }

  .subtitle-follow {
    font-size: 1rem;
    color: #1d1d1f;
    font-weight: 500;
    font-style: italic;
    line-height: 1.6;
  }

  /* ── Page sections ────────────────────────────────────────────────────── */
  .sections {
    max-width: 1040px;
    margin: 0 auto;
    padding: 4rem 1.5rem 5rem;
    display: flex;
    flex-direction: column;
    gap: 4rem;
  }

  section {
    display: flex;
    flex-direction: column;
    gap: 1.1rem;
  }

  /* ── Narrative sections (Part I) ─────────────────────────────────────── */
  section.narrative .sec-head p {
    max-width: 560px;
    font-size: 0.97rem;
    line-height: 1.72;
    color: #3a3a3c;
  }

  .step-label {
    font-size: 0.72rem;
    font-weight: 700;
    letter-spacing: 0.1em;
    color: #aeaeb2;
    text-transform: uppercase;
    margin-bottom: 0.4rem;
  }

  /* ── Sec-head shared ─────────────────────────────────────────────────── */
  .sec-head h2 {
    font-size: 1.3rem;
    font-weight: 600;
    letter-spacing: -0.015em;
    color: #1d1d1f;
    margin-bottom: 0.4rem;
  }

  .sec-head p {
    color: #6e6e73;
    font-size: 0.93rem;
    max-width: 600px;
    line-height: 1.6;
  }

  /* ── Martini-glass pivot ─────────────────────────────────────────────── */
  .pivot {
    display: flex;
    align-items: center;
    gap: 1.25rem;
    margin: 0.5rem 0 -1rem;
  }

  .pivot-line {
    flex: 1;
    height: 1px;
    background: #e8e8ed;
  }

  .pivot-label {
    font-size: 0.72rem;
    font-weight: 700;
    letter-spacing: 0.1em;
    text-transform: uppercase;
    color: #aeaeb2;
    white-space: nowrap;
  }

  .pivot-intro {
    font-size: 0.93rem;
    color: #6e6e73;
    max-width: 560px;
    line-height: 1.7;
    margin-top: -1rem;
  }

  /* ── Card ─────────────────────────────────────────────────────────────── */
  .card {
    padding: 0.5rem 0 0;
    overflow: hidden;
  }

  .heatmap-card {
    padding: 0.5rem 0 0;
    overflow-x: auto;
  }

  /* ── Footer ───────────────────────────────────────────────────────────── */
  footer {
    text-align: center;
    padding: 2.5rem 1.5rem;
    border-top: 1px solid #e8e8ed;
    color: #86868b;
    font-size: 0.82rem;
  }

  footer em {
    font-style: normal;
    font-family: 'SF Mono', 'SFMono-Regular', ui-monospace, Menlo, monospace;
    color: #6e6e73;
  }
</style>
