
<script>
    import { onMount } from 'svelte';
    import * as d3 from 'd3';
    import * as topojson from 'topojson-client';
    

    let county_data = [];
    let us;
    let svg;
    let zoom;
    let initialTransform = d3.zoomIdentity;
    let isZoomed = false;
    let lastClicked = null;



    onMount(async () => {
        const res = await fetch('county_data.csv'); 
        const csv = await res.text();
        county_data = d3.csvParse(csv, d3.autoType)
        console.log(county_data);
        const resJSON = await fetch('states-albers-10m.json');
        us = await resJSON.json();
        console.log(us);

        renderChart();
    });
 
    function renderChart() {


        const path = d3.geoPath();

        const width = 1000;
        const height = 640;

        const zoom = d3.zoom()
            .scaleExtent([1, 8])
            .on("zoom", zoomed);

        svg = d3.select("svg")
            .attr("viewBox", [0, 0, width, height])
            .attr("width", width)
            .attr("height", height)
            .style("max-width", "100%")
            .style("height", "100%")
            .call(zoom);

        const g = svg.append("g");

        const states = g.append("g")
            .attr("fill", "#444")
            .attr("cursor", "pointer")
            .selectAll("path")
            .data(topojson.feature(us, us.objects.states).features)
            .join("path")
                .on("click", clicked)
                .attr("d", path);

        states.append("title")
            .text(d => d.properties.name);

        g.append("path")
            .attr("fill", "none")
            .attr("stroke", "white")
            .attr("stroke-linejoin", "round")
            .attr("d", path(topojson.mesh(us, us.objects.states, (a, b) => a !== b)));
        
        svg.on("dblclick", reset);

        function reset() {
            states.transition().style("fill", null);
            svg.transition().duration(750).call(
                zoom.transform,
                d3.zoomIdentity, // Reset zoom
            );

            isZoomed = false;
            lastClicked = null;
        }


        function clicked(event, d) {
            const [[x0, y0], [x1, y1]] = path.bounds(d);
            event.stopPropagation();

            // Check if we're zooming in on the same element or if the map is already zoomed in
            if (isZoomed && lastClicked === this) {
                // Reset to initial zoom
                reset();
            } else {
                // Proceed with zooming in
                states.transition().style("fill", null);
                d3.select(this).transition().style("fill", "green");
                svg.transition().duration(750).call(
                    zoom.transform,
                    d3.zoomIdentity
                        .translate(width / 2, height / 2)
                        .scale(Math.min(8, 0.9 / Math.max((x1 - x0) / width, (y1 - y0) / height)))
                        .translate(-(x0 + x1) / 2, -(y0 + y1) / 2),
                    d3.pointer(event, svg.node())
                );

                isZoomed = true;
                lastClicked = this;
            }
        }


        function zoomed(event) {
            const {transform} = event;
            g.attr("transform", transform);
            g.attr("stroke-width", 1 / transform.k);
        }
    }
</script>

<main>
    <svg></svg> <!-- SVG element for D3 to target -->
</main>

<style>
    main {
        text-align: center;
        display: flex;
        justify-content: center;
        align-items: center;
        height: 100vh; 
        width: 100vw; 
    }
    svg {
        margin: auto; /* Center the SVG horizontally */
        display: block;
        width: 100%; /* Make SVG responsive */
        height: auto; /* Maintain aspect ratio */
    }

</style>



