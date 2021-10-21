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
    const svg = d3 // create a d3 selection of an element
      .select(el) // el variable holds the div where this whole visualization component gets injected
      .append('svg') // insert an SVG element
      .attr('viewBox', [0, 0, width * 1.4, height * 1.4]); // https://developer.mozilla.org/en-US/docs/Web/SVG/Attribute/viewBox

    const tiler = d3Tile.tile().extent([
      // d3.tile() generates coordinates x/y/z e.g. 1/1/2
      [0, 0],
      [width, height],
    ]);

    // understanding the zoom part might be most complex - go through this to start with the basics : https://observablehq.com/@enjalot/d3-zoom-edumix
    // Setup the zoom
    const zoom = d3 // d3.zoom is for panning and zooming
      .zoom()
      .scaleExtent([1 << 8, 1 << 22]) // https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Left_shift, so 1 << 8 is 256 ; 1 << 22 is 4194304
      .extent([
        [0, 0],
        [width, height], // the above matrix is 'mapped' to the screen shape
      ])
      .on('zoom', (event) => zoomed(event.transform)); // attaching event listener

    const tileGroup = svg //svg is like collections of groups of elements
      .append('g')
      .attr('pointer-events', 'none')
      .attr('font-family', 'var(--sans-serif)')
      .attr('font-size', 16);

    let tile = tileGroup.selectAll('g'); // handle for tile

    // svg.call(zoom) sets up the mouse/touch listeners on the SVG element for the zoom

    // this sets the initial zoom to the "identity" transform
    // which is just scale of 1 and no translation in x or y.
    // and then translate to

    svg.call(zoom).call(
      zoom.transform,
      d3.zoomIdentity.translate(width >> 1, height >> 1).scale(1 << 10) // https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Right_shift...so 3000 >> 1 is 1500
    );

    function zoomed(transform) {
      // This is what actually changes what we see, the "transform"
      const tiles = tiler(transform); // tiler function generates tiles, see above
      console.log(tiles);

      // When we set translate() on an SVG element, we're moving around its coordinate system
      // https://developer.mozilla.org/en-US/docs/Web/SVG/Attribute/transform#translate

      tileGroup.attr(
        'transform',
        `
      scale(${tiles.scale})
      translate(${tiles.translate.join(',')})
    `
      );

      tile = tile
        .data(tiles, (d) => d) //attaching the generated tiles
        .join(
          (
            enter // enter is a special term to say that new data is available
          ) =>
            enter
              .append('g') // create a new group per item and transform it to it's position
              .attr(
                'transform',
                ([x, y]) => `translate(${x}, ${y}) scale(${1 / 256})`
              )
              .call((g) =>
                g
                  .append('rect') // add rectangle under another <g></g> tag
                  .attr('fill', (d) => d3.interpolateRainbow(hilbert(...d))) // colormap - https://corte.si/posts/code/hilbert/swatches/
                  .attr('fill-opacity', 0.5)
                  .attr('stroke', 'black')
                  .attr('width', 256)
                  .attr('height', 256)
              )
              .call(
                (g) =>
                  g
                    .append('text')
                    .attr('x', '0.4em')
                    .attr('y', '1.2em')
                    .text((d) => d.join('/')) // this creates the text on each tile e.g. 1/3/2
              )
        );
    }

    return svg.node();
  }

  function hilbert(x, y, z) {
    // don't worry about this colormap generating code
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
