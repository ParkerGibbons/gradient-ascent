<script lang="ts">
	import { onMount } from 'svelte';

	export let settings: {
		colors: string[];
		speed: number;
		isAnimated: boolean;
	};

	let canvas: HTMLCanvasElement;

	const hexToRgb = (hex: string) => {
		const bigint = parseInt(hex.slice(1), 16);
		const r = (bigint >> 16) & 255;
		const g = (bigint >> 8) & 255;
		const b = bigint & 255;
		return [r / 255, g / 255, b / 255];
	};

	onMount(() => {
		const gl = canvas.getContext('webgl');
		if (!gl) return;

		const vertexShaderSource = `
        attribute vec4 a_position;
        void main() {
          gl_Position = a_position;
        }
      `;

		const fragmentShaderSource = `
  precision mediump float;
  uniform vec2 u_resolution;
  uniform float u_time;
  uniform vec3 u_colors[6];
  uniform int u_colorCount;
  uniform float u_speed;
  uniform bool u_isAnimated;

  vec3 getGradientColor(float t) {
    float segment = 1.0 / float(u_colorCount - 1);
    if (u_colorCount == 2) {
      return mix(u_colors[0], u_colors[1], t);
    } else if (u_colorCount == 3) {
      if (t < segment) {
        return mix(u_colors[0], u_colors[1], t / segment);
      } else {
        return mix(u_colors[1], u_colors[2], (t - segment) / segment);
      }
    } else if (u_colorCount == 4) {
      if (t < segment) {
        return mix(u_colors[0], u_colors[1], t / segment);
      } else if (t < 2.0 * segment) {
        return mix(u_colors[1], u_colors[2], (t - segment) / segment);
      } else {
        return mix(u_colors[2], u_colors[3], (t - 2.0 * segment) / segment);
      }
    } else if (u_colorCount == 5) {
      if (t < segment) {
        return mix(u_colors[0], u_colors[1], t / segment);
      } else if (t < 2.0 * segment) {
        return mix(u_colors[1], u_colors[2], (t - segment) / segment);
      } else if (t < 3.0 * segment) {
        return mix(u_colors[2], u_colors[3], (t - 2.0 * segment) / segment);
      } else {
        return mix(u_colors[3], u_colors[4], (t - 3.0 * segment) / segment);
      }
    } else {
      if (t < segment) {
        return mix(u_colors[0], u_colors[1], t / segment);
      } else if (t < 2.0 * segment) {
        return mix(u_colors[1], u_colors[2], (t - segment) / segment);
      } else if (t < 3.0 * segment) {
        return mix(u_colors[2], u_colors[3], (t - 2.0 * segment) / segment);
      } else if (t < 4.0 * segment) {
        return mix(u_colors[3], u_colors[4], (t - 3.0 * segment) / segment);
      } else {
        return mix(u_colors[4], u_colors[5], (t - 4.0 * segment) / segment);
      }
    }
  }

  void main() {
    vec2 st = gl_FragCoord.xy / u_resolution;
    float t = u_isAnimated ? abs(sin(u_time * u_speed + st.x + st.y)) : st.x;
    vec3 color = getGradientColor(t);
    gl_FragColor = vec4(color, 1.0);
  }
`;

		const createShader = (gl: WebGLRenderingContext, type: number, source: string) => {
			const shader = gl.createShader(type);
			gl.shaderSource(shader, source);
			gl.compileShader(shader);
			if (!gl.getShaderParameter(shader, gl.COMPILE_STATUS)) {
				console.error(gl.getShaderInfoLog(shader));
				gl.deleteShader(shader);
				return null;
			}
			return shader;
		};

		const vertexShader = createShader(gl, gl.VERTEX_SHADER, vertexShaderSource);
		const fragmentShader = createShader(gl, gl.FRAGMENT_SHADER, fragmentShaderSource);
		if (!vertexShader || !fragmentShader) return;

		const program = gl.createProgram();
		gl.attachShader(program, vertexShader);
		gl.attachShader(program, fragmentShader);
		gl.linkProgram(program);
		if (!gl.getProgramParameter(program, gl.LINK_STATUS)) {
			console.error(gl.getProgramInfoLog(program));
			return;
		}
		gl.useProgram(program);

		const positionBuffer = gl.createBuffer();
		gl.bindBuffer(gl.ARRAY_BUFFER, positionBuffer);
		const positions = [-1, -1, 1, -1, -1, 1, -1, 1, 1, -1, 1, 1];
		gl.bufferData(gl.ARRAY_BUFFER, new Float32Array(positions), gl.STATIC_DRAW);

		const positionAttributeLocation = gl.getAttribLocation(program, 'a_position');
		gl.enableVertexAttribArray(positionAttributeLocation);
		gl.vertexAttribPointer(positionAttributeLocation, 2, gl.FLOAT, false, 0, 0);

		const resolutionUniformLocation = gl.getUniformLocation(program, 'u_resolution');
		const timeUniformLocation = gl.getUniformLocation(program, 'u_time');
		const color1UniformLocation = gl.getUniformLocation(program, 'u_color1');
		const color2UniformLocation = gl.getUniformLocation(program, 'u_color2');
		const speedUniformLocation = gl.getUniformLocation(program, 'u_speed');

		const resizeCanvasToDisplaySize = (canvas: HTMLCanvasElement) => {
			const displayWidth = canvas.clientWidth;
			const displayHeight = canvas.clientHeight;

			if (canvas.width !== displayWidth || canvas.height !== displayHeight) {
				canvas.width = displayWidth;
				canvas.height = displayHeight;
			}
		};

		const colorUniformLocations = Array(6)
			.fill(0)
			.map((_, i) => gl.getUniformLocation(program, `u_colors[${i}]`));
		const colorCountUniformLocation = gl.getUniformLocation(program, 'u_colorCount');
		const isAnimatedUniformLocation = gl.getUniformLocation(program, 'u_isAnimated');

		const render = (time: number) => {
			resizeCanvasToDisplaySize(canvas);

			gl.viewport(0, 0, gl.drawingBufferWidth, gl.drawingBufferHeight);
			gl.clear(gl.COLOR_BUFFER_BIT);

			gl.uniform2f(resolutionUniformLocation, gl.drawingBufferWidth, gl.drawingBufferHeight);
			gl.uniform1f(timeUniformLocation, time * 0.001);

			settings.colors.forEach((color, i) => {
				gl.uniform3fv(colorUniformLocations[i], hexToRgb(color));
			});
			gl.uniform1i(colorCountUniformLocation, settings.colors.length);
			gl.uniform1f(speedUniformLocation, settings.speed);
			gl.uniform1i(isAnimatedUniformLocation, settings.isAnimated ? 1 : 0);

			gl.drawArrays(gl.TRIANGLES, 0, 6);

			requestAnimationFrame(render);
		};
		requestAnimationFrame(render);
	});
</script>

<canvas bind:this={canvas} class="h-full w-full" />
