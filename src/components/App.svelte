<h1 style ='text-align: center'> End of COVID-19: Multi-Perspective Summary</h1>

<p class='subtitle'> On January 30, 2023, the Biden Administration announced its intent to 
    end the national emergency and public health emergency declarations 
    on May 11, 2023, related to the COVID-19 pandemic. Here, we have come up with
    a visualization to help compare how severe the pandemic was for each state (when NOT zoomed in)
    and for each county (when zoomed in). 
</p >

<script>
    import { onMount } from 'svelte';
    import * as d3 from 'd3';
    import * as topojson from 'topojson-client';
    

    let county_data = [];
    let state_tot_cases = [];
    let us;
    let county;
    let svg;
    let zoom;
    let initialTransform = d3.zoomIdentity;
    let isZoomed = false;
    let lastClicked = null;


    // load data (county, state, topojson)
    onMount(async () => {
        // map covid data
        const res = await fetch('county_data_fip.csv'); 
        const csv = await res.text();
        county_data = d3.csvParse(csv, d3.autoType)
        console.log(county_data);

        const res1 = await fetch('state_tot_cases.csv'); 
        const csv1 = await res1.text();
        state_tot_cases = d3.csvParse(csv1, d3.autoType);
        console.log(state_tot_cases);

        // map data
        const resJSON = await fetch('counties-albers-10m.json');
        county = await resJSON.json();
        console.log(county);

        const resJSON1 = await fetch('states-albers-10m.json');
        us = await resJSON1.json();
        console.log(us);

        renderChart();
    });
 
    // function that creates map, includes all actions
    function renderChart() {

        const path = d3.geoPath();
        const width = 1000;
        const height = 640;

        //zoom in
        const zoom = d3.zoom()
            .scaleExtent([1, 8])
            .on("zoom", zoomed);

        // main us map
        svg = d3.select("svg")
            .attr("viewBox", [0, 0, width, height])
            .attr("width", width)
            .attr("height", height)
            .style('transform', 'scale(0.8)')
            .style("max-width", "100%")
            .style("height", "90%")
            .call(zoom);

        const g = svg.append("g");

        // color scale for state and state
        // const maxStateCases = 12251820;
        const stateColorScale = d3.scaleQuantize([0.17, 0.41], d3.schemeBlues[5]);
        const countyColorScale = d3.scaleQuantize([0.17, 0.41], d3.schemeBlues[5]);

        // map state abbreviations to total cases
        let casesByState = {}; 
        state_tot_cases.forEach(d => {
            casesByState[d.state_full] = d.percent_cases;
        });

        const states = g.append("g")
            .attr("fill", "#444")
            .attr("cursor", "pointer")
            .selectAll("path")
            .data(topojson.feature(us, us.objects.states).features) // load topodata to render us map
            .join("path")
                .attr("fill", d => {
                    const state_name = d.properties.name; 
                    const cases = casesByState[state_name]; 
                    return stateColorScale(cases);
                }) // fill color of each state according to scale
                .on("click", clicked)
                .attr("d", path)
                .join("path")
                .attr("d", path)
                .attr("class", "state")
                .on("click", clicked)
                .on("mouseover", handleMouseOver) 
                .on("mouseout", handleMouseOut); 


        
        function handleMouseOver(event, d) {
            const [x, y] = path.centroid(d); // Get the centroid of the state path.
            const stateName = d.properties.name; // Get the state name from the TopoJSON properties.
            const cases = casesByState[stateName] || 0; // Get the cases or default to 0 if not found.

            d3.select(this)
                .attr("fill", "#fffea8") // Make the stroke color black

            const infoBoxGroup = g.append("g")
                .attr("class", "info-box-group")
                .attr("transform", `translate(${x}, ${y})`)
                .style("pointer-events", "none"); // Ignore mouse events (moving on infobox)

            // Append a rectangle to act as the background box.
            infoBoxGroup.append("rect")
                .attr("x", 15) 
                .attr("y", -35) // Position the box above the centroid
                .attr("width", 140) // Set the width of the box.
                .attr("height", 50) // Set the height of the box.
                .attr("fill", "white") // Set the fill color of the box.
                .attr("stroke", "black") // Set the stroke color of the box.
                .attr("class", "state-info-box"); // a class for styling.

            // Append text to show the state name inside the box.
            infoBoxGroup.append("text")
                .attr("x", 85) // Center the text horizontally in the box.
                .attr("y", -15) // Position the text in the box
                .attr("text-anchor", "middle") // Center the text.
                .text(stateName)
                .attr("class", "state-name-text") // a class for styling.
                .style('font-weight', 'bold');

            // Append another text to show the percentage of cases inside the box.
            infoBoxGroup.append("text")
                .attr("x", 85) // Center the text horizontally in the box.
                .attr("y", 0) // Position the text in the box; adjust as needed.
                .attr("text-anchor", "middle") // Center the text.
                .text(`${(cases*100).toFixed(2)}% Population`) // Display the percentage of cases.
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
            const cases = casesByState[d.properties.name]; 
            return stateColorScale(cases); 
        }

        // create boundaries (outline of maps)
        g.append("path")
            .attr("fill", "none")
            .attr("stroke", "white")
            .attr("stroke-linejoin", "round")
            .attr("d", path(topojson.mesh(us, us.objects.states, (a, b) => a !== b)));
        
        svg.on("dblclick", reset);

        function reset() {
            states.transition().style("fill", null);
            g.selectAll(".county").remove();
        
            svg.transition().duration(750).call(
                zoom.transform,
                d3.zoomIdentity, // Reset zoom
            );

            isZoomed = false;
            lastClicked = null;
        }


        function clicked(event, d) {
            const [[x0, y0], [x1, y1]] = path.bounds(d);
            const stateName = d.properties.name;
            const stateId = d.id;
            
            event.stopPropagation();

            // Check if we're zooming in on the same element or if the map is already zoomed in
            if (isZoomed) {
                // Reset to initial zoom
                reset();
            } else {
                // Proceed with zooming in
                states.transition().style("fill", null);
                d3.select(this).transition().style("fill", "grey");
                svg.transition().duration(750).call(
                    zoom.transform,
                    d3.zoomIdentity
                        .translate(width / 2, height / 2)
                        .scale(Math.max(2, 0.4 / Math.max((x1 - x0) / width, (y1 - y0) / height)))
                        .translate(-(x0 + x1) / 2, -(y0 + y1) / 2),
                    d3.pointer(event, svg.node())
                );

                isZoomed = true;
                // lastClicked = this;
                lastClicked = this;

                // Remove previous counties if any
                g.selectAll(".county").remove();

                // Filter county data for the selected state
                const countiesForState = county_data.filter(county => county.state_full === stateName);
          

                console.log('countiesForState')
                console.log(countiesForState)

                var zeros = d3.format('05d')
                // Find the corresponding counties in the topoJSON
                const countyFeatures = topojson.feature(county, county.objects.counties).features
                    .filter(feature => countiesForState.some(county => zeros(county.FIPS).toString() === feature.id && county.state_full === stateName));

                    
                console.log('countyFeatures')
                console.log(countyFeatures)
                

                // Render counties
                const countyPaths = g.selectAll(".county")
                    .data(countyFeatures)
                    .enter().append("path")
                    .attr("class", "county")
                    .attr("d", path)
                    .attr('stroke', 'black')
                    .attr("fill", d => {
                        // Determine the fill based on the county's data
                        const countyData = countiesForState.find(county => county.county === d.properties.name);
                        return countyData ? countyColorScale(countyData.percent_cases) : "#ccc";
                    })
                    .on('click', clicked)
                countyPaths.append("title")
                    .text(d => d.properties.name);
                    
            }
            g.selectAll('.info-box-group').remove()
        }


        function zoomed(event) {
            const {transform} = event;
            g.attr("transform", transform);
            g.attr("stroke-width", 1 / transform.k);
        }

        // Legend 
        const legendHeight = 200; // Total height of the legend
        const legendWidth = 20; // Width of each legend item
        const legendMargin = 10; // Margin around the legend
        const legendNumBlocks = stateColorScale.range().length; // Number of blocks in the legend
        const legendBlockHeight = legendHeight / legendNumBlocks; // Height of each block

        // Define the legend group
        
        const l = svg.append('g')
        const legend = l.append('g')
            .attr('id', 'legend')
            .attr('transform', 'translate(850,555)');

        // Add colored rectangles for each segment of the legend
        const [x_0, x_1, x_2, x_3, x_4] = stateColorScale.range()

        legend.selectAll('rect')
            .data([x_4, x_3, x_2, x_1, x_0])
            .enter().append('rect')
                .attr('x',(d, i) => i * legendBlockHeight)
                .attr('y', 0)
                .attr('width', legendBlockHeight)
                .attr('height', legendWidth)
                .attr('fill', d => d);

        const legend_title = l.append('g')
            .attr('id', 'legend_title')
            .attr('transform', 'translate(840,555)');
        
        // Title text
        legend_title.append('text') 
            .attr('x', 100)
            .attr('y', -10) // Position the title above the legend rectangles
            .text('% Population Positive for COVID-19')
            .attr('text-anchor', 'middle')
            .attr('class', 'legend-title')
            .style('font-weight', 'bold');

            
        const legendScale = d3.scaleLinear()
            .domain(stateColorScale.domain())
            .range([legendHeight, 0]);

        const legendAxis = d3.axisBottom(legendScale)
            .tickSize(0)
            .ticks(legendNumBlocks)
            .tickFormat(d => `${d}%`);

    
        legend.call(legendAxis)
            .selectAll('text')
            .style('text-anchor', 'start')
            .attr('dx', 5)
            .attr('dy', 0)
            .attr('transform', 'translate(-20,30)');
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
        margin-bottom: -150px;
        margin-top: -50px;
    }
    svg {
        margin-top: -100px;
        display: block; /* Center the SVG horizontally */
        width: 100%; /* Make SVG responsive */
        height: auto; /* Maintain aspect ratio */
    }

    .legend-title {
    fill: #000;
    font-size: 14px;
    font-weight: bold;
    }

    .subtitle {
        font-size: 18px;
        width: 75%; /* Set the width to take the full container width */
        /* max-width: 600px; Max width to avoid being too stretched */
        margin: 0 auto; /* This centers the block horizontally */
        text-align: center;
        color: #646161; /* This sets the text color */
        padding: 20px; /* This adds space inside the borders of the element */
        box-sizing: border-box; /* This ensures padding is included in the width */
    }

    .subtitle_bottom {
        font-size: 18px;
        width: 75%; /* Set the width to take the full container width */
        /* max-width: 600px; Max width to avoid being too stretched */
        margin: 0 auto; /* This centers the block horizontally */
        text-align: center;
        color: #646161; /* This sets the text color */
        padding: 20px; /* This adds space inside the borders of the element */
        box-sizing: border-box; /* This ensures padding is included in the width */
    }


</style>

<p class='subtitle_bottom'> Note that some data is missing for some counties due to 
    missing official data from CDC for regions with small populations. Also note that 
    since 2023 population data for each state and county is not yet posted officially, we have used the 
    estimates for 2022 populations from US Census."</p >