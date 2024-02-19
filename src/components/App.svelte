<script>
    import { onMount } from 'svelte';
    import * as d3 from 'd3';
    import * as topojson from 'topojson-client';
    

    let county_data = [];
    let state_tot_cases = [];
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

        const res1 = await fetch('state_tot_cases.csv'); 
        const csv1 = await res1.text();
        state_tot_cases = d3.csvParse(csv1, d3.autoType);
        console.log(state_tot_cases);

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

        // const maxStateCases = 12251820;
        const stateColorScale = d3.scaleQuantize([0.17, 0.41], d3.schemeBlues[5]);

        let casesByState = {}; // This will map state abbreviations to total cases
        state_tot_cases.forEach(d => {
     
            casesByState[d.state_full] = d.percent_cases;
        });

        const states = g.append("g")
            .attr("fill", "#444")
            .attr("cursor", "pointer")
            .selectAll("path")
            .data(topojson.feature(us, us.objects.states).features)
            .join("path")
                .attr("fill", d => {
                    const state_name = d.properties.name; 
        
                    const cases = casesByState[state_name]; // Default to 0 if no data available
                    return stateColorScale(cases);
                })
                .on("click", clicked)
                .attr("d", path)
                .join("path")
                .attr("d", path)
                .attr("class", "state")
                .on("click", clicked)
                .on("mouseover", handleMouseOver) // Add this event listener
                .on("mouseout", handleMouseOut);  // Add this event listener


        function handleMouseOver(event, d) {
            const [x, y] = path.centroid(d); // Get the centroid of the state path.
            const stateName = d.properties.name; // Get the state name from the TopoJSON properties.
            const cases = casesByState[stateName] || 0; // Get the cases or default to 0 if not found.

            d3.select(this)
                .attr("fill", "#fffea8") // Make the stroke color black

            const infoBoxGroup = g.append("g")
                .attr("class", "info-box-group")
                .attr("transform", `translate(${x}, ${y})`)
                .style("pointer-events", "none"); // Ignore mouse events

            // Append a rectangle to act as the background box.
            infoBoxGroup.append("rect")
                .attr("x", 25) // Position the box centered around the centroid; adjust as needed.
                .attr("y", -35) // Position the box above the centroid; adjust as needed.
                .attr("width", 100) // Set the width of the box.
                .attr("height", 50) // Set the height of the box.
                .attr("fill", "white") // Set the fill color of the box.
                .attr("stroke", "black") // Set the stroke color of the box.
                .attr("class", "state-info-box"); // Add a class for styling.

            // Append text to show the state name inside the box.
            infoBoxGroup.append("text")
                .attr("x", 75) // Center the text horizontally in the box.
                .attr("y", -15) // Position the text in the box; adjust as needed.
                .attr("text-anchor", "middle") // Center the text.
                .text(stateName)
                .attr("class", "state-name-text") // Add a class for styling.
                .style('font-weight', 'bold');

            // Append another text to show the percentage of cases inside the box.
            infoBoxGroup.append("text")
                .attr("x", 75) // Center the text horizontally in the box.
                .attr("y", 0) // Position the text in the box; adjust as needed.
                .attr("text-anchor", "middle") // Center the text.
                .text(`${cases.toFixed(2)}% cases`) // Display the percentage of cases.
                .attr("class", "state-cases-text"); // Add a class for styling.
        }

        function handleMouseOut(event, d) {
            // Remove the group with the box and text when the mouse leaves the state path.
            g.selectAll(".info-box-group").remove();

            // Reset the fill color to its original state
            const originalFillColor = determineOriginalFillColor(d); // Implement this function based on your logic
            d3.select(this)
                .attr("fill", originalFillColor) // Use the original fill color       
        }

        function determineOriginalFillColor(d) {
        // Example for a choropleth map where color is determined by data
            const cases = casesByState[d.properties.name]; // Assuming casesByState maps state names to data
            return stateColorScale(cases); // Assuming stateColorScale is your d3 scale for coloring states
        }


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

        const legendHeight = 200; // Total height of the legend
        const legendWidth = 20; // Width of each legend item
        const legendMargin = 10; // Margin around the legend
        const legendNumBlocks = stateColorScale.range().length; // Number of blocks in the legend
        const legendBlockHeight = legendHeight / legendNumBlocks; // Height of each block

        // Define the legend group
        const legend = svg.append('g')
            .attr('id', 'legend')
            .attr('transform', `translate(${width - legendWidth - legendMargin}, ${height - legendHeight - legendMargin})`);

        // Add colored rectangles for each segment of the legend

        const [x_0, x_1, x_2, x_3, x_4] = stateColorScale.range()

        legend.selectAll('rect')
            .data([x_4, x_3, x_2, x_1, x_0])
            .enter().append('rect')
                .attr('x', 0)
                .attr('y', (d, i) => i * legendBlockHeight)
                .attr('width', legendWidth)
                .attr('height', legendBlockHeight)
                .attr('fill', d => d);


        // legend.append('text')
        //     .attr('class', 'legend-title')
        //     .attr('x', legendWidth/2)
        //     .attr('y', 15) // Position the title above the legend rectangles
        //     .style('text-anchor', 'middle')
        //     .style('font-weight', 'bold')
        //     .text('Percent Cases'); // Title text

        // Add text labels for each segment of the legend
        const legendScale = d3.scaleLinear()
            .domain(stateColorScale.domain())
            .range([legendHeight, 0]);

        const legendAxis = d3.axisRight(legendScale)
            .ticks(legendNumBlocks)
            .tickFormat(d => `${d}%`);


    
        legend.call(legendAxis)
            .selectAll('text')
            .style('text-anchor', 'start')
            .attr('x', 5)
            .attr('dy', 0)
            .attr('transform', 'translate(20,0)');
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

    .state-info-box {
    fill: #fff;
    stroke: #000;
    opacity: 0.8;
    }

    .state-name-text, .state-cases-text {
        font-size: 12px;
        fill:
         #000;
    }

    .legend-title {
    fill: #000;
    font-size: 14px;
    font-weight: bold;
    }

/* Add any additional styling as needed */

</style>