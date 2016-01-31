# radial-link-graph
A bookmarklet that lets you see all links on a current page in a Radial Reingold–Tilford Tree

```
function deepmerge(target, src) {
  var array = Array.isArray(src);
  var dst = array && [] || {};
  if (array) {
    target = target || [];
    dst = dst.concat(target);
    src.forEach(function(e, i) {
      if (typeof dst[i] === 'undefined') {
        dst[i] = e;
      } else if (typeof e === 'object') {
        dst[i] = deepmerge(target[i], e);
      } else {
        if (target.indexOf(e) === -1) {
          dst.push(e);
        }
      }
    });
  } else {
    if (target && typeof target === 'object') {
      Object.keys(target).forEach(function(key) {
        dst[key] = target[key];
      })
    }
    Object.keys(src).forEach(function(key) {
      if (typeof src[key] !== 'object' || !src[key]) {
        dst[key] = src[key];
      } else {
        if (!target[key]) {
          dst[key] = src[key];
        } else {
          dst[key] = deepmerge(target[key], src[key]);
        }
      }
    });
  }
  return dst;
}
var rx = RegExp("href=\"(http[^\"]+)\"", "g");
var tree = {};
while (match = rx.exec(document.body.innerHTML)) {
  var url = new URL(match[1]);
  var segs = url.hostname.split('.').reverse();
  if (["com","co"].indexOf(segs[segs.length-2]) > -1) { 
    v = segs.pop(); 
    segs[segs.length-1] = segs[segs.length-1] + '.' + v; 
  }
  var obj = {};
  function recurse(obj, segs) {
    if (segs.length == 0) {
      return {
        paths: [url.pathname]
      };
    }
    var k = segs.shift();
    obj[k] = {};
    obj[k] = recurse(obj[k], segs);
    return obj;
  }
  obj = recurse(obj, segs);
  tree = deepmerge(tree, obj);
}
function getChildren(_tree) {
    var children = [];
    var d3tree = {};
    for(var k in _tree) {
      if (_tree.hasOwnProperty(k)) {
        d3tree = {};
        if (k == "paths") {
          for(var kk in _tree[k]) {
            children.push({name: _tree[k][kk], size: 1});
          }
        } else {
          d3tree["name"] = k;
          d3tree["children"] = getChildren(_tree[k]);
          d3tree["size"] = d3tree["children"].length * 100;
          children.push(d3tree);
        }
      }
    }
    return children;
}
var dtree = {name: "root"};
var children = [];
for(var k in tree) {
    var child = {name: k};
    child['children'] = getChildren(tree[k]);
    children.push(child);
}
dtree["children"] = children;

function drawGraph() {
var diameter = 1600;
var tree = d3.layout.tree()
    .size([360, diameter / 2 - 120])
    .separation(function(a, b) { return (a.parent == b.parent ? 1 : 2) / a.depth; });
var diagonal = d3.svg.diagonal.radial()
    .projection(function(d) { return [d.y, d.x / 180 * Math.PI]; });
var svg = d3.select("body").append("svg")
    .attr("width", diameter * 1.5)
    .attr("height", (diameter - 150) * 1.5)
  .append("g")
    .attr("transform", "translate(" + diameter / 2 + "," + diameter / 2 + ")");
var nodes = tree.nodes(dtree),
    links = tree.links(nodes);
var link = svg.selectAll(".link")
    .data(links)
  .enter().append("path")
    .attr("class", "link")
    .attr("d", diagonal);
var node = svg.selectAll(".node")
    .data(nodes)
  .enter().append("g")
    .attr("class", "node")
    .attr("transform", function(d) { return "rotate(" + (d.x - 90) + ")translate(" + d.y + ")"; });
node.append("circle")
    .attr("r", 4.5);
node.append("text")
    .attr("dy", ".31em")
    .attr("text-anchor", function(d) { return d.x < 180 ? "start" : "end"; })
    .attr("transform", function(d) { return d.x < 180 ? "translate(8)" : "rotate(180)translate(-8)"; })
    .text(function(d) { return d.name; });
d3.select(self.frameElement).style("height", diameter - 150 + "px");
}
document.body.innerHTML = "<style>.node circle {fill: #fff;  stroke: steelblue;  stroke-width: 1.5px;}.node {  font: 10px sans-serif;}.link {  fill: none;  stroke: #ccc;  stroke-width: 1.5px;}</style>";
var s = document.createElement('script');
s.type = 'text/javascript';
s.onload = function() { drawGraph(); };
s.src = 'https://cdnjs.cloudflare.com/ajax/libs/d3/3.5.14/d3.min.js';
document.body.appendChild(s);
```
