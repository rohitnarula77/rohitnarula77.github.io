/*
*    main.js
*/

// global variables
let allCalls
let calls
let stackedArea
let timeline

const parseTime = d3.timeParse("%d/%m/%Y")
const formatTime = d3.timeFormat("%d/%m/%Y")

d3.json("data/calls.json").then(data => {    
	data.forEach(d => {
		d.call_revenue = Number(d.call_revenue)
		d.units_sold = Number(d.units_sold)
		d.call_duration = Number(d.call_duration)
		d.date = parseTime(d.date)
	})

	allCalls = data
	calls = data
  stackedArea = new StackedAreaChart("#stacked-area")
  timeline = new Timeline("#timeline")
})

d3.selectAll("#var-select").on("change", function (data) {
  	stackedArea.wrangleData()
	timeline.wrangleData()
});

function brushed() {
	const selection = d3.event.selection || timeline.x.range()
	const newValues = selection.map(timeline.x.invert)
	changeDates(newValues)
}

function changeDates(values) {
	calls = allCalls.filter(d => ((d.date > values[0]) && (d.date < values[1])))
	d3.selectAll("#dateLabel1").text(formatTime(values[0]))
	d3.selectAll("#dateLabel2").text(formatTime(values[1]))
	stackedArea.wrangleData()
}