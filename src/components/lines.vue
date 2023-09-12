<template>
  <canvas class="canvas" ref="myCanvas" />
</template>

<script>
import all_loops from "../assets/loops.json";
import * as THREE from "three";
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls.js";
let meshScale = 1 / 100.0;

let scene, camera, renderer, rayCaster, controls, flow;

scene = new THREE.Scene();
camera = new THREE.PerspectiveCamera(
  15,
  window.innerWidth / window.innerHeight,
  10,
  200
);
camera.position.set(0, -10, 70);
camera.lookAt(scene.position);

export default {
  name: "lines",
  data() {
    return {
      loopsThreeRaw: [],
      loopsThreeRotated: [],
      loopsConverted: [],
      lineMeshes: [],
      mouseXNormalized: 0,
      mouseYNormalized: 0,
    };
  },
  methods: {
    convertToThreeVector(loopRaw) {
      let loop = [];
      for (let j = 0; j < loopRaw.length; j += 15) {
        let z = loopRaw[j][2] * meshScale;
        let y = loopRaw[j][1] * meshScale;
        let x = loopRaw[j][0] * meshScale;
        loop.push(new THREE.Vector3(x, y, z));
      }

      // we need to break it into two parts
      // because we want one part of back to go to left
      // and other part to go to right
      let breakIndex = 0;
      for (let j = 0; j < loop.length - 1; j++) {
        if (loop[j + 1].x * loop[j].x < 0 && loop[j].z < 0) {
          breakIndex = j + 1;
          break;
        }
      }
      let loopRotated = [];
      for (let j = breakIndex; j < loop.length; j++) {
        loopRotated.push(loop[j]);
      }
      for (let j = 0; j < breakIndex; j++) {
        loopRotated.push(loop[j]);
      }
      // make sure the order is the same
      if (loopRotated[0].x > loopRotated[loopRotated.length - 1].x) {
        loopRotated.reverse();
      }
      return loopRotated;
    },
    rotateAroundXY(xRot, yRot) {
      let axis1 = new THREE.Vector3(1, 0, 0);
      let axis2 = new THREE.Vector3(0, 1, 0);
      for (let i = 0; i < this.loopsThreeRaw.length; i++) {
        for (let j = 0; j < this.loopsThreeRaw[i].length; j++) {
          this.loopsThreeRotated[i][j] = this.loopsThreeRaw[i][j]
            .clone()
            .applyAxisAngle(axis1, xRot)
            .applyAxisAngle(axis2, yRot);
        }
      }
    },

    mouseMove(event) {
      // The browser's coordinate system has (0,0) at the top-left
      // and (window.innerWidth, window.innerHeight) at the bottom-right.

      // Normalize the mouse coordinates from -1 to 1
      this.mouseXNormalized = (event.clientX / window.innerWidth) * 2 - 1;
      this.mouseYNormalized = -(event.clientY / window.innerHeight) * 2 + 1; // Note the negative as the screen's y-coordinate goes from top to bottom.
    },

    moveBackLines() {
      for (let i = 0; i < this.loopsThreeRotated.length; i++) {
        for (let j = 0; j < this.loopsThreeRotated[i].length; j++) {
          let z =
            this.loopsThreeRotated[i][j].z < 0
              ? 0
              : this.loopsThreeRotated[i][j].z;
          let x =
            this.loopsThreeRotated[i][j].z >= 0
              ? this.loopsThreeRotated[i][j].x
              : this.loopsThreeRaw[i][j].x > 0
              ? 100018
              : -100018;
          this.loopsConverted[i][j] = new THREE.Vector3(
            x,
            this.loopsThreeRotated[i][j].y,
            z
          );
        }
      }
    },

    preProcessLoops() {
      // we first need to clean the loops
      let newLoops = [];
      let lastHeight = all_loops[0][0][1];
      let currentHeightLargestLength = all_loops[0].length;
      let currentHeightPickedIndex = 0;
      for (let i = 1; i < all_loops.length; i++) {
        if (all_loops[i][0][1] != lastHeight) {
          // first we push the largest loop of the last height
          // and make it threejs vectors
          newLoops.push(
            this.convertToThreeVector(all_loops[currentHeightPickedIndex])
          );
          // this is a new height
          lastHeight = all_loops[i][0][1];
          currentHeightLargestLength = all_loops[i].length;
          currentHeightPickedIndex = i;
        } else {
          // this is the same height
          if (all_loops[i].length > currentHeightLargestLength) {
            currentHeightLargestLength = all_loops[i].length;
            currentHeightPickedIndex = i;
          }
        }
      }
      // push the last one
      newLoops.push(
        this.convertToThreeVector(all_loops[currentHeightPickedIndex])
      );

      for (let i = 0; i < newLoops.length; i++) {
        this.loopsThreeRaw.push(newLoops[i]);
      }
      for (let i = 0; i < newLoops.length; i++) {
        this.loopsConverted.push([]);
        this.loopsThreeRotated.push([]);
        for (let j = 0; j < newLoops[i].length; j++) {
          this.loopsConverted[i].push(newLoops[i][j]);
          this.loopsThreeRotated[i].push(newLoops[i][j]);
        }
      }
    },
    async init() {
      // define the renderer first
      renderer = new THREE.WebGLRenderer({
        canvas: this.$refs.myCanvas,
        antialias: true,
        powerPreference: "high-performance",
      });
      renderer.setPixelRatio(window.devicePixelRatio);
      renderer.setSize(window.innerWidth, window.innerHeight);
      renderer.setClearColor("#000000", 1);

      const material = new THREE.MeshBasicMaterial({ color: 0xffffff });

      // add lines
      for (let i = 0; i < this.loopsConverted.length; i++) {
        const points = this.loopsConverted[i];
        const geometry = new THREE.BufferGeometry().setFromPoints(points);
        const line = new THREE.Line(geometry, material);
        this.lineMeshes.push(geometry);
        scene.add(line);
      }

      window.addEventListener("resize", this.onWindowResize);
      window.addEventListener("mousemove", this.mouseMove);
    },
    animate() {
      let me = this;
      setTimeout(() => {
        me.moveBackLines();
        this.rotateAroundXY(
          (-Math.PI / 6.0) * this.mouseYNormalized - Math.PI / 20.0,
          (Math.PI / 6.0) * this.mouseXNormalized
        );
        me.lineMeshes.forEach((mesh, index) => {
          // mesh.dispose();
          mesh.setFromPoints(me.loopsConverted[index]);
        });
        requestAnimationFrame(me.animate);
      }, 1000 / 90);
      this.render();
    },

    render() {
      renderer.render(scene, camera);
    },
    onWindowResize() {
      camera.aspect = window.innerWidth / window.innerHeight;
      camera.updateProjectionMatrix();
      renderer.setSize(window.innerWidth, window.innerHeight);
    },
  },
  async mounted() {
    this.preProcessLoops();
    this.moveBackLines();
    this.init();
    this.animate();
  },
};
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
canvas {
  width: 100vw;
  height: 100vh;
  background-color: black;
}
</style>
