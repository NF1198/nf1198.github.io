---
layout: post
title:  "Analyzing a point along a line"
date:   2016-09-14 13:31:00 -0600
categories: javascript geometry
customjs:
 - https://ajax.googleapis.com/ajax/libs/jquery/3.1.0/jquery.min.js
---

How to compute the relative position of point w, between points a and b.

<svg id="svg2" xmlns:rdf="http://www.w3.org/1999/02/22-rdf-syntax-ns#" xmlns="http://www.w3.org/2000/svg" viewBox="0 0 357.00001 191" width="357" version="1.1" xmlns:cc="http://creativecommons.org/ns#" height="191" xmlns:dc="http://purl.org/dc/elements/1.1/" style="margin-left:auto; margin-right:auto; margin-top: 1.5em; margin-bottom: 1.5em; display: block;">
 <metadata id="metadata7">
  <rdf:RDF>
   <cc:Work rdf:about="">
    <dc:format>image/svg+xml</dc:format>
    <dc:type rdf:resource="http://purl.org/dc/dcmitype/StillImage"/>
    <dc:title/>
   </cc:Work>
  </rdf:RDF>
 </metadata>
 <g id="layer1" transform="translate(564,276)">
  <text id="text9361" style="word-spacing:0px;letter-spacing:0px" xml:space="preserve" font-size="22.364px" line-height="125%" y="-125.67734" x="-253.88962" font-family="MathJax_Main" fill="#000000"><tspan id="tspan9363" y="-125.67734" x="-253.88962">a</tspan></text>
  <text id="text9365" style="word-spacing:0px;letter-spacing:0px" xml:space="preserve" font-size="22.364px" line-height="125%" y="-241.33597" x="-524.05823" font-family="MathJax_Main" fill="#000000"><tspan id="tspan9367" y="-241.33597" x="-524.05823">b</tspan></text>
  <text id="text9369" style="word-spacing:0px;letter-spacing:0px" xml:space="preserve" font-size="22.364px" line-height="125%" y="-141.51006" x="-371.41547" font-family="MathJax_Main" fill="#000000"><tspan id="tspan9371" y="-141.51006" x="-371.41547">i</tspan></text>
  <text id="text9373" style="word-spacing:0px;letter-spacing:0px" xml:space="preserve" font-size="22.364px" line-height="125%" y="-236.03662" x="-311.3175" font-family="MathJax_Main" fill="#000000"><tspan id="tspan9375" y="-236.03662" x="-311.3175">w</tspan></text>
  <g id="g17213" fill-rule="evenodd" transform="matrix(.91463 .40429 -.40429 .91463 -121.47 -43.568)">
   <path id="path17215" style="color-rendering:auto;text-decoration-color:#000000;color:#000000;isolation:auto;mix-blend-mode:normal;shape-rendering:auto;solid-color:#000000;block-progression:tb;text-decoration-line:none;text-decoration-style:solid;image-rendering:auto;white-space:normal;text-indent:0;text-transform:none" d="m-439.29-11.502v2.5h179v-2.5h-179z"/>
   <path id="path17217" d="m-444.19-10.252c0-2.76 2.24-5 5-5s5 2.24 5 5-2.24 5-5 5-5-2.24-5-5z"/>
   <path id="path17219" style="color-rendering:auto;text-decoration-color:#000000;color:#000000;isolation:auto;mix-blend-mode:normal;shape-rendering:auto;solid-color:#000000;block-progression:tb;text-decoration-line:none;text-decoration-style:solid;image-rendering:auto;white-space:normal;text-indent:0;text-transform:none" d="m-261.04-89.352v79h2.5v-79h-2.5z" fill="#d40000"/>
   <path id="path17221" d="m-259.79-94.252c2.76 0 5 2.24 5 5s-2.24 5-5 5-5-2.24-5-5 2.24-5 5-5z"/>
   <path id="path17223" style="color-rendering:auto;text-decoration-color:#000000;color:#000000;isolation:auto;mix-blend-mode:normal;shape-rendering:auto;solid-color:#000000;block-progression:tb;text-decoration-line:none;text-decoration-style:solid;image-rendering:auto;white-space:normal;text-indent:0;text-transform:none" d="m-258.79-11.502v2.5h101.5v-2.5h-101.5z" fill="#00f"/>
   <path id="path17225" d="m-259.79-15.252c2.76 0 5 2.24 5 5s-2.24 5-5 5-5-2.24-5-5 2.24-5 5-5z"/>
   <path id="path17227" d="m-162.19-10.252c0-2.76 2.24-5 5-5s5 2.24 5 5-2.24 5-5 5-5-2.24-5-5z"/>
  </g>
  <text id="text17229" style="word-spacing:0px;letter-spacing:0px" xml:space="preserve" font-size="22.364px" line-height="125%" y="-114.64" x="-319.08957" font-family="MathJax_Main" fill="#000000"><tspan id="tspan17231" y="-114.64" x="-319.08957">t</tspan></text>
  <text id="text17233" style="word-spacing:0px;letter-spacing:0px" xml:space="preserve" font-size="22.364px" line-height="125%" y="-195.95728" x="-358.68753" font-family="MathJax_Main" fill="#000000"><tspan id="tspan17235" y="-195.95728" x="-358.68753">d</tspan></text>
  <rect id="rect17339" height="190" width="356" stroke="#000" y="-275.5" x="-563.5" fill="none"/>
 </g>
</svg>

{% highlight javascript linenos %}
/**
 * 
 * @param {[x,y]} a line end-point
 * @param {[x,y]} b line end-point
 * @param {[x,y]} w point to test
 * @returns {object or null} Returns null if ((a == b) and (c != a)), otherwise
 * return an object like:
 *   {
 *      t {real} : fractional distance between a and b,
 *      i {[x, y] : perpendicular point of intersection between a and b,
 *      d {real} : distance from i to the test point (w) 
 *                      d > 0 ==> d is to the right of a-b,
 *                      d < 0 ==> d is to the left of a-b
 *   }
 */
function PointAlongLine(a, b, w) {
    var ax = a[0], ay = a[1];
    var bx = b[0], by = b[1];
    var wx = w[0], wy = w[1];

    if (ax === bx && ay === by) {
        if (ax === wx && ay === wy) {
            return {t: 1, i: [ax, ay], d: 0};
        } else {
            return null;
        }
    }

    var denom = Math.pow(bx - ax, 2) + Math.pow(by - ay, 2);

    var t = ((wx - ax) * (bx - ax) + (wy - ay) * (by - ay)) / (denom);

    var ix = ax + t * (bx - ax);
    var iy = ay + t * (by - ay);

    var dist = ((by - ay) * wx - (bx - ax) * wy + bx * ay - by * ax) / (Math.sqrt(denom));

    return {
        t: t,
        i: [ix, iy],
        d: dist
    };
}
{% endhighlight %}
