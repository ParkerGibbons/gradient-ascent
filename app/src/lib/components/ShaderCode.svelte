<script lang="ts">
	import { Button } from '$lib/components/ui/button';
	import { Textarea } from '$lib/components/ui/textarea';

	export let settings: {
		colors: string[];
		speed: number;
		isAnimated: boolean;
		animationPreset: string;
		scale: number;
		chaos: number;
	};

	$: shaderCode = generateShaderCode(settings);

	function generateShaderCode(settings) {
		return `
precision mediump float;
uniform vec2 u_resolution;
uniform float u_time;
uniform vec3 u_colors[${settings.colors.length}];
uniform float u_speed;
uniform bool u_isAnimated;
uniform float u_scale;
uniform float u_chaos;
uniform int u_animationPreset;

${generateGradientColorFunction(settings.colors.length)}

${generateSimplexNoiseFunction()}

${generateAnimationValueFunction()}

void main() {
  vec2 st = gl_FragCoord.xy / u_resolution;
  float t = ${settings.isAnimated ? 'getAnimationValue(st)' : 'st.x'};
  t = mix(t, snoise(st * 10.0), ${settings.chaos.toFixed(2)}); // Add chaos
  vec3 color = getGradientColor(t);
  gl_FragColor = vec4(color, 1.0);
}`;
	}

	function generateGradientColorFunction(colorCount) {
		return `
vec3 getGradientColor(float t) {
  float segment = 1.0 / ${colorCount - 1}.0;
  ${generateColorMixCode(colorCount)}
}`;
	}

	function generateColorMixCode(colorCount) {
		let code = '';
		for (let i = 0; i < colorCount - 1; i++) {
			code += `
  if (t < ${i + 1}.0 * segment) {
    return mix(u_colors[${i}], u_colors[${i + 1}], (t - ${i}.0 * segment) / segment);
  }`;
		}
		code += `
  return u_colors[${colorCount - 1}];`;
		return code;
	}

	function generateSimplexNoiseFunction() {
		return `
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
}`;
	}

	function generateAnimationValueFunction() {
		return `
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
    le + t * 0.1) + 1.0) * 0.5;
  } else { // Ripple
    vec2 center = vec2(0.5, 0.5);
    float dist = length(st - center);
    return abs(sin(t + dist * 20.0 * u_scale));
  }
}`;
	}

	function copyToClipboard() {
		navigator.clipboard.writeText(shaderCode).then(() => {
			alert('Shader code copied to clipboard!');
		});
	}
</script>

<div class="space-y-4 p-4">
	<Textarea rows="20" readonly value={shaderCode} />
	<Button on:click={copyToClipboard}>Copy to Clipboard</Button>
</div>
