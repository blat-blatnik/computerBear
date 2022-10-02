---
layout: post
title: "How to seamlessly sub-divide a circle at any scale"
date: 2020-08-21 13:00:00 +0300
categories: rendering
math: true
---

**Update** I've migrated this blog to [blog.bearcats.nl](https://blog.bearcats.nl). You can find an updated version of this post [here](https://blog.bearcats.nl/seamlessly-subdivide-circle/).

GPUs render graphics strictly in triangles. A box or a square can be perfectly re-created using 2 triangles, and so can many other shapes with jagged edges. Circles on the other hand can be a little bit more problematic.

Triangles are jagged by nature, so there's no way to _perfectly_ build a circle out triangles. But we can approximate. What we normally do, is we divide the circle into $N$ equal sections, and then approximate each section with a triangle.

We usually just choose a fixed value for $N$ ahead of time, and as long as we choose a high enough value, the seams won't be noticeable. This is very simple, and usually it's effective. However, if we _zoom in_ on a circle made like this then the seams can become noticeable.

![Figure 1](../assets/circles/circles.png#center)

> **Figure 1** A circle approximated using 16 triangles rasterized at different scales. From left to right we have 46, 64, 114, and 190 pixel wide circles.

How do we fix this? Well, we could just choose a larger $N$. This will allow us to zoom further in on the circle, but then we've just pushed the problem further back. Not only that, but we would also be wasting resources drawing a highly-detailed circle even the user is zoomed-out.

If your application or game supports _arbitrary_ zooming and you don't want your user's to see the jaggies, or if you don't want to waste resources rendering high-detail circles that are barely visible, then you shouldn't sub-divide your circles into a fixed number of triangles.

Using a little bit of trigonometry, we can calculate a value of $N$ that sub-divides the circles such that the seams aren't noticeable within some known error. What we really want to do is sub-divide the circle until the _curve-to-chord distance_ is small.

![Figure 2](../assets/circles/curve-to-chord.png#center)

> **Figure 2** A diagram of a triangle used to approximate a circle with radius $r$. The curve-to-chord distance is highlighted in red.

As you can see from the diagram, the curve-to-chord distance can be calculated as $r - d$. In turn, $d$ can be calculated simply as $r \cos\theta$. 

Let's call our acceptable curve-to-chord distance $\epsilon$. Now we set $\epsilon = r - r \cos\theta$, and we can use the fact that $\theta = 2\pi / N$ to calculate the number of subdivisions we need to get $\epsilon$. This turns out to be:

$$N = 2 \cos^{-1}\Big(1 - \frac{\epsilon}{r}\Big)$$

Here's some OpenGL code that calculates $N$ like this, and then renders the circle using a [triangle fan](https://en.wikipedia.org/wiki/Triangle_fan).

```c
/* x, y    position of the *center* of the circle
 * error   an acceptable curve-to-chord distance, must be > 0 
 * SCALE   the current zoom-level of your program's view
 *          > 1 means zoomed-in
 *          < 1 means zoomed-out 
 * all parameters should be in the same units (pixels, em, ..) 
 */ 
void drawCircle(double x, double y, double radius, double error) {
    double theta = 2 * acos(1 - error / (SCALE * radius));
    int segments = (int)fmax(ceil(2 * pi / theta), 3);

    glBegin(GL_TRIANGLE_FAN);
    glVertex2d(x, y);
    for (int i = 0; i <= segments; ++i) {
        double segx = x + radius * cos(2 * pi * i / segments);
        double segy = y + radius * sin(2 * pi * i / segments);
        glVertex2d(segx, segy);
    }
    glEnd();
}
```

I actually used a very similar technique to draw the Q-learning agents when I made the [Escape Room](https://github.com/blat-blatnik/Escape-Room). Later I saw that the same techique was described in [Graphics Gems V](http://index-of.co.uk/Game-Development/Programming/Graphics%20Gems%205.pdf#page=178) but I haven't seen it posted anywhere online.

Anyway, using this technique you can render seamless looking circles at any zoom level, as long as you set the error as low as you can according to your memory and performance budget. You can try out how it works for yourself on the canvas below.

## Try it out

<noscript>
  <p style="text-align:center;">
    You have to enable javascript to see the demo.
  </p>
</noscript>
<canvas id="canvas" tabindex="-1">Canvas not supported</canvas>

<script type="text/javascript">
var canvas = document.getElementById("canvas");
var ctx = canvas.getContext("2d", { alpha: false });
var rect = canvas.getBoundingClientRect();
canvas.setAttribute("width", rect.width);
canvas.setAttribute("height", rect.height);
var width = canvas.width;
var height = canvas.height;
var fill = true;
var scale = 1;
var translateX = 0;
var translateY = 0;
var translating = false;
var mouseX = 0;
var mouseY = 0;
var error = 0.5;
var maxSegments = 2000;

ctx.font = "400 18px Arial";
ctx.textAlign = "center";

function reset() {
	scale = 1;
	translateX = 0;
	translateY = 0;
	error = 0.5;
	window.requestAnimationFrame(render);
}

function drawTriangle(x1, y1, x2, y2, x3, y3) {
	ctx.beginPath();
	ctx.moveTo(x1, y1);
	ctx.lineTo(x2, y2);
	ctx.lineTo(x3, y3);
	ctx.closePath();
	ctx.stroke();
	if (fill) ctx.fill();
}

function drawCircle(x, y, r, error) {
	error = Math.min(error, scale * r);
	var theta = 2 * Math.acos(1 - error / (scale * r));
	var segments = Math.min(maxSegments, Math.max(3, Math.ceil(2 * Math.PI / theta)));
	for (var i = 1; i <= segments; ++i) {
		var x1 = x + r * Math.cos(2 * Math.PI * (i - 1) / segments);
		var y1 = y + r * Math.sin(2 * Math.PI * (i - 1) / segments);
		var x2 = x + r * Math.cos(2 * Math.PI * i / segments);
		var y2 = y + r * Math.sin(2 * Math.PI * i / segments);
		drawTriangle(x, y, x1, y1, x2, y2);
	}
	return segments;
}

function render() {
	width = canvas.width;
	height = canvas.height;
	ctx.globalCompositeOperation = "source-over";
	ctx.lineWidth = 1 / scale;
	ctx.fillStyle = "white";
	ctx.fillRect(0, 0, width, height);
	ctx.save();
	ctx.translate(translateX, translateY);
	ctx.scale(scale, scale);
	fill = true;
	ctx.fillStyle = "black";
	drawCircle(1 * width / 3, height / 2, width / 12, error);
	
	fill = false;
	ctx.strokeStyle = "black";
	var segments = drawCircle(2 * width / 3, height / 2, width / 12, error);
	ctx.restore();
	
	ctx.fillStyle = "#444444";
	ctx.fillText("error: " + error.toFixed(3) + "px", width / 2, 32);
	if (segments === maxSegments)
		ctx.fillText(segments + " segments (max)", width / 2, 64);
	else
		ctx.fillText(segments + " segments", width / 2, 64);
}

canvas.addEventListener('wheel', function (event) {
	event.preventDefault();
	var modifierDown = event.getModifierState("Alt") || 
		event.getModifierState("AltGraph") || 
		event.getModifierState("Control") ||
		event.getModifierState("Meta") ||
		event.getModifierState("Shift");
	if (modifierDown) {
		var oldScale = scale;
		if (event.deltaY < 0)
			scale *= 1.1;
		else
			scale /= 1.1;
		var x = event.offsetX;
		var y = event.offsetY;
		var sx = (x - translateX) / oldScale;
		var sy = (y - translateY) / oldScale;
		translateX -= (scale - oldScale) * sx;
		translateY -= (scale - oldScale) * sy;
	} else {
		var delta = 0.025;
		if (error >= 0.5)
			delta = 0.05;
		if (error >= 1)
			delta = 0.10;
		if (error >= 5)
			delta = 0.25;
		if (error >= 10)
			delta = 0.5;
		if (event.deltaY < 0)
			error += delta;
		else
			error -= delta;
		error = Math.max(error, 0.025);
	}
	window.requestAnimationFrame(render);
});

canvas.addEventListener('mousedown', function (event) {
	mouseX = event.screenX;
	mouseY = event.screenY;
	translating = true;
});

window.addEventListener('mouseup', function (event) {
	translating = false;
});

window.addEventListener('mousemove', function (event) {
	if (translating) {
		translateX += event.screenX - mouseX;
		translateY += event.screenY - mouseY;
		mouseX = event.screenX;
		mouseY = event.screenY;
		window.requestAnimationFrame(render);
	}
});

canvas.addEventListener('keydown', function (event) {
	if (event.keyCode == 32) // space
		reset();
});

reset();
</script>

<p style="text-align:center; margin-top:10px;">
  <kbd>SCROLL</kbd> to change the error, <kbd>SHIFT+SCROLL</kbd> to zoom, <kbd>CLICK-DRAG</kbd> the view, <kbd>SPACE</kbd> reset view.
<p>
