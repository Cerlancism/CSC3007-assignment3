<template>
    <div>
        <div style="position: fixed" v-if="this.hovering">
            <v-card max-width="240" outlined>
                <v-list-item three-line>
                    <v-list-item-content>
                        <v-list-item-title>
                            {{ this.hoverDistrict | capitalise }}
                        </v-list-item-title>
                        <v-list-item-subtitle>
                            {{ this.hoverZone | capitalise }}
                        </v-list-item-subtitle>
                        <v-list-item-subtitle>
                            Popluation: {{ this.hoverPopulation }}
                        </v-list-item-subtitle>
                    </v-list-item-content>
                </v-list-item>
            </v-card>
        </div>
        <svg id="sgmap" class="ocean"></svg>
    </div>
</template>


<script lang="ts">
    import { Component, Prop, Vue, } from 'vue-property-decorator';
    import * as d3 from "d3";

    import { Population } from '@/model/Popluation'

    function orderMagnitude(n: number)
    {
        var order = Math.floor(Math.log(n) / Math.LN10
            + 0.000000001);
        return Math.pow(10, order);
    }

    @Component(
        {
            filters: {
                capitalise(text: string)
                {
                    return text.replace(/(\w)(\w*)/g,
                        function (g0, g1, g2) { return g1.toUpperCase() + g2.toLowerCase(); })
                }
            }
        }
    )
    export default class Choropleth extends Vue
    {
        hovering = false
        hoverDistrict = "District Title"
        hoverZone = "Zone Title"
        hoverPopulation = NaN

        async mounted()
        {
            let width = 1000, height = 600;

            let svg = d3.select("svg")
                .attr("viewBox", "0 0 " + width + " " + height)

            // Load external data
            const [sgmap, population2021Raw] = await Promise.all([d3.json("data/sgmap.json"), d3.csv("data/population2021.csv")]) as [any, any[]]

            const population2021 = population2021Raw.map(x =>
            {
                const pop = Number(x['Population'])
                return {
                    subzone: x['Subzone'],
                    planningArea: x['Planning Area'],
                    population: isNaN(pop) ? 0 : pop
                } as Population
            })
                .sort((a, b) => a.population - b.population)


            // Compuation
            const [populationMin, populationMax] = [population2021.find(x => x.population > 0), population2021[population2021.length - 1]]
            const populationCap = 150000

            console.log("Population MinMax", populationMin, populationMax)

            const domainSteps = 9
            const domain = [...Array(domainSteps - 1).keys()].map(x => populationCap / domainSteps + x * (populationCap / domainSteps));

            domain.unshift(0)

            console.log("domain", domain)

            

            const colorScaler = d3.scaleLinear()
                .domain(domain)
                .range(d3.schemeGreys[domain.length] as any);

            // Map and projection
            var projection = d3.geoMercator()
                .center([103.851959, 1.290270])
                .fitExtent([[20, 20], [980, 580]], sgmap);

            let geopath = d3.geoPath().projection(projection);

            svg.append("g")
                .attr("id", "districts")
                .selectAll("path")
                .data(sgmap.features)
                .enter()
                .append("path")
                .attr("d", geopath as any)
                .attr("fill", (data: any) =>
                {
                    const pathProperties = data.properties as { [key in string]: string }
                    const areaName = pathProperties["Planning Area Name"]
                    const subzoneName = pathProperties["Subzone Name"]
                    const populationAreas = population2021.filter(x => x.planningArea.toLowerCase() === areaName.toLowerCase())

                    // console.log(areaName, subzoneName, populationAreas)

                    const populationSubzone = populationAreas.find(x => x.subzone.toLowerCase() === subzoneName.toLowerCase())

                    let population = 0

                    if (populationSubzone?.population)
                    {
                        population = populationSubzone.population
                    }
                    else if (populationAreas[0]?.population)
                    {
                        population = populationAreas[0].population
                    }

                    data.properties["population"] = population

                    return colorScaler(population)
                })
                .on("mouseover", (event, data: any) =>
                {
                    this.hovering = true
                    this.hoverDistrict = data.properties["Planning Area Name"]
                    this.hoverZone = data.properties["Name"]
                    this.hoverPopulation = data.properties["population"]
                    d3.select(event.target).classed("map-hover", true);
                })
                .on("mouseout", (event, data) =>
                {
                    this.hovering = false
                    d3.select(event.target).classed("map-hover", false);
                })

            // Legend
            const gradientDefs = svg.append("defs");
            const linearGradient = gradientDefs.append("linearGradient")
                .attr("id", "legend-gradient")
                .attr("x1", 0)
                .attr("y1", 0.5)
                .attr("x2", 1)
                .attr("y2", 0.5);

            for (let i = 0; i <= 10; i++)
            {
                linearGradient.append("stop")
                    .attr("offset", `${i * 10}%`)
                    .attr("stop-color", d3.interpolateGreys(/*1 - */i * 0.1));
            }

            const legendTransform = {
                x: width / 3 + 100,
                y: height - 70,
                height: 10,
                width: width / 2,
            };

            let legendScaler = d3
                .scaleLinear()
                .domain([0, populationCap])
                .range([0, legendTransform.width]);

            svg.append("rect")
                .attr("x", legendTransform.x)
                .attr("y", legendTransform.y)
                .attr("width", legendTransform.width)
                .attr("height", legendTransform.height)
                .attr("fill", "url(#legend-gradient)");

            svg.append("g")
                .attr("class", "x-axis")
                .attr("transform", `translate(${legendTransform.x}, ${legendTransform.y + legendTransform.height})`)
                .call(d3.axisBottom(legendScaler).ticks(domain.length))
                .selectAll("text")
                .style("text-anchor", "center")
                .style("font-size", "12px");
        }
    }
</script>

<style>
    .ocean {
        background-color: #bbdefb;
    }

    .map-hover {
        stroke: #2196f3;
        stroke-width: 2px;
    }
</style>