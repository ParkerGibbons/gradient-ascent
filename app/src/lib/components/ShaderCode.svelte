<script lang="ts">
	import { Button } from '$lib/components/ui/button';
	import { Textarea } from '$lib/components/ui/textarea';

	export let settings: {
		colors: string[];
		speed: number;
		isAnimated: boolean;
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
  
  vec3 getGradientColor(float t) {
    float segment = 1.0 / ${settings.colors.length - 1}.0;
    ${generateColorMixCode(settings.colors.length)}
  }
  
  void main() {
    vec2 st = gl_FragCoord.xy / u_resolution;
    float t = ${settings.isAnimated ? 'abs(sin(u_time * u_speed + st.x + st.y))' : 'st.x'};
    vec3 color = getGradientColor(t);
    gl_FragColor = vec4(color, 1.0);
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
