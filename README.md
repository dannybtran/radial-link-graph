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
var i = 0;
while (match = rx.exec(document.body.innerHTML)) {
  if (i >= 10000) { break; }
  var url = new URL(match[1]);
  var segs = url.hostname.split('.').reverse();
  if (['com','co'].indexOf(segs[1]) > -1) { 
    v = segs.shift();   
    segs[0] = segs[0] + '.' + v; 
  }
  segs = segs.map(function(s) { return "." + s + "."; });
  var _paths = url.pathname.split('/');
  paths = [];
  for(var k in _paths) {
    if (_paths[k]) {
      paths.push("/" + _paths[k] + "/");
    }
  }
  segs = segs.concat(paths);  
  var obj = {};
  function recurse(obj, segs) {
    if (segs.length == 0) {
      return obj;
    }
    var k = segs.shift();
    obj[k] = {};
    obj[k] = recurse(obj[k], segs);
    return obj;
  }
  obj = recurse(obj, segs);
  tree = deepmerge(tree, obj);
  i++;
}
function getChildren(_tree) {
    var children = [];
    var d3tree = {};
    for(var k in _tree) {
      if (_tree.hasOwnProperty(k)) {
        d3tree = {};
        d3tree["name"] = k;
        d3tree["children"] = getChildren(_tree[k]);
        d3tree["size"] = d3tree["children"].length;
        children.push(d3tree);
      }
    }
    return children;
}
var dtree = {name: ""};
var children = [];
for(var k in tree) {
    var child = {name: k};
    child['children'] = getChildren(tree[k]);
    children.push(child);
}
dtree["children"] = children;
document.body.innerHTML = "<style>html { background-color: white !important; background-image: none !important; }.hidden { opacity: 0.3; } body { background-color: white !important; background-image: none !important; }.node circle {fill: #fff;  stroke: steelblue;  stroke-width: 1.5px;}.node {  font: 12px sans-serif;} .node.highlight circle {fill: #00f} .link {  fill: none;  stroke: #ccc;  stroke-width: 1.5px;} .link.highlight { stroke: #bbb; }</style>";
document.body.innerHTML += "<div style='position: fixed; top: 10px; left: 10px;'>Zoom <button onmousedown='zin()' onmouseup='zinstop()'>+</button><button onmousedown='zout()' onmouseup='zoutstop()'>-</button></div>";
function drawGraph(diameter) {
  d3.select('svg').remove();
var tree = d3.layout.tree()
    .size([360, diameter/2])
    .separation(function(a, b) { return (a.parent == b.parent ? 1 : 2) / a.depth; });
var diagonal = d3.svg.diagonal.radial()
    .projection(function(d) { return [d.y, d.x / 180 * Math.PI]; });
var svg = d3.select("body").append("svg")
    .attr("width", diameter * 2)
    .attr("height", diameter * 2)
    .attr("style", "margin: auto; display: block;")
  .append("g")
    .attr("transform", "translate(" + diameter + "," + diameter + ")");
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
    .on('mouseover', function(e) { 
       d3.select(this.parentNode).attr('class','node highlight');
       window._ancestors = [];
       window._predecessors = [];
       var datum = d3.select(this).datum();
       while (datum.parent) {
         window._ancestors.push(datum);
         datum = datum.parent;
       }
       var datum = d3.select(this).datum();
       function getChillen(cs) {
         if (cs.children) {
           for(var i = 0; i < cs.children.length; i++) {
             window._predecessors.push(cs.children[i]);           
             getChillen(cs.children[i]);
           }
         }
       }
       getChillen(datum);
       var links = svg.selectAll('.link');
       links.attr('class','link hidden');
       links.filter(function(d, i) {
         if (window._ancestors.indexOf(d.target) > -1 || window._predecessors.indexOf(d.target) > -1) {
           return true;
         }
         return false;
       }).attr('class', 'link highlight');
       var nodes = svg.selectAll('.node');
       nodes.attr('class','node hidden');
       nodes.filter(function(d, i) {
         if (window._ancestors.indexOf(d) > -1 || window._predecessors.indexOf(d) > -1) {
           return true;
         }
         return false;
       }).attr('class', 'node highlight');       
     })
     .on('mouseout', function(e) {
       svg.selectAll('.node').attr('class','node');
       svg.selectAll('.link').attr('class','link');
     })
    .attr("fill", function(d) { return d.highlighted ? "#00f" : "#fff"; })
    .attr("r", 5);
node.append("text")
    .attr("dy", ".31em")
    .attr("text-anchor", function(d) { return d.x < 180 ? "start" : "end"; })
    .attr("transform", function(d) { return d.x < 180 ? "translate(8)" : "rotate(180)translate(-8)"; })
    .text(function(d) { return d.name; });
d3.select(self.frameElement).style("height", diameter - 150 + "px");
centerScroll();
}
var _radius = window.innerHeight/2;
function centerScroll() { window.scrollTo(document.body.scrollWidth/2 - window.innerWidth/2, document.body.scrollHeight/2 - window.innerHeight/2); }
var zoomingIn = false;
var zoomingOut = false;
function zin() { zoomingIn = true; }
function zinstop() { zoomingIn = false; }
function zout() { zoomingOut = true; }
function zoutstop() { zoomingOut = false; }
document.body.addEventListener('mouseup', function() {
  zoomingIn = false; zoomingOut = false;
});
setInterval(function() {
  if (zoomingIn) { _radius += 20; drawGraph(_radius*2); }
  else if (zoomingOut) { _radius -= 20; drawGraph(_radius*2); }  
}, 10);

var s = document.createElement('script');
s.type = 'text/javascript';
s.onload = function() { 
  drawGraph(window.innerHeight); 
};
s.src = 'https://cdnjs.cloudflare.com/ajax/libs/d3/3.5.14/d3.min.js';
document.body.appendChild(s);
```
