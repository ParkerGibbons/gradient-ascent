<script lang="ts">
	import { onMount } from 'svelte';

	export let settings: {
		colors: string[];
		speed: number;
		isAnimated: boolean;
		animationPreset: string;
		scale: number;
		chaos: number;
	};

	let canvas: HTMLCanvasElement;
    let program: WebGLProgram;
    let gl: WebGLRenderingContext;

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
  uniform float u_scale;
  uniform float u_chaos;
  uniform int u_animationPreset;

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

  vec3 mod289(vec3 x) { return x - floor(x * (1.0 / 289.0)) * 289.0; }
  vec2 mod289(vec2 x) { return x - floor(x * (1.0 / 289.0)) * 289.0; }
  vec3 permute(vec3 x) { return mod289(((x*34.0)+1.0)*x); }

  float snoise(vec2 v) {
    const vec4 C = vec4(0.211324865405187, 0.366025403784439,
             -0.577350269189626, 0.024390243902439);
    vec2 i  = floor(v + dot(v, C.yy) );
    vec2 x0 = v -   i + dot(i, C.xx);
    vec2 i1;
    i1 = (x0.x > x0.y) ? vec2(1.0, 0.0) : vec2(0.0, 1.0);
    vec4 x12 = x0.xyxy + C.xxzz;
    x12.xy -= i1;
    i = mod289(i);
    vec3 p = permute( permute( i.y + vec3(0.0, i1.y, 1.0 ))
      + i.x + vec3(0.0, i1.x, 1.0 ));
    vec3 m = max(0.5 - vec3(dot(x0,x0), dot(x12.xy,x12.xy),
    dot(x12.zw,x12.zw)), 0.0);
    m = m*m ;
    m = m*m ;
    vec3 x = 2.0 * fract(p * C.www) - 1.0;
    vec3 h = abs(x) - 0.5;
    vec3 ox = floor(x + 0.5);
    vec3 a0 = x - ox;
    m *= 1.79284291400159 - 0.85373472095314 * ( a0*a0 + h*h );
    vec3 g;
    g.x  = a0.x  * x0.x  + h.x  * x0.y;
    g.yz = a0.yz * x12.xz + h.yz * x12.yw;
    return 130.0 * dot(m, g);
  }

  float getAnimationValue(vec2 st) {
    float t = u_time * u_speed;
    if (u_animationPreset == 0) { // Wave
      return abs(sin(t + st.x * u_scale + st.y * u_scale));
    } else if (u_animationPreset == 1) { // Spiral
      float angle = atan(st.y - 0.5, st.x - 0.5);
      float dist = length(st - 0.5);
      return abs(sin(t + angle * 3.0 + dist * 10.0 * u_scale));
    } else if (u_animationPreset == 2) { // Noise
      return (snoise(st * u_scale + t * 0.1) + 1.0) * 0.5;
    } else { // Ripple
      vec2 center = vec2(0.5, 0.5);
      float dist = length(st - center);
      return abs(sin(t + dist * 20.0 * u_scale));
    }
  }

  void main() {
    vec2 st = gl_FragCoord.xy / u_resolution;
    float t = u_isAnimated ? getAnimationValue(st) : st.x;
    t = mix(t, snoise(st * 10.0), u_chaos); // Add chaos
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
			gl.uniform1f(gl.getUniformLocation(program, 'u_scale'), settings.scale);
			gl.uniform1f(gl.getUniformLocation(program, 'u_chaos'), settings.chaos);
			gl.uniform1i(
				gl.getUniformLocation(program, 'u_animationPreset'),
				['wave', 'spiral', 'noise', 'ripple'].indexOf(settings.animationPreset)
			);

			gl.drawArrays(gl.TRIANGLES, 0, 6);

			requestAnimationFrame(render);
		};
		requestAnimationFrame(render);
	});

	// Reactive statement to update animation preset
	$: if (gl && program) {
		const animationPresetIndex = ['wave', 'spiral', 'noise', 'ripple'].indexOf(
			settings.animationPreset
		);
		gl.uniform1i(gl.getUniformLocation(program, 'u_animationPreset'), animationPresetIndex);
	}
</script>

<canvas bind:this={canvas} class="h-full w-full" />
