---
layout: post
title: "D3 playground"
---

<script type="text/javascript" src="http://d3js.org/d3.v3.min.js"></script>

above

<div id="foo">
  <p id="tstep">t = 250</p>
  <p id="faster">faster</p>
  <p id="slower">slower</p>
  <p id="start">start</p>
  <p id="stop">stop</p>
</div>

<div id="scales">
  scales:
</div>

below

<script type="text/javascript">
  function TimeSeriesScales(per_timescale) {
    this.scale2series = {1: []};
    this.count = 0;
    this.per_timescale = per_timescale;
    this.scale = 1;
    this.scales = [1];
    this.series = [];
    this.get_series = function(scale) {
      return this.scale2series[scale];
    }
    this.add_point = function(x, y) {
      this.count += 1
      for (var s = 0; s < this.scales.length; s++) {
        sc = this.scales[s];
        if (this.count % sc === 0) {
          this.scale2series[sc].push([x, y]);
          if (this.scale2series[sc].length >= this.per_timescale) {
            this.scale2series[sc].shift();
          }
        }
      }
      if (this.count % this.scale === 0) {
        this.series.push([x,y])
      }
      if (this.count % (this.per_timescale * this.scale) === 0) {
        this.scale *= 10;
        this.scales.push(this.scale);
        this.scale2series[this.scale] = this.series.slice(this.series.length-this.per_timescale, this.series.length)
      }
    }
    this.add_points = function(xys) {
      for (var i = 0; i < xys.length; i++) {
        var xy = xys[i]
        this.add_point(xy[0], xy[1]);
      }
    }
  }
  tss = new TimeSeriesScales(50);
  var i_tss = 0;
  var curr_tss = 0;
  var more_tss = function() {
    i_tss++;
    curr_tss += d3.random.normal()();
    tss.add_point(i_tss, curr_tss);
  }
  for (var i = 0; i < 50; i++){
    more_tss();
  }

  console.log("foo");
  //Width and height
  var w = 500;
  var h = 300;
  var padding = 40;

  //Create scale functions
  var xScale = d3.scale.linear()
             .domain([d3.min(tss.scale2series[1], function(d) { return d[0]; }),
                      d3.max(tss.scale2series[1], function(d) { return d[0]; })])
             .range([padding, w - padding * 2]);

  var yScale = d3.scale.linear()
             .domain([d3.min(tss.scale2series[1], function(d) { return d[1]; }),
                      d3.max(tss.scale2series[1], function(d) { return d[1]; })])
             .range([h - padding, padding]);

  var rScale = d3.scale.linear()
             .domain([0, d3.max(tss.scale2series[1], function(d) { return d[1]; })])
             .range([2, 5]);

  //Define X axis
  var xAxis = d3.svg.axis()
            .scale(xScale)
            .orient("bottom")
            .ticks(5);

  //Define Y axis
  var yAxis = d3.svg.axis()
            .scale(yScale)
            .orient("left")
            .ticks(5);

  //Create SVG element
  var svg = d3.select("#foo")
        .append("svg")
        .attr("width", w)
        .attr("height", h);

  //Create circles
  svg.selectAll("circle")
     .data(tss.scale2series[1])
     .enter()
     .append("circle")
     .attr("cx", function(d) {
        return xScale(d[0]);
     })
     .attr("cy", function(d) {
        return yScale(d[1]);
     })
     .attr("r", function(d) {
        return 3;
     });

  //Create X axis
  svg.append("g")
    .attr("class", "x axis")
    .attr("transform", "translate(0," + (h - padding) + ")")
    .call(xAxis);

  //Create Y axis
  svg.append("g")
    .attr("class", "y axis")
    .attr("transform", "translate(" + padding + ",0)")
    .call(yAxis);

  var t = 250;

  scales = d3.select("#scales");
  scale = 1;
  scales.selectAll("p")
    .data(tss.scales)
    .enter()
    .append("p")
    .text((d) => d)
    .on("click", function (d) {scale = d; another(250)});

  var another = function (dur) {
    more_tss();

    scales.selectAll("p")
      .data(tss.scales)
      .enter()
      .append("p")
      .text((d) => {return d;})
      .on("click", function (d) {scale = d; another(250)});

    // console.log(tss.series[tss.series.length-1]);
    pts = svg.selectAll("circle")
     .data(tss.scale2series[scale], function(d) {return d[0];});
    pts.enter()
     .append("circle")
     .attr("cx", function(d) {
        return xScale(d[0]+scale);
     })
     .attr("cy", function(d) {
        return yScale(d[1]);
     })

    xScale.domain([d3.min(tss.scale2series[scale], function(d) { return d[0]; }),
                   d3.max(tss.scale2series[scale], function(d) { return d[0]; })]);
    yScale.domain([d3.min(tss.scale2series[scale], function(d) { return d[1]; }),
                   d3.max(tss.scale2series[scale], function(d) { return d[1]; })]);
    pts
     .transition()
     .attr("cx", function(d) {
        return xScale(d[0]);
     })
     .attr("cy", function(d) {
        return yScale(d[1]);
     })
     .attr("r", function(d) {
        return 3;
     })
     .duration(dur)
     .ease("linear");
    svg.select(".x.axis")
      .transition()
      .duration(dur)
      .ease("linear")
      .call(xAxis);
    svg.select(".y.axis")
      .transition()
      .duration(dur)
      .ease("linear")
      .call(yAxis);
    pts.exit()
      .remove();
  };

  var keep_going = false;
  var going = false;
  var always_another = function() {
    setTimeout(function () {
        if (keep_going) {
          another(t*scale);
          always_another();
        }
      },
      t);
  }
  always_another();
  d3.select("#faster").on("click",() => {t *= 0.5; d3.select("#tstep").text("t = " + t/1000.0);})
  d3.select("#slower").on("click", () => {t *= 2; d3.select("#tstep").text("t = " + t/1000.0);});
  d3.select("#start").on("click", () => {keep_going = true; if (!going) {going = true; always_another();}});
  d3.select("#stop").on("click", () => {keep_going = false; going = false;});

</script>