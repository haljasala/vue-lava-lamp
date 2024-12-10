<template>
  <canvas ref="canvas" class="background-canvas"></canvas>
</template>
<script setup>
import { ref, onMounted, onUnmounted } from "vue";
import { useQuasar } from "quasar";

const $q = useQuasar();

const canvas = ref(null);

//morphing gradient background canvas
onMounted(() => {
  const ctx = canvas.value.getContext("2d");
  let width = (canvas.value.width = window.innerWidth);
  let height = (canvas.value.height = window.innerHeight);

  //offscreen canvas
  const offscreenCanvas = document.createElement("canvas");
  const offCtx = offscreenCanvas.getContext("2d");
  offscreenCanvas.width = width;
  offscreenCanvas.height = height;

  function resizeCanvas() {
    width = canvas.value.width = window.innerWidth;
    height = canvas.value.height = window.innerHeight;
    offscreenCanvas.width = width;
    offscreenCanvas.height = height;
    init(); //re-initialize circles on resize
  }

  window.addEventListener("resize", resizeCanvas);

  let circles = [];
  let animationFrameId;

  //mouse position
  let mouse = {
    x: width / 2,
    y: height / 2,
  };

  function onMouseMove(event) {
    mouse.x = event.clientX;
    mouse.y = event.clientY;
  }

  window.addEventListener("mousemove", onMouseMove);

  class Circle {
    constructor(x, y, radius, color, velocity, isInteractive = false) {
      this.x = x;
      this.y = y;
      this.radius = radius;
      this.color = color;
      this.velocity = velocity;
      this.isInteractive = isInteractive;
    }

    update() {
      this.x += this.velocity.x;
      this.y += this.velocity.y;

      //boundary detection
      if (this.x - this.radius > width) {
        this.x = -this.radius;
      }
      if (this.x + this.radius < 0) {
        this.x = width + this.radius;
      }
      if (this.y - this.radius > height) {
        this.y = -this.radius;
      }
      if (this.y + this.radius < 0) {
        this.y = height + this.radius;
      }
    }

    draw(context) {
      const gradient = context.createRadialGradient(
        this.x,
        this.y,
        0,
        this.x,
        this.y,
        this.radius,
      );
      gradient.addColorStop(0, `rgba(${this.color}, 1)`);
      gradient.addColorStop(1, `rgba(${this.color}, 0)`);

      context.fillStyle = gradient;
      context.beginPath();
      context.arc(this.x, this.y, this.radius, 0, Math.PI * 2, false);
      context.fill();
    }
  }

  function init() {
    circles = [];

    const colors = [
      "18, 113, 255", // color1
      "221, 74, 255", // color2
      "100, 220, 255", // color3
      "200, 50, 50", // color4
      "180, 180, 50", // color5
      "140, 100, 255", // interactive color
    ];

    //create circles
    for (let i = 0; i < 5; i++) {
      const radius = (width + height) * 0.2;
      const x = Math.random() * width;
      const y = Math.random() * height;
      const speedMultiplier = Math.random() * 4 + 1;
      const velocity = {
        x: (Math.random() - 0.5) * speedMultiplier,
        y: (Math.random() - 0.5) * speedMultiplier,
      };

      circles.push(new Circle(x, y, radius, colors[i], velocity));
    }

    //interactive circle
    const interactiveCircle = new Circle(
      width / 2,
      height / 2,
      (width + height) * 0.1,
      colors[5],
      { x: 0, y: 0 },
      true,
    );
    circles.push(interactiveCircle);
  }

  function animate() {
    //clear the main canvas
    ctx.clearRect(0, 0, width, height);

    //draw background gradient on the main canvas
    const bgGradient = ctx.createLinearGradient(0, 0, width, height);
    bgGradient.addColorStop(0, "rgb(108, 0, 162)");
    bgGradient.addColorStop(1, "rgb(0, 17, 82)");
    ctx.fillStyle = bgGradient;
    ctx.fillRect(0, 0, width, height);

    //clear the offscreen canvas
    offCtx.clearRect(0, 0, width, height);

    //set blur filter and composite operation on offscreen canvas
    //use blurrier for lightmode and less for darkmode
    const blurMode = !$q.dark.isActive ? "blur(80px)" : "blur(40px)";
    offCtx.filter = blurMode;
    offCtx.globalCompositeOperation = "source-over";

    //draw and update circles on the offscreen canvas
    circles.forEach((circle) => {
      if (circle.isInteractive) {
        circle.x += (mouse.x - circle.x) * 0.1;
        circle.y += (mouse.y - circle.y) * 0.1;
      } else {
        circle.update();
      }
      circle.draw(offCtx);
    });

    //use lighter for lightmode and darken for darkmode
    const lightMode = !$q.dark.isActive ? "lighter" : "hue";

    //draw the offscreen canvas onto the main canvas
    ctx.globalCompositeOperation = lightMode;
    ctx.drawImage(offscreenCanvas, 0, 0);

    //reset composite operation on main canvas
    ctx.globalCompositeOperation = "source-over";

    animationFrameId = requestAnimationFrame(animate);
  }

  init();
  animate();

  onUnmounted(() => {
    window.removeEventListener("resize", resizeCanvas);
    window.removeEventListener("mousemove", onMouseMove);
    cancelAnimationFrame(animationFrameId);
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
