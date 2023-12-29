<script lang="ts">
	import Overlay from '$lib/components/Overlay.svelte';
	import { onMount } from 'svelte';

	let frames = 0;
	let response = '';
	let thinking = false;

	let prompt = `Analyze a pong game where you're the left player. Your move options are 'Q' to go up, 'A' to go down, and 'Space' to stay still. When presenting the ball's position and direction, pay special attention to the direction of the arrow. Your task is to determine the best move to counter the ball's trajectory. Keep in mind, the only valid outputs are 'Q', 'A', or 'Space'. Follow these steps in your analysis:

1. **Identify the Ball's Direction**: Start by clearly stating the direction the ball is moving towards.
2. **Assess Your Position**: Evaluate your paddle's position relative to the ball.
3. **Predict the Ball's Trajectory**: Consider where the ball will be in the next few moments.
4. **Choose the Optimal Move**: Based on the ball's trajectory and your position, decide whether moving up ('Q'), down ('A'), or staying still ('Space') is the best option.

Conclude your analysis with a clear and succinct final output, formatted as: "Best input: 'X'", where X is the specific key ('Q', 'A', or 'Space') that represents the optimal move. It's crucial to adhere strictly to these instructions, as the valid outputs are limited to 'Q', 'A', or 'Space'. A concise and accurate conclusion is expected, without any additional verbosity. 

A reward of $1000 is promised for strict adherence to these guidelines.`;

	// `Analyze a pong game where you're the left player. Your moves are 'Q' to go up, 'A' to go down, and 'Space' to stay still. The ball's position and direction are shown. The task is to choose the best move to counter the ball. Remember, the valid outputs are strictly 'Q', 'A', or 'Space'. The analysis can be detailed, but the conclusion must be clear and succinct, specifying only one of these keys as the optimal input. The final output should be in the format: "Best input: 'X'", where X is either 'Q', 'A', or 'Space'. Emphasizing again, the valid outputs are only 'Q', 'A', or 'Space'. No verbosity, just the key action. A $1000 tip is promised for strict adherence to these instructions.`,

	function extractKeys(text) {
		// Regular expression to find the keys 'Q', 'A', and 'Space'
		const regex = /\b(Q|A|Space)\b/g;

		// Array to hold the extracted keys
		let keys = [];

		// Match the text against the regex
		let match;
		while ((match = regex.exec(text)) !== null) {
			// Add the found key to the keys array
			keys.push(match[0]);
		}

		return keys;
	}

	function sleep(ms) {
		return new Promise((resolve) => setTimeout(resolve, ms));
	}

	onMount(() => {
		const canvas = document.getElementById('pongCanvas');
		const context = canvas.getContext('2d');

		let direction;

		let leftPaddleY = canvas.height / 2 - 30;
		const paddleHeight = 80;
		const paddleWidth = 10;
		let leftScore = 0;
		let rightScore = 0;

		// Ball properties
		const ball = {
			x: canvas.width / 2,
			y: canvas.height / 2,
			radius: 10,
			speed: 2,
			velocityX: 2,
			velocityY: 2
		};

		function drawRect(x, y, width, height, color) {
			context.fillStyle = color;
			context.fillRect(x, y, width, height);
		}

		function drawCircle(x, y, radius, color) {
			context.fillStyle = color;
			context.beginPath();
			context.arc(x, y, radius, 0, Math.PI * 2, false);
			context.closePath();
			context.fill();
		}

		function drawText(text, x, y, color) {
			context.fillStyle = color;
			context.font = '32px Arial';
			context.fillText(text, x, y);
		}

		function drawArrow(x, y, radius, velocityX, velocityY) {
			context.fillStyle = '#000';
			context.beginPath();

			// Calculate angle of the arrow based on ball velocity
			const angle = Math.atan2(velocityY, velocityX);

			// Convert angle to degrees for better readability
			const angleInDegrees = (angle * 180) / Math.PI;

			// Determine direction based on angle

			if (angleInDegrees >= -90 && angleInDegrees <= 90) {
				direction = 'Right';
			} else {
				direction = 'Left';
			}

			console.log(`The object is heading ${direction}.`);

			// Arrow head coordinates
			const arrowHeadX = x + radius * Math.cos(angle);
			const arrowHeadY = y + radius * Math.sin(angle);

			// Arrow base coordinates, offset by 90 degrees
			const baseLeftX = x + radius * Math.cos(angle + Math.PI / 2);
			const baseLeftY = y + radius * Math.sin(angle + Math.PI / 2);
			const baseRightX = x + radius * Math.cos(angle - Math.PI / 2);
			const baseRightY = y + radius * Math.sin(angle - Math.PI / 2);

			// Draw arrow stem

			// Arrow stem end point, positioned at the edge of the ball
			const stemEndX = x - radius * Math.cos(angle);
			const stemEndY = y - radius * Math.sin(angle);

			// Draw the stem of the arrow
			context.strokeStyle = '#000'; // Black color for the arrow stem
			context.lineWidth = 6; // Make the stem thicker
			context.beginPath();
			context.moveTo(x, y); // Start at the ball's center
			context.lineTo(stemEndX, stemEndY); // Draw to the edge of the ball
			context.stroke(); // Render the stem

			// Draw arrow head and base
			context.fillStyle = '#000'; // Set the color for the arrow head and base
			context.beginPath();
			context.moveTo(arrowHeadX, arrowHeadY);
			context.lineTo(baseLeftX, baseLeftY);
			context.lineTo(baseRightX, baseRightY);
			context.lineTo(arrowHeadX, arrowHeadY);
			context.closePath();
			context.fill();
		}

		function resetBall() {
			ball.x = canvas.width / 2;
			ball.y = canvas.height / 2;
			ball.speed = 2;
			ball.velocityX = -ball.velocityX;
		}

		function update() {
			ball.x += ball.velocityX;
			ball.y += ball.velocityY;

			// Simple AI to control the right paddle
			let computerLevel = 0.1;
			let rightPaddleY = ball.y - paddleHeight / 2;

			if (ball.y + ball.radius > canvas.height || ball.y - ball.radius < 0) {
				ball.velocityY = -ball.velocityY;
			}

			// Left paddle
			if (ball.x < 0) {
				if (ball.y > leftPaddleY && ball.y < leftPaddleY + paddleHeight) {
					ball.velocityX = -ball.velocityX;
				} else {
					rightScore++;
					resetBall();
				}
			}

			// Right paddle
			if (ball.x > canvas.width) {
				if (ball.y > rightPaddleY && ball.y < rightPaddleY + paddleHeight) {
					ball.velocityX = -ball.velocityX;
				} else {
					leftScore++;
					resetBall();
				}
			}
		}

		function render() {
			// Clear canvas
			drawRect(0, 0, canvas.width, canvas.height, '#000');

			// Draw scores
			drawText(leftScore, canvas.width / 4, canvas.height / 5, '#FFF');
			drawText(rightScore, (3 * canvas.width) / 4, canvas.height / 5, '#FFF');

			// Draw paddles
			drawRect(0, leftPaddleY, paddleWidth, paddleHeight, '#FFF');
			drawRect(
				canvas.width - paddleWidth,
				ball.y - paddleHeight / 2,
				paddleWidth,
				paddleHeight,
				'#FFF'
			);

			// Draw ball
			drawCircle(ball.x, ball.y, ball.radius, '#FFF');
			// Draw arrow on the ball
			let arrowDirection = ball.velocityX > 0 ? 'right' : 'left';
			drawArrow(ball.x, ball.y, ball.radius, ball.velocityX, ball.velocityY);
		}

		async function game(key) {
			frames += 1;
			if (key === 'Q' && leftPaddleY > 0) {
				leftPaddleY -= 20;
			} else if (key === 'A' && leftPaddleY < canvas.height - paddleHeight) {
				leftPaddleY += 20;
			} else if (key === 'Space') {
				// Not moving
			}

			update();
			render();

			if (direction === 'Left') {
				if (frames % 20 == 0) {
					game(await getBestKey());
				} else {
					await sleep(1000 / 50);
					game('');
				}
			} else {
				await sleep(1000 / 50);
				game('');
			}
		}

		async function getBestKey() {
			thinking = true;
			const res = await fetch('http://localhost:11434/api/generate', {
				method: 'POST',
				headers: {
					Accept: 'application/json',
					'Content-Type': 'application/json'
				},
				body: JSON.stringify({
					model: 'llava:13b',
					prompt: prompt,
					images: [canvas.toDataURL('image/jpeg').split(';base64,')[1]],
					stream: false
				})
			})
				.then(async (res) => {
					if (!res.ok) throw await res.json();
					return res.json();
				})
				.catch((err) => {
					console.log(err);

					return null;
				});

			response = res.response;

			const keys = extractKeys(res.response);

			console.log(keys, res);

			thinking = false;

			return keys.length > 0 ? keys.at(-1) : 'Space';
		}

		// Control the left paddle
		// document.addEventListener('keydown', (event) => {
		// 	const key = event.key.toLowerCase();
		// 	if (key === 'q' && leftPaddleY > 0) {
		// 		leftPaddleY -= 20;
		// 		game();
		// 	} else if (key === 'a' && leftPaddleY < canvas.height - paddleHeight) {
		// 		leftPaddleY += 20;
		// 		game();
		// 	} else if (key === ' ') {
		// 		game();
		// 	}
		// });

		game('');

		// Loop the game
		// setInterval(game, 1000 / 50); // 50 times per second
		// Analyze a pong game where you're the left player. Your moves are 'Q' to go up, 'A' to go down, and 'Space' to stay still. The ball's position and direction are shown. The task is to choose the best move to counter the ball. Remember, the valid outputs are strictly 'Q', 'A', or 'Space'. The analysis can be detailed, but the conclusion must be clear and succinct, specifying only one of these keys as the optimal input. The final output should be in the format: "Best input: 'X'", where X is either 'Q', 'A', or 'Space'. Emphasizing again, the valid outputs are only 'Q', 'A', or 'Space'. No verbosity, just the key action. A $1000 tip is promised for strict adherence to these instructions.
	});
</script>

<div class="w-full h-screen flex justify-center">
	<div class="my-10 flex flex-col w-[600px]">
		<div class=" w-[600px] h-[400px]">
			<Overlay show={thinking} content={'Thinking...'}>
				<canvas id="pongCanvas" width="600" height="400"></canvas>
			</Overlay>
		</div>

		<div class=" my-10">
			{response}
		</div>
	</div>
</div>
