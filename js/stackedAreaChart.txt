/*
*    stackedAreaChart.js
*/

class StackedAreaChart {
  constructor(_parentElement) {
    this.parentElement = _parentElement

    this.initVis()
  }

  initVis() {
    const vis = this

    vis.MARGIN = { LEFT: 80, RIGHT: 150, TOP: 50, BOTTOM: 40 }
    vis.WIDTH = 960 - vis.MARGIN.LEFT - vis.MARGIN.RIGHT
    vis.HEIGHT = 450 - vis.MARGIN.TOP - vis.MARGIN.BOTTOM

    vis.svg = d3.select(vis.parentElement).append("svg")
      .attr("width", vis.WIDTH + vis.MARGIN.LEFT + vis.MARGIN.RIGHT)
      .attr("height", vis.HEIGHT + vis.MARGIN.TOP + vis.MARGIN.BOTTOM)

    vis.g = vis.svg.append("g")
      .attr("transform", "translate(" + vis.MARGIN.LEFT + "," + vis.MARGIN.TOP + ")")

    vis.tooltip = d3.select('#tooltip');
    vis.tooltip
        .style("position", "absolute")
        .style("width", "200px")
        .style("height", "100px")
        .style("pointer-events", "none")
        .style("opacity", 0);

    vis.color = d3.scaleOrdinal(d3.schemePastel1)

    vis.x = d3.scaleTime().range([0, vis.WIDTH])
    vis.y = d3.scaleLinear().range([vis.HEIGHT, 0])

    vis.yAxisCall = d3.axisLeft()
    vis.xAxisCall = d3.axisBottom()
      .ticks(4)
    vis.xAxis = vis.g.append("g")
      .attr("class", "x axis")
      .attr("transform", "translate(0," + vis.HEIGHT + ")")
    vis.yAxis = vis.g.append("g")
      .attr("class", "y axis")

    vis.stack = d3.stack()
      .keys(["west", "south", "northeast", "midwest"])

    vis.area = d3.area()
      .x(d => vis.x(parseTime(d.data.date)))
      .y0(d => vis.y(d[0]))
      .y1(d => vis.y(d[1]))

    vis.addLegend()

    vis.wrangleData()
  }


  wrangleData() {
    const vis = this

    vis.variable = d3.select("#var-select").property("value")
    vis.dayNest = d3.nest()
      .key(d => formatTime(d.date))
      .entries(calls)

    vis.dataFiltered = vis.dayNest
      .map(day => day.values.reduce(
        (accumulator, current) => {
            accumulator.date = day.key
            accumulator[current.team] = accumulator[current.team] + current[vis.variable]
            return accumulator
        }, {
          "northeast": 0,
          "midwest": 0,
          "south": 0,
          "west": 0
        }
      ))

     vis.sorted = vis.dataFiltered.sort(function(x, y){
		const [xday, xmonth, xyear] = x.date.split('/');
		const xdate = new Date(+xyear, +xmonth - 1, +xday);
		const [yday, ymonth, yyear] = y.date.split('/');
		const ydate = new Date(+yyear, +ymonth - 1, +yday);
        return d3.ascending(xdate, ydate);
      })

	  var convertDateToString = function(d) {
	    const [xday, xmonth, xyear] = d.split('/');
	    return new Date(+xyear, +xmonth - 1, +xday);
	  }

      vis.mapped = vis.sorted.map(function(d) { 
     	return {
     		date:convertDateToString(d.date),
     		midwest: d.midwest,
     		northeast: d.northeast,
     		south: d.south,
     		west: d.west
     	}
      });

    vis.updateVis()
  }


  updateVis() {
    const vis = this
    vis.t = d3.transition().duration(750)
    const annotationMaxObject = { date: '', maxValue: Number.MIN_VALUE, maxRegion: '', maxRegionValue: Number.MIN_VALUE};
    const annotationMinObject = { date: '', minValue: Number.MAX_VALUE, minRegion: '', minRegionValue: Number.MAX_VALUE};
    if (vis.dataFiltered.length <= 1) {
    	return
    }
    vis.maxDateVal = d3.max(vis.dataFiltered, d => {
      var vals = d3.keys(d).map(key => key !== 'date' ? d[key] : 0)
      var sum = d3.sum(vals);
      if (sum > annotationMaxObject.maxValue) {
      	annotationMaxObject.maxValue = sum
      	annotationMaxObject.date = d.date
      	annotationMaxObject.maxRegionValue = d3.max(vals.filter(function(value, index, array){
      		return value != 0
      	}))
      	for (let i = 0; i < vals.length; i++) {
  			if (d.northeast == annotationMaxObject.maxRegionValue) {
  				annotationMaxObject.maxRegion = 'northeast';
  			}
  			else if (d.midwest == annotationMaxObject.maxRegionValue) {
  				annotationMaxObject.maxRegion = 'midwest';
  			}
  			else if (d.south == annotationMaxObject.maxRegionValue) {
  				annotationMaxObject.maxRegion = 'south';
  			}
  			else if (d.west == annotationMaxObject.maxRegionValue) {
  				annotationMaxObject.maxRegion = 'west';
  			} 
		}
      }
      return sum
    })

    vis.minDateVal = d3.min(vis.dataFiltered, d => {
      var vals = d3.keys(d).map(key => key !== 'date' ? d[key] : 0)
      var sum = d3.sum(vals)
      if (sum < annotationMinObject.minValue) {
      	annotationMinObject.minValue = sum
      	annotationMinObject.date = d.date
      	annotationMinObject.minRegionValue = d3.min(vals.filter(function(value, index, array){
      		return value != 0
      	}))
      	for (let i = 0; i < vals.length; i++) {
  			if (d.northeast == annotationMinObject.minRegionValue) {
  				annotationMinObject.minRegion = 'northeast';
  			}
  			else if (d.midwest == annotationMinObject.minRegionValue) {
  				annotationMinObject.minRegion = 'midwest';
  			}
  			else if (d.south == annotationMinObject.minRegionValue) {
  				annotationMinObject.minRegion = 'south';
  			}
  			else if (d.west == annotationMinObject.minRegionValue) {
  				annotationMinObject.minRegion = 'west';
  			} 
		}
      }
      return sum
    })

    // update scales
    vis.x.domain(d3.extent(vis.dataFiltered, (d) => {
      return parseTime(d.date)
    }))
    vis.y.domain([0, vis.maxDateVal])

    // update axes
    vis.xAxisCall.scale(vis.x)
    vis.xAxis.transition(vis.t).call(vis.xAxisCall)
    vis.yAxisCall.scale(vis.y)
    vis.yAxis.transition(vis.t).call(vis.yAxisCall)

    const type = d3.annotationLabel
    var capitalizeFirstLetter = function(string) {
      return string.charAt(0).toUpperCase() + string.slice(1);
    }

    const annotations = [
      {
        note: {
          label: 'Date: ' + annotationMaxObject.date + ', Max region: ' + capitalizeFirstLetter(annotationMaxObject.maxRegion) + ', value: ' + Math.round(annotationMaxObject.maxRegionValue),
          title: 'Max cummulative ' + vis.variable + ': ' + Math.round(annotationMaxObject.maxValue), 
          wrap: 120,
          align: "left"
        },
        connector: {
            end: "arrow" // 'dot' also available
        },
        x: vis.x(parseTime(annotationMaxObject.date)),
        y: vis.y(annotationMaxObject.maxValue),
        dy: 30,
        dx: 20,
        color: 'black'
      },
      {
        note: {
          label: 'Date: ' + annotationMinObject.date + ', Min region: ' + capitalizeFirstLetter(annotationMinObject.minRegion) + ', value: ' + Math.round(annotationMinObject.minRegionValue),
          title: 'Min cummulative ' + vis.variable + ': ' + Math.round(annotationMinObject.minValue),
          wrap: 120,
          align: "left" 
        },
        connector: {
            end: "arrow" // 'dot' also available
        },
        x: vis.x(parseTime(annotationMinObject.date)),
        y: vis.y(annotationMinObject.minValue),
        dy: 30,
        dx: 20,
        color: 'black'
      }
    ]

  const makeAnnotations = d3.annotation()
      .annotations(annotations);

  vis.g.selectAll(".annotation-group").remove()

  vis.g
	.append("g")
	.attr("class", "annotation-group")
	.call(makeAnnotations)


    vis.teams = vis.g.selectAll(".team")
      .data(vis.stack(vis.dataFiltered))

    vis.stackData = vis.stack(vis.dataFiltered);
    
    // update the path for each team
    vis.teams.select(".area")
      .attr("d", vis.area)

     var bisectDate = d3.bisector(function(d) {
     	return d.date;
     }).left;

    vis.teams.enter().append("g")
      .attr("class", d => "team " + d.key)
      .append("path")
        .attr("class", "area")
        .attr("d", vis.area)
        .style("fill", d => vis.color(d.key))
        .style("fill-opacity", 0.5)
        .on("mouseover", function(d) {
        	var x0 = vis.x.invert(d3.mouse(this)[0]),
        	i = bisectDate(vis.mapped, x0);
        	var date = vis.stackData[0][i].data.date;
        	var west = vis.stackData[0][i].data.west;
        	var south = vis.stackData[1][i].data.south;
        	var northeast = vis.stackData[2][i].data.northeast;
        	var midwest =  vis.stackData[3][i].data.midwest;
        	const states = [
        	{ label: "Midwest", color: vis.color("midwest" ), value: midwest},
        	{ label: "Northeast", color: vis.color("northeast"), value: northeast },
        	{ label: "South", color: vis.color("south"), value: south},
        	{ label: "West", color: vis.color("west"), value: west },
        	]

        	vis.tooltip.html(date + " (dd/MM/yyyy) ")
        	.style("opacity", 1)
        	.style('display', 'block')
        	.style("left", (d3.event.pageX-25) + "px")
        	.style("top", (d3.event.pageY-75) + "px")
        	.selectAll()
        	.data(states).enter()
        	.append('div')
        	// .style('color', d => d.color)
        	.html(d => d.label + ': ' + Math.round(d.value));
        })
        .on("mouseout", function(d) {
        	vis.tooltip.style("opacity", 0)
        })
        .on("mousemove", function(d) {
        	var x0 = vis.x.invert(d3.mouse(this)[0]),
        	i = bisectDate(vis.mapped, x0);
        	var date = vis.stackData[0][i].data.date;
	        var west = vis.stackData[0][i].data.west;
	        var south = vis.stackData[1][i].data.south;
	        var northeast = vis.stackData[2][i].data.northeast;
	        var midwest = vis.stackData[3][i].data.midwest;

        	const states = [
        	{ label: "Midwest", color: vis.color("midwest" ), value: midwest},
        	{ label: "Northeast", color: vis.color("northeast"), value: northeast },
        	{ label: "South", color: vis.color("south"), value: south},
        	{ label: "West", color: vis.color("west"), value: west },
        	]

        	vis.tooltip.html(date + " (dd/MM/yyyy) ")
        	.style("opacity", 1)
        	.style('display', 'block')
        	.style("left", (d3.event.pageX-25) + "px")
        	.style("top", (d3.event.pageY-75) + "px")
        	.selectAll()
        	.data(states).enter()
        	.append('div')
        	// .style('color', d => d.color)
        	.html(d => d.label + ': ' + Math.round(d.value));
        });

  }

  addLegend() {
    const vis = this

    const legend = vis.g.append("g")
      .attr("transform", "translate(50, -25)")

    const legendArray = [
      { label: "Midwest", color: vis.color("midwest" )},
      { label: "Northeast", color: vis.color("northeast") },
      { label: "South", color: vis.color("south") },
      { label: "West", color: vis.color("west") }
      
    ]

    const legendCol = legend.selectAll(".legendCol")
      .data(legendArray)
      .enter().append("g")
        .attr("class", "legendCol")
        .attr("transform", (d, i) => "translate(" + (i * 150) + ",0)")
        
    legendCol.append("rect")
      .attr("class", "legendRect")
      .attr("width", 10)
      .attr("height", 10)
      .attr("fill", d => d.color)
      .attr("fill-opacity", 0.5)

    legendCol.append("text")
      .attr("class", "legendText")
      .attr("x", 20)
      .attr("y", 10)
      .attr("text-anchor", "start")
      .text(d => d.label)
  }
}