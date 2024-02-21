<h1 style ='text-align: center; font-size: 46px'> Less Populated States and Counties Faced Greater COVID-19 Pandemic Challenges </h1>
<h2 style ='text-align: center'> A Dive into Statewide and Local Data Insights </h2>
<a style ='text-align: center' href="dsc106writeup.html">my link</a>

<p class='subtitle'> With the rise in vaccination rates leading the charge, the Biden Administration ended the national emergency and public health emergency declarations 
    on May 11, 2023, related to the COVID-19 pandemic. Below, we have come up with
    a visualization to help compare how severe the pandemic was for each state (when tha map is not zoomed in)
    and for each county (when the map is zoomed in) based on how much of the population 
    has previously tested positive for COVID-19 as of the last day of records - May 11, 2023. 
</p >
<p class='subtitle'> Indicated by the intensity of blue, you may notice that <bold class='bolded'> states with smaller populations,
    such as Alaska, North Dakota, Rhode Island, along with the contiguous states of Kentucky, Tennessee, 
    and West Virginia, have experienced a higher prevalence of COVID-19. </bold>
    Each of these states has reported that more than 35% of their populations have cumulatively 
    tested positive for COVID-19 as of May 11, 2023. </p >

<br>
<p class = 'normal_left_text'>💡💡 Advice: try clicking on a state, and click again: </p >


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

        // Color scale for state and state
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
            const [x, y] = path.centroid(d); // centroid of the state path.
            const stateName = d.properties.name; // state name from the TopoJSON properties.
            const cases = casesByState[stateName] || 0; 

            let populationByState = {}; 
            state_tot_cases.forEach(d => {
                populationByState[d.state_full] = d.population_data_millions;
            });

            const population = populationByState[d.state_full];

            d3.select(this)
                .attr("fill", "#fffea8") 

            const infoBoxGroup = g.append("g")
                .attr("class", "info-box-group")
                .attr("transform", `translate(${x}, ${y})`)
                .style("pointer-events", "none"); // ignore mouse events (moving on infobox)

            // Append a rectangle to act as the background box.
            infoBoxGroup.append("rect")
                .attr("x", 15) 
                .attr("y", -35) 
                .attr("width", 180) 
                .attr("height", 55) 
                .attr("fill", "white") 
                .attr("stroke", "black") 
                .attr("class", "state-info-box"); 

            // Append text to show the state name inside the box.
            infoBoxGroup.append("text")
                .attr("x", 105) 
                .attr("y", -15) 
                .attr("text-anchor", "middle") 
                .text(stateName)
                .attr("class", "state-name-text") 
                .style('font-weight', 'bold');

            // Append another text to show the percentage of population being tested inside the box.
            infoBoxGroup.append("text")
                .attr("x", 110) 
                .attr("y", 0) 
                .attr("text-anchor", "middle") 
                .text(`${(cases*100).toFixed(2)}%`) 
                .attr('text-highlight', 'yellow')
                .attr("class", "state-cases-text"); 
            
            // infoBoxGroup.append("rect")
            //     .attr("x", 107)
            //     .attr("y", 13)
            //     .attr("height", 30)
            //     .attr("rx", 15)
            //     .attr("ry", 15)
            //     .style("fill", "#black");

            infoBoxGroup.append("text")
                // .attr('margintop', 10)
                .attr("x", 107) 
                .attr("y", 13) // Position the text in the box (based on the box coordinates)
                .attr("text-anchor", "middle") 
                .text(`Population: ${(populationByState[d.properties.name]).toFixed(3)} M`) 
                .attr("class", "state-cases-text"); 
        }

        function handleMouseOut(event, d) {
            // Remove the box of text when the mouse leaves a state
            g.selectAll(".info-box-group").remove();

            // Reset the fill color to its original state
            const originalFillColor = determineOriginalFillColor(d); 
            d3.select(this)
                .attr("fill", originalFillColor) 
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

            if (isZoomed) { // Check if the map is already zoomed in
                reset();
            } else { // if not zoomed in already, zoom in
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
        const legendHeight = 200; 
        const legendWidth = 20;
        const legendMargin = 10; 
        const legendNumBlocks = stateColorScale.range().length; 
        const legendBlockHeight = legendHeight / legendNumBlocks;

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
            .attr('y', -10) 
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
        height: auto; 
    }

    .legend-title {
    fill: #000;
    font-size: 14px;
    font-weight: bold;
    }

    .subtitle {
        font-size: 18px;
        width: 76%; 
        /* max-width: 600px;  */
        margin: 0 auto; /* centers the block horizontally */
        text-align: center;
        color: #646161; 
        padding: 20px; /* adds space inside the borders of the element */
        box-sizing: border-box; 
    }

    .subtitle_more_padding {
        font-size: 18px;
        width: 76%; 
        /* max-width: 600px;  */
        margin: 0 auto; /* centers the block horizontally */
        text-align: center;
        color: #646161; 
        padding: 30px; /* adds space inside the borders of the element */
        box-sizing: border-box; 
    }

    .subtitle_bottom {
        font-size: 18px;
        width: 75%; 
        /* max-width: 600px; */
        margin: 0 auto; 
        text-align: center;
        color: #646161; 
        padding: 20px; 
        box-sizing: border-box; 
    }

    .normal_center_text {
        font-size: 18px;
        width: 75%; 
        /* max-width: 600px; */
        margin: 0 auto; 
        text-align: center;
        color: #646161; 
        padding: 0 0 20px 20px; 
        box-sizing: border-box; 
    }

    .normal_left_text {
        font-size: 18px;
        width: 75%; 
        /* max-width: 600px; */
        margin: 0 auto; 
        text-align: left;
        color: #646161; 
        padding: 0 0 0px 20px; 
        box-sizing: border-box; 
    }


    .bolded {
        font-weight: bold;
        color: #235097;
    }
    
</style>


<p class='subtitle_bottom'> ** Please note that our county-level data may be incomplete in some instances,
     as official CDC records for areas with small populations are not always available.
</p >

<p class='normal_center_text'> ** 
    Additionally, since the most recent 2023 population data for states and counties have not 
    been officially released, we have utilized the 2022 population estimates from the U.S. 
    Census to conduct our analysis."
</p >