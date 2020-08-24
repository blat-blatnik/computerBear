---
layout: page
title: About
permalink: /about/
---

Hi there, I'm Blat Blatnik - a student of artificial intelligence and a huge programming geek. I decided to create [`computerBear`]({{"/" | relative_url}}) to share some of my coding discoveries and frustrations. 

I'm particularly interested in game development and low-level programming in general, and I'm working on my own game right now.

## Snake

Need a break? How about a game of Snake?

<noscript>
  <p style="text-align:center;">
    You have to enable javascript to play Snake.
  </p>
</noscript>
<canvas id="canvas" tabindex="-1" style="height: 50%;">Canvas not supported</canvas>

<script type="text/javascript">
	var Key = {
		Left:   37,
		Up:     38,
		Right:  39,
		Down:   40,
		W:      87,
		A:      65,
		S:      83,
		D:      68,
		Num8:  104,
		Num6:  102,
		Num5:  101,
		Num4:  100,
		Space:  32,
		Enter:  13,
		Escape: 27
	}

	var GameState = {
		Playing:  0,
		Paused:   1,
		Start:    2,
		GameOver: 3
	}

	var Direction = {
		None:  0,
		Left:  1,
		Right: 2,
		Up:    3,
		Down:  4
	}
	
	var GameSpeed = 20; // tiles per second
	var NumTilesX = 40;
	var NumTilesY = 20;

	var canvas = document.getElementById("canvas");
    var ctx = canvas.getContext("2d", { alpha: false });
    var rect = canvas.getBoundingClientRect();
	canvas.setAttribute("width", rect.width);
	canvas.setAttribute("height", rect.height);
	canvas.focus();
	
	var tileWidth = NumTilesX;
	var tileHeight = NumTilesY;
	var input = Array(4);
	var inputCursor = 0;
	var direction = Direction.None;
	var headX = 0;
	var headY = 0;
	var startX = Math.floor(NumTilesX / 2);
	var startY = Math.floor(NumTilesY / 2);
	var segments = Array(NumTilesX * NumTilesY * 2);
	segments[0] = startX;
	segments[1] = startY;
	segments[2] = startX;
	segments[3] = startY;
	segments[4] = startX;
	segments[5] = startY;
	var numSegments = 6;
	var gameState = GameState.Start;
	var score = 0;
	var scoreString = "score: 0";
	var prevTime = getTime();
	var gameTime = 0;
	var fruitBag = Array(2 * NumTilesX * NumTilesY);
	var fruit = [];
	var nextFruitCursor = fruitBag.length;
	var eatenFruit = [];
	var precomputerFruitColors = Array(101);
	var fruitColors = [
		{ r: 255, g: 0, b: 0 },
		{ r: 255, g: 136, b: 0 },
		{ r: 97, g: 46, b: 0 }];

	for (var y = 0; y < NumTilesY; ++y) {
		for (var x = 0; x < NumTilesX; ++x) {
			var idx = 2 * (y * NumTilesX + x);
			fruitBag[idx + 0] = x;
			fruitBag[idx + 1] = y;
		}
	}

	for (var i = 0; i < precomputerFruitColors.length; ++i) {
		var color = getFruitColor(i / (precomputerFruitColors.length - 1));
		precomputerFruitColors[i] = "rgb(" + color.r + "," + color.g + "," + color.b + ")";
	}

	nextFruit();
    canvas.addEventListener('keydown', onKeyDown);
    canvas.addEventListener('focusout', onFocusLost);
	updateAndRender();

	function resetGame() {
		inputCursor = 0;
		direction = Direction.None;
		startX = Math.floor(NumTilesX / 2);
		startY = Math.floor(NumTilesY / 2);
		numSegments = 6;
		segments[0] = startX;
		segments[1] = startY;
		segments[2] = startX;
		segments[3] = startY;
		segments[4] = startX;
		segments[5] = startY;
		headX = 0;
		headY = 0;
		gameState = GameState.Start;
		score = 0;
		scoreString = "score: 0";
		gameTime = 0;
		eatenFruit.length = 0;
		fruit.length = 0;
		nextFruit();
		window.requestAnimationFrame(updateAndRender);
	}

	function updateAndRender() {
		update();
		render();
	}

	function update() {
		var time = getTime();
		var deltaTime = (time - prevTime) / 1000;
		prevTime = time;
		if (deltaTime <= 0.018) // 60 FPS maximum
			deltaTime = 1 / 60;
		else if (deltaTime >= 0.019 && deltaTime <= 0.021)
			deltaTime = 1 / 50;
		else if (deltaTime >= 0.034)
			deltaTime = 1 / 30; // 30 FPS minimum

		if (gameState == GameState.Playing) {
			gameTime += deltaTime;
			
			if (direction == Direction.Left)
				headX -= GameSpeed * deltaTime;
			else if (direction == Direction.Right)
				headX += GameSpeed * deltaTime;
			else if (direction == Direction.Up)
				headY -= GameSpeed * deltaTime;
			else if (direction == Direction.Down)
				headY += GameSpeed * deltaTime;

			var nowX = Math.floor(headX);
			var nowY = Math.floor(headY);

			var moved = direction != Direction.None && (Math.abs(headX) >= 1 || Math.abs(headY) >= 1);
			if (moved) {
				for (var i = numSegments - 2; i >= 2; i -= 2) {
					segments[i + 0] = segments[i - 2];
					segments[i + 1] = segments[i - 1];
				}

				if (headX <= -1) {
					segments[0]--;
					headX++;
				} else if (headX >= +1) {
					segments[0]++;
					headX--;
				} else if (headY <= -1) {
					segments[1]--;
					headY++;
				} else if (headY >= +1) {
					segments[1]++;
					headY--;
				}

				if (segments[0] < 0)
					segments[0] = NumTilesX - 1;
				if (segments[0] >= NumTilesX)
					segments[0] = 0;
				if (segments[1] < 0)
					segments[1] = NumTilesY - 1;
				if (segments[1] >= NumTilesY)
					segments[1] = 0;

				for (var i = 0; i < fruit.length; i += 3) {
					var fruitX = fruit[i + 0];
					var fruitY = fruit[i + 1];
					var spawnTime = fruit[i + 2];
					if (segments[0] == fruitX && segments[1] == fruitY) {
                        console.log('eat fruit. ' + ((fruit.length - 3) / 3) + ' remaining');
						var decay = getFruitDecay(spawnTime);
						score += Math.max(1, Math.ceil((1 - decay) * numSegments / 2));
						scoreString = "score: " + Math.ceil(score);
						eatenFruit.push(fruitX, fruitY, gameTime);
						fruit.splice(i, 3);
						if (fruit.length == 0) {
                            console.log('trying to make next fruit');
							nextFruit();
                        }
						break;
					}
				}

				var lastX = segments[numSegments - 2];
				var lastY = segments[numSegments - 1];
				for (var i = 0; i < eatenFruit.length; i += 3) {
					var x = eatenFruit[i + 0];
					var y = eatenFruit[i + 1];
					if (x == lastX && y == lastY) {
						segments[numSegments++] = lastX;
						segments[numSegments++] = lastY;
						eatenFruit.splice(i, 3);
						i -= 3;
					}
				}
				
				if (numSegments > 6) {
					for (var i = 2; i < numSegments; i += 2) {
						if (segments[0] == segments[i + 0] && segments[1] == segments[i + 1]) {
							gameState = GameState.GameOver;
							break;
						}
					}
				}
			}

			if ((moved || direction == Direction.None) && inputCursor > 0) {
				var newDir = input[0];
				for (var i = 1; i < inputCursor; ++i)
					input[i - 1] = input[i];
				--inputCursor;
				if (!directionsConflict(newDir, direction)) {
					direction = newDir;
					headX = 0;
					headY = 0;
				}
			}
		}
	}

	function render() {
		var width = canvas.width;
		var height = canvas.height;
		tileWidth = width / NumTilesX;
		tileHeight = height / NumTilesY;
		ctx.globalCompositeOperation = "source-over";
		ctx.fillStyle = "white";
		ctx.fillRect(0, 0, width, height);
		drawFruit();
		ctx.fillStyle = "rgb(50,50,50)";
		var snakeLen = 1 + numSegments / 2;
		for (var i = 0; i < eatenFruit.length; i += 3) {
			var x = eatenFruit[i + 0];
			var y = eatenFruit[i + 1];
			var t = eatenFruit[i + 2];
			var dt = gameTime - t;
			var dist = (GameSpeed * dt) / snakeLen;
			var size = 0.8 + (1 - dist) * 0.3;
			var offset = (1 - size) / 2;
			fillRoundedTile(x + offset, y + offset, size, size, size / 4, false);
		}
		
		drawSnake()
		if (gameState == GameState.Playing || gameState == GameState.Paused) {
			ctx.font = "400 20px Serif";
			ctx.textAlign = "center";
			ctx.fillStyle = "rgb(120,120,120)";
			ctx.fillText(scoreString, width / 2, 40);
		}
		
		if (gameState == GameState.Playing)
			window.requestAnimationFrame(updateAndRender);
		else {
			if (gameState != GameState.Start) {
				ctx.globalCompositeOperation = "multiply";
				ctx.fillStyle = "rgb(200,200,200)";
				ctx.fillRect(0, 0, width, height);
				ctx.globalCompositeOperation = "source-over";
			}
			
			ctx.font = "400 32px Serif";
			ctx.textAlign = "center";
			ctx.fillStyle = "rgb(120,120,120)"
			if (gameState == GameState.Start) {
				ctx.fillText("Move to Start", width / 2, height / 4);
			} else if (gameState == GameState.Paused) {
                ctx.fillText("Paused", width / 2, height / 2);
                ctx.font = "400 20px Serif";
				ctx.fillText("Press space to resume", width / 2, height / 2 + 40);
			} else if (gameState == GameState.GameOver) {
				ctx.fillText("Game Over!", width / 2, height / 4);
				ctx.fillText(scoreString, width / 2, height / 4 + 40);
				ctx.font = "400 20px Serif";
				ctx.fillText("Press space to try again", width / 2, height / 4 + 80);
			}
		}
	}

	function drawSnake() {
		var tailX = segments[numSegments - 2];
		var tailY = segments[numSegments - 1];
		var d = getDir(tailX, tailY, segments[numSegments - 4], segments[numSegments - 3]);
		var headOffset = Math.abs(headX) + Math.abs(headY);
		if (d == Direction.Left)
			tailX -= headOffset;
		else if (d == Direction.Right)
			tailX += headOffset;
		else if (d == Direction.Up)
			tailY -= headOffset;
		else if (d == Direction.Down)
			tailY += headOffset
		ctx.fillStyle = "black"
		var lx = segments[0];
		var ly = segments[1];
		fillSnakeSegment(lx + headX, ly + headY, lx, ly);
		for (var i = 2; i < numSegments - 2; i += 2) {
			var x = segments[i + 0];
			var y = segments[i + 1];
			fillSnakeSegment(lx, ly, x, y);
			lx = x;
			ly = y;
		}
		fillSnakeSegment(tailX, tailY, lx, ly);
	}

	function fillSnakeSegment(x1, y1, x2, y2) {
		var minx = Math.min(x1, x2);
		var maxx = Math.max(x1, x2);
		var miny = Math.min(y1, y2);
		var maxy = Math.max(y1, y2);
		var dx = maxx - minx;
		var dy = maxy - miny;
		var w = dx + 1;
		var h = dy + 1
		if (dx > 1) {
			fillSegment(minx - 1, miny, 2, 1);
			fillSegment(maxx, maxy, 2, 1);
		} else if (dy > 1) {
			fillSegment(minx, miny - 1, 1, 2);
			fillSegment(maxx, maxy, 1, 2);
		} else {
			fillSegment(minx, miny, w, h);
			if (minx < 0)
				fillSegment(NumTilesX + minx, miny, w, h);
			else if (maxx > NumTilesX - 1)
				fillSegment(minx - NumTilesX, miny, w, h);
			else if (miny < 0)
				fillSegment(minx, NumTilesY + miny, w, h);
			else if (maxy > NumTilesY - 1)
				fillSegment(minx, miny - NumTilesY, w, h);
		}
	}
	
	function fillSegment(x, y, width, height) {
		x += 0.1;
		y += 0.1;
		width -= 0.2;
        height -= 0.2;
		fillRoundedTile(x, y, width, height, 0.5, true);
	}

	function fillRoundedTile(x, y, width, height, radius, pixelAlign) {
		x *= tileWidth;
		y *= tileHeight;
		width *= tileWidth;
		height *= tileHeight;
        radius *= Math.min(tileWidth, tileHeight);
        if (pixelAlign) {
            x = Math.ceil(x);
            y = Math.ceil(y);
            width = Math.floor(width);
            height = Math.floor(height);
        } else {
            x += 0.5;
            y += 0.5;
        }
        ctx.beginPath();		
        ctx.moveTo(x + radius, y);
		ctx.lineTo(x + width - radius, y);
		ctx.quadraticCurveTo(x + width, y, x + width, y + radius);
		ctx.lineTo(x + width, y + height - radius);
		ctx.quadraticCurveTo(x + width, y + height, x + width - radius, y + height);
		ctx.lineTo(x + radius, y + height);
		ctx.quadraticCurveTo(x, y + height, x, y + height - radius);
		ctx.lineTo(x, y + radius);
		ctx.quadraticCurveTo(x, y, x + radius, y);
		ctx.closePath();
		ctx.fill();
	}

	function drawFruit() {
		for (var i = 0; i < fruit.length; i += 3) {
			var x = fruit[i + 0];
			var y = fruit[i + 1];
			var spawnTime = fruit[i + 2];
			var decay = getFruitDecay(spawnTime);
			ctx.fillStyle = precomputerFruitColors[Math.floor(decay * 100)];
			var size = 1 - smoothstep(0, 1, decay) / 3;
			var offset = (1 - size) / 2;
			fillRoundedTile(x + offset, y + offset, size, size, 0.25 * size, false);
		}
	}

	function getFruitDecay(spawnTime) {
		return clamp((gameTime - spawnTime) / (5 + numSegments / 20), 0, 1);
	}

	function nextFruit() {
		if (numSegments < 2 * NumTilesX * NumTilesY) {
			var fruitX = 0;
			var fruitY = 0;
			do {
				if (nextFruitCursor >= fruitBag.length) {
					// shuffle the fruit bag
					for (var i = fruitBag.length - 2; i >= 2; i -= 2) {
						var r = 2 * Math.floor(Math.random() * i / 2);
						var tx = fruitBag[i + 0];
						var ty = fruitBag[i + 1];
						fruitBag[i + 0] = fruitBag[r + 0];
						fruitBag[i + 1] = fruitBag[r + 1];
						fruitBag[r + 0] = tx;
						fruitBag[r + 1] = ty;
					}
					nextFruitCursor = 0;
				}
				fruitX = fruitBag[nextFruitCursor++];
				fruitY = fruitBag[nextFruitCursor++];
            } while (snakeAtTile(fruitX, fruitY));
            
            console.log('made fruit at (' + fruitX + ', ' + fruitY + ')');

			fruit.push(fruitX, fruitY, gameTime);
			if (((numSegments - 20) / 2000) > Math.random()) {
				// make a special fruit pattern
				var x1;
				var x2;
				var x3;
				var y1;
				var y2;
				var y3;
				var rand = Math.random();
				if (rand < 0.25 && fruitX < NumTilesX - 1 && fruitY < NumTilesY - 1) {
					x1 = fruitX + 1;
					y1 = fruitY;
					x2 = fruitX;
					y2 = fruitY + 1;
					x3 = fruitX + 1;
					y3 = fruitY + 1;
				} else if (rand < 0.5 && fruitX < NumTilesX - 3) {
					x1 = fruitX + 1;
					y1 = fruitY;
					x2 = fruitX + 2;
					y2 = fruitY;
					x3 = fruitX + 3;
					y3 = fruitY;
				} else if (rand < 0.75 && fruitY < NumTilesY - 3) {
					x1 = fruitX;
					y1 = fruitY + 1;
					x2 = fruitX;
					y2 = fruitY + 2;
					x3 = fruitX;
					y3 = fruitY + 3;
				} else if (fruitX < NumTilesX - 2 && fruitY < NumTilesY - 1) {
					x1 = fruitX + 1;
					y1 = fruitY;
					x2 = fruitX + 1;
					y2 = fruitY + 1;
					x3 = fruitX + 2;
					y3 = fruitY + 1;
				}

				if (!snakeAtTile(x1, y1) && !snakeAtTile(x2, y2) && !snakeAtTile(x3, y3))
					fruit.push(x1, y1, gameTime, x2, y2, gameTime, x3, y3, gameTime);
			}
		}
	}

	function getFruitColor(interpolationFactor) {
		var idx = clamp(interpolationFactor * fruitColors.length, 0, fruitColors.length - 1);
		var color1 = fruitColors[Math.floor(idx)];
		var color2 = fruitColors[Math.ceil(idx)];
		var factor = idx - Math.floor(idx);
		return {
			r: lerp(color1.r, color2.r, smoothstep(0, 1, factor)),
			g: lerp(color1.g, color2.g, smoothstep(0, 1, factor)),
			b: lerp(color1.b, color2.b, smoothstep(0, 1, factor)),
		};
    }
    
    function onFocusLost(event) {
        if (gameState == GameState.Playing)
            gameState = GameState.Paused;
    }

	function onKeyDown(event) {
		var prevGameState = gameState;

		if (event.keyCode == Key.Space || event.keyCode == Key.Enter || event.keyCode == Key.Escape) {
            event.preventDefault();
			if (gameState == GameState.Playing)
				gameState = GameState.Paused;
			else if (gameState == GameState.Paused)
				gameState = GameState.Playing;
			else if (gameState == GameState.GameOver)
				resetGame();
		} else {
            event.preventDefault();
			var newDir = Direction.None;
			switch (event.keyCode) {
				case Key.Left:
				case Key.A:
				case Key.Num4:
					newDir = Direction.Left;
					break;
				case Key.Up:
				case Key.W:
				case Key.Num8:
					newDir = Direction.Up;
					break;
				case Key.Right:
				case Key.D:
				case Key.Num6:
					newDir = Direction.Right;
					break;
				case Key.Down:
				case Key.S:
				case Key.Num5:
					newDir = Direction.Down;
					break;
				default: break;
			}

			if (newDir != Direction.None && gameState == GameState.Start || gameState == GameState.Playing) {
				if (inputCursor < input.length) {
					var prevDir;
					if (inputCursor == 0)
						prevDir = direction;
					else
						prevDir = input[inputCursor - 1];

					if (!directionsConflict(newDir, prevDir))
						input[inputCursor++] = newDir;
				}

				if (gameState == GameState.Start)
					gameState = GameState.Playing;
			}
		}

		if (gameState == GameState.Playing && prevGameState != GameState.Playing)
			window.requestAnimationFrame(updateAndRender);
	}

	function snakeAtTile(x, y) {
		for (var i = 0; i < numSegments; i += 2)
			if (segments[i + 0] == x && segments[i + 1] == y)
				return true;
		return false;
	}

	function directionsConflict(dir1, dir2) {
		if (dir1 == Direction.None || dir2 == Direction.None)
			return false;
		else {
			return (dir1 == Direction.Left || dir1 == Direction.Right) == (dir2 == Direction.Left || dir2 == Direction.Right) ||
				   (dir1 == Direction.Up || dir1 == Direction.Down) == (dir2 == Direction.Up || dir2 == Direction.Down);
		}
	}

	function getDir(fromX, fromY, toX, toY) {
		var dx = toX - fromX;
		var dy = toY - fromY;
		if ((dx > 0 && dx < NumTilesX / 2) || (dx < -NumTilesX / 2))
			return Direction.Right;
		else if ((dx < 0 && -dx < NumTilesX / 2) || (dx > NumTilesX / 2))
			return Direction.Left;
		else if ((dy > 0 && dy < NumTilesY / 2) || (dy < -NumTilesY / 2))
			return Direction.Down;
		else if ((dy < 0 && -dy < NumTilesY / 2) || (dy > NumTilesY / 2))
			return Direction.Up;
		else
			return Direction.None;
	}

	function getTime() {
		if (window.performance.now)
			return window.performance.now();
		else if (window.performance.webkitNow)
			return window.performance.webkitNow();
		else
			return new Date().getTime();
	}

	function lerp(a, b, amount) {
		return a + (b - a) * amount;
	}

	function clamp(x, min, max) {
		return Math.min(Math.max(x, min), max);
	}

	function smoothstep(edge0, edge1, x) {
		var t = clamp((x - edge0) / (edge1 - edge0), 0, 1);
		return t * t * (3 - 2 * t);
	}
</script>