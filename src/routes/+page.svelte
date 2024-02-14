<script>
import { onMount } from 'svelte' ;
import * as d3 from 'd3';
let tempData = [];

 onMount(async () => {

     const res = await fetch('county_data.csv'); 

     const csv = await res.text();

     tempData = d3.csvParse(csv, d3.autoType)

     console.log(tempData);

 });
 </script>
