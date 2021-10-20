<script>
  import { onMount } from 'svelte';
  import * as d3 from 'd3';
  import * as d3Tile from 'd3-tile';

  const height = 3000;
  const width = 3000;

  let el;

  onMount(() => {
    init();
    /* make a class with following methods
    update();
    destroy();
    */
  });

  function init() {
    const svg = d3
      .select(el)
      .append('svg')
      .attr('viewBox', [0, 0, width * 1.4, height * 1.4]);

    const tiler = d3Tile.tile().extent([
      [0, 0],
      [width, height],
    ]);

    const zoom = d3
      .zoom()
      .scaleExtent([1 << 8, 1 << 22])
      .extent([
        [0, 0],
        [width, height],
      ])
      .on('zoom', (event) => zoomed(event.transform));

    const tileGroup = svg
      .append('g')
      .attr('pointer-events', 'none')
      .attr('font-family', 'var(--sans-serif)')
      .attr('font-size', 16);

    let tile = tileGroup.selectAll('g');

    svg
      .call(zoom)
      .call(
        zoom.transform,
        d3.zoomIdentity.translate(width >> 1, height >> 1).scale(1 << 10)
      );

    function zoomed(transform) {
      const tiles = tiler(transform);
      console.log(tiles);

      tileGroup.attr(
        'transform',
        `
      scale(${tiles.scale})
      translate(${tiles.translate.join(',')})
    `
      );

      tile = tile
        .data(tiles, (d) => d)
        .join((enter) =>
          enter
            .append('g')
            .attr(
              'transform',
              ([x, y]) => `translate(${x}, ${y}) scale(${1 / 256})`
            )
            .call((g) =>
              g
                .append('rect')
                .attr('fill', (d) => d3.interpolateRainbow(hilbert(...d)))
                .attr('fill-opacity', 0.5)
                .attr('stroke', 'black')
                .attr('width', 256)
                .attr('height', 256)
            )
            .call((g) =>
              g
                .append('text')
                .attr('x', '0.4em')
                .attr('y', '1.2em')
                .text((d) => d.join('/'))
            )
        );
    }

    return svg.node();
  }

  function hilbert(x, y, z) {
    let n = 1 << z,
      rx,
      ry,
      s,
      d = 0;
    for (s = n >> 1; s > 0; s >>= 1) {
      rx = (x & s) > 0;
      ry = (y & s) > 0;
      d += s * s * ((3 * rx) ^ ry);
      [x, y] = rot(n, x, y, rx, ry);
    }
    return d / (1 << (z * 2));
  }

  function rot(n, x, y, rx, ry) {
    if (!ry) {
      if (rx) {
        x = n - 1 - x;
        y = n - 1 - y;
      }
      return [y, x];
    }
    return [x, y];
  }
</script>

<div bind:this={el} class="chart" />

<style>
  .chart :global(div) {
    font: 10px sans-serif;
    background-color: steelblue;
    text-align: right;
    padding: 3px;
    margin: 1px;
    color: white;
    height: 600px;
    width: 600px;
  }
</style>
