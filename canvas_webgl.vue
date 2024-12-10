<template>
  <canvas ref="canvas" class="background-canvas"></canvas>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from "vue";
import { useQuasar } from "quasar";

const $q = useQuasar();
const canvas = ref(null);

onMounted(async () => {
  const gl = canvas.value.getContext("webgl");
  if (!gl) {
    console.error("WebGL not supported");
    return;
  }

  let width = (canvas.value.width = window.innerWidth);
  let height = (canvas.value.height = window.innerHeight);

  //listen for resize
  function resizeCanvas() {
    width = canvas.value.width = window.innerWidth;
    height = canvas.value.height = window.innerHeight;
    gl.viewport(0, 0, width, height);
  }
  window.addEventListener("resize", resizeCanvas);

  //mouse tracking
  let mouse = { x: width / 2, y: height / 2 };
  function onMouseMove(e) {
    mouse.x = e.clientX;
    mouse.y = e.clientY;
  }
  window.addEventListener("mousemove", onMouseMove);

  //circle data
  const circleColors = [
    [18 / 255, 113 / 255, 1.0],
    [221 / 255, 74 / 255, 1.0],
    [100 / 255, 220 / 255, 1.0],
    [200 / 255, 50 / 255, 50 / 255],
    [180 / 255, 180 / 255, 50 / 255],
    [140 / 255, 100 / 255, 1.0],
  ];

  let circles = [];
  function initCircles() {
    circles = [];
    const baseRadius = (width + height) * 0.2;
    //5 moving circles, add more for more chaos
    for (let i = 0; i < 5; i++) {
      const radius = baseRadius;
      const x = Math.random() * width;
      const y = Math.random() * height;
      const speedMultiplier = Math.random() * 4 + 1;
      const vx = (Math.random() - 0.5) * speedMultiplier;
      const vy = (Math.random() - 0.5) * speedMultiplier;
      circles.push({
        x,
        y,
        radius,
        color: circleColors[i],
        vx,
        vy,
        interactive: false,
      });
    }

    //interactive circle
    const interactiveRadius = (width + height) * 0.1;
    circles.push({
      x: width / 2,
      y: height / 2,
      radius: interactiveRadius,
      color: circleColors[5],
      vx: 0,
      vy: 0,
      interactive: true,
    });
  }

  initCircles();

  //create shaders
  const vertexSrc = `
    attribute vec2 a_position;
    varying vec2 v_uv;
    void main(void) {
      v_uv = a_position * 0.5 + 0.5; // map from [-1,1] to [0,1]
      v_uv.y = 1.0 - v_uv.y; // flip y
      gl_Position = vec4(a_position, 0.0, 1.0);
    }
  `;

  const fragmentSrc = `
precision mediump float;
varying vec2 v_uv;

uniform vec2 u_resolution;
uniform bool u_darkMode;
uniform int u_circleCount;
uniform vec3 u_circlesColor[6];
uniform vec3 u_circlesPosRad[6];
uniform vec2 u_mouse;

void main(void) {
    vec2 st = v_uv * u_resolution;

    //background gradient
    vec3 topColor = vec3(108.0/255.0, 0.0, 162.0/255.0);
    vec3 bottomColor = vec3(0.0, 17.0/255.0, 82.0/255.0);
    vec3 bgColor = mix(topColor, bottomColor, st.y / u_resolution.y);

    float fieldSum = 0.0;
    vec3 weightedColorSum = vec3(0.0);
    
    //calculate field for each circle
    for (int i = 0; i < 6; i++) {
        if (i >= u_circleCount) { break; }
        vec3 posRad = u_circlesPosRad[i];
        vec2 cPos = vec2(posRad.r, posRad.g);
        float radius = posRad.b;
        float dist = length(st - cPos);

        //gaussian-like field:
        float sigma = radius * 0.5;
        float val = exp(- (dist * dist) / (2.0 * sigma * sigma));

        //accumulate field and weighted color
        fieldSum += val;
        weightedColorSum += u_circlesColor[i] * val;
    }

    vec3 finalCirclesColor = vec3(0.0);
    if (fieldSum > 0.0) {
      finalCirclesColor = weightedColorSum / fieldSum;
    }

    //to enhance the lava-lamp effect, we can apply a nonlinear curve to fieldSum
    //this makes overlap regions "pop" more:
    float intensity = pow(fieldSum, 1.4); // tweak exponent for effect

    //mix the final circle color with the background based on intensity
    //in light mode, we might add more brightness as intensity grows
    //in dark mode, we can keep it moodier.
    vec3 finalColor;
    if (!u_darkMode) {
        //light mode: as intensity grows, show more of the blob color
        finalColor = mix(bgColor, finalCirclesColor, clamp(intensity, 0.0, 1.0));
        //optionally, you can add more saturation:
        // finalColor = finalColor * (0.5 + 0.5 * intensity); // just an example tweak
    } else {
        //dark mode: blend more subtly (uncomment next line and comment the line after)
        // finalColor = mix(bgColor, finalCirclesColor, clamp(intensity * 0.5, 0.0, 1.0));
        finalColor = mix(bgColor, finalCirclesColor, clamp(intensity, 0.0, 1.0));
    }

    gl_FragColor = vec4(finalColor, 1.0);
}


  `;

  function createShader(type, source) {
    const shader = gl.createShader(type);
    gl.shaderSource(shader, source);
    gl.compileShader(shader);
    if (!gl.getShaderParameter(shader, gl.COMPILE_STATUS)) {
      console.error("Shader compile error:", gl.getShaderInfoLog(shader));
      gl.deleteShader(shader);
      return null;
    }
    return shader;
  }

  const vertShader = createShader(gl.VERTEX_SHADER, vertexSrc);
  const fragShader = createShader(gl.FRAGMENT_SHADER, fragmentSrc);

  const program = gl.createProgram();
  gl.attachShader(program, vertShader);
  gl.attachShader(program, fragShader);
  gl.linkProgram(program);
  if (!gl.getProgramParameter(program, gl.LINK_STATUS)) {
    console.error("Program link error:", gl.getProgramInfoLog(program));
  }

  gl.useProgram(program);

  //fullscreen quad
  const quadBuffer = gl.createBuffer();
  gl.bindBuffer(gl.ARRAY_BUFFER, quadBuffer);
  const vertices = new Float32Array([-1, -1, 1, -1, -1, 1, -1, 1, 1, -1, 1, 1]);
  gl.bufferData(gl.ARRAY_BUFFER, vertices, gl.STATIC_DRAW);

  const a_position = gl.getAttribLocation(program, "a_position");
  gl.enableVertexAttribArray(a_position);
  gl.vertexAttribPointer(a_position, 2, gl.FLOAT, false, 0, 0);

  //uniform locations
  const u_resolution = gl.getUniformLocation(program, "u_resolution");
  const u_darkMode = gl.getUniformLocation(program, "u_darkMode");
  const u_circleCount = gl.getUniformLocation(program, "u_circleCount");
  const u_circlesColor = gl.getUniformLocation(program, "u_circlesColor");
  const u_circlesPosRad = gl.getUniformLocation(program, "u_circlesPosRad");
  const u_mouse = gl.getUniformLocation(program, "u_mouse");

  //set static uniforms
  gl.uniform2f(u_resolution, width, height);

  function updateCircles() {
    for (let i = 0; i < circles.length; i++) {
      const c = circles[i];
      if (!c.interactive) {
        //move circle
        c.x += c.vx;
        c.y += c.vy;
        //wrap around screen
        if (c.x - c.radius > width) c.x = -c.radius;
        if (c.x + c.radius < 0) c.x = width + c.radius;
        if (c.y - c.radius > height) c.y = -c.radius;
        if (c.y + c.radius < 0) c.y = height + c.radius;
      } else {
        //interactive circle follows mouse
        c.x += (mouse.x - c.x) * 0.1;
        c.y += (mouse.y - c.y) * 0.1;
      }
    }
  }

  function render() {
    updateCircles();

    gl.viewport(0, 0, width, height);
    gl.clearColor(0, 0, 0, 1);
    gl.clear(gl.COLOR_BUFFER_BIT);

    gl.useProgram(program);

    gl.uniform1i(u_circleCount, circles.length);
    gl.uniform1i(u_darkMode, $q.dark.isActive ? 1 : 0);
    gl.uniform2f(u_resolution, width, height);
    gl.uniform2f(u_mouse, mouse.x, mouse.y);

    //prepare arrays for uniform data
    //for colors:
    let colorsArr = [];
    //for positions and radius:
    let posRadArr = [];
    for (let i = 0; i < 6; i++) {
      if (i < circles.length) {
        const c = circles[i];
        colorsArr.push(c.color[0], c.color[1], c.color[2]);
        posRadArr.push(c.x, c.y, c.radius);
      } else {
        //dummy values
        colorsArr.push(0, 0, 0);
        posRadArr.push(0, 0, 0);
      }
    }

    gl.uniform3fv(u_circlesColor, new Float32Array(colorsArr));
    gl.uniform3fv(u_circlesPosRad, new Float32Array(posRadArr));

    gl.drawArrays(gl.TRIANGLES, 0, 6);

    requestAnimationFrame(render);
  }

  render();

  onUnmounted(() => {
    window.removeEventListener("resize", resizeCanvas);
    window.removeEventListener("mousemove", onMouseMove);
  });
});
</script>

<style lang="scss">
.background-canvas {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  z-index: -1;
}
</style>
