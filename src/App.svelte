<script lang="ts">
  import { onDestroy, onMount } from "svelte";
  import * as THREE from "three";
  import { OrbitControls } from "three/examples/jsm/controls/OrbitControls";
  import { OBJLoader } from "three/examples/jsm/loaders/OBJLoader";
  import { MTLLoader } from "three/examples/jsm/loaders/MTLLoader";
  import { UnrealBloomPass } from "three/examples/jsm/postprocessing/UnrealBloomPass";
  import { EffectComposer } from "three/examples/jsm/postprocessing/EffectComposer";
  import { RenderPass } from "three/examples/jsm/postprocessing/RenderPass";
  import { ShaderPass } from "three/examples/jsm/postprocessing/ShaderPass";
  import { Mesh, ShaderMaterial } from "three";
  import { Fireworks } from "fireworks-js";

  let canvas: HTMLCanvasElement;
  let fireworksContainer: HTMLCanvasElement;
  let camera: THREE.PerspectiveCamera;
  let renderer: THREE.WebGLRenderer;
  let scene: THREE.Scene;
  let controls: OrbitControls;
  let composer: EffectComposer;
  let finalComposer: EffectComposer;
  let lastDelta = 0;
  const normalMeshes: Array<THREE.Mesh<THREE.BufferGeometry, THREE.MeshPhongMaterial>> = [];
  const bloomMeshes: Array<THREE.Mesh<THREE.BufferGeometry, THREE.MeshPhongMaterial>> = [];
  const numberMeshes: Array<THREE.Mesh<THREE.BufferGeometry, THREE.MeshPhongMaterial>> = [null, null, null, null, null, null, null, null, null, null, null];

  onMount(() => {
    camera = new THREE.PerspectiveCamera(50, window.innerWidth / window.innerHeight, 0.01, 100);

    controls = new OrbitControls(camera, canvas);
    controls.enableDamping = true;
    controls.enablePan = false;

    camera.position.set(10, 5, 5);
    controls.target.set(0, 2, 0);
    //Scene
    scene = new THREE.Scene();

    const directionalLight = new THREE.DirectionalLight(0xffffff, 0);
    const ambientLight = new THREE.AmbientLight(0xffffff, 0.1);

    const geometry = new THREE.BoxGeometry(0.5, 0.5, 0.5);
    const material = new THREE.MeshBasicMaterial({ color: 0x00ff00 });
    const cube = new THREE.Mesh(geometry, material);
    const pointLight1 = new THREE.PointLight(0xffffff, 1, 100);
    const pointLight2 = new THREE.PointLight(0xffffff, 1, 100);
    const pointLight3 = new THREE.PointLight(0xffffff, 1, 100);
    const pointLight4 = new THREE.PointLight(0xffffff, 1, 100);
    const pointLight5 = new THREE.PointLight(0xffffff, 1, 100);
    const pointLight6 = new THREE.PointLight(0xffffff, 1, 100);

    pointLight1.position.set(1.5, 1.1, 1.5);
    pointLight2.position.set(-0.2, 1.1, 1.3);
    pointLight3.position.set(-0.2, 1.1, -1.4);
    pointLight4.position.set(1.5, 1.1, -1.6);
    pointLight5.position.set(-1.6, 2.1, 1.2);
    pointLight6.position.set(-1.6, 2.1, -1.3);
    pointLight1.castShadow = true;
    pointLight4.castShadow = true;

    cube.position.set(1.5, 1.2, 1.5);

    directionalLight.position.x = 5;
    directionalLight.position.y = 5;
    directionalLight.position.z = 7.5;
    directionalLight.castShadow = true;
    directionalLight.shadow.mapSize = new THREE.Vector2(4096, 4096);
    directionalLight.shadow.camera.left = -10;
    directionalLight.shadow.camera.right = 10;
    directionalLight.shadow.camera.top = 10;
    directionalLight.shadow.camera.bottom = -10;
    scene.add(ambientLight);
    scene.add(directionalLight);
    scene.add(pointLight1);
    scene.add(pointLight2);
    scene.add(pointLight3);
    scene.add(pointLight4);
    scene.add(pointLight5);
    scene.add(pointLight6);

    // scene.add(cube);

    for (let i = 0; i <= 12; i++) {
      const mtlLoader = new MTLLoader();
      const objLoader = new OBJLoader();
      mtlLoader.load(`nowruz-display-${i}.mtl`, (materials) => {
        materials.preload();

        objLoader.setMaterials(materials);
        objLoader.load(`nowruz-display-${i}.obj`, (obj) => {
          obj.traverse(function (child) {
            if (child instanceof THREE.Mesh) {
              if (i == 11) child.name = "candle";
              if (i == 12) child.name = "base";
              if (i != 11 && i != 12) {
                if (i == 10) numberMeshes[0] = child;
                if (i == 9) numberMeshes[1] = child;
                if (i == 8) numberMeshes[2] = child;
                if (i == 7) numberMeshes[3] = child;
                if (i == 6) numberMeshes[4] = child;
                if (i == 5) numberMeshes[5] = child;
                if (i == 4) numberMeshes[6] = child;
                if (i == 3) numberMeshes[7] = child;
                if (i == 2) numberMeshes[8] = child;
                if (i == 1) numberMeshes[9] = child;
                if (i == 0) numberMeshes[10] = child;

                numberMeshes.push(child);
                child.name = `n${i}`;
                child.visible = false;
              }
              child.castShadow = true;
              child.receiveShadow = true;
              if (i == 11) bloomMeshes.push(child);
              else normalMeshes.push(child);
            }
          });
          scene.add(obj);
        });
      });
    }

    // Rendering
    renderer = new THREE.WebGLRenderer({ antialias: true, canvas });
    renderer.setSize(window.innerWidth, window.innerHeight);
    renderer.shadowMap.enabled = true;
    renderer.shadowMap.type = THREE.BasicShadowMap;
    renderer.toneMapping = THREE.ReinhardToneMapping;
    renderer.setAnimationLoop(animation);

    const renderScene = new RenderPass(scene, camera);
    const bloomPass = new UnrealBloomPass(new THREE.Vector2(window.innerWidth, window.innerHeight), 0.7, 1, 0);

    composer = new EffectComposer(renderer);
    composer.renderToScreen = false;
    composer.addPass(renderScene);
    composer.addPass(bloomPass);

    const finalPass = new ShaderPass(
      new ShaderMaterial({
        uniforms: {
          baseTexture: { value: null },
          bloomTexture1: { value: composer.renderTarget2.texture },
        },
        vertexShader: `
          varying vec2 vUv;
          void main() {
            vUv = uv;
            gl_Position = projectionMatrix * modelViewMatrix * vec4( position, 1.0 );
          }
          `,
        fragmentShader: `
          uniform sampler2D baseTexture;
          uniform sampler2D bloomTexture1;

          varying vec2 vUv;

          void main() {
            gl_FragColor = texture2D( baseTexture, vUv );
            gl_FragColor += vec4( 1.0 ) * texture2D( bloomTexture1, vUv );
          }
          `,
        defines: {},
      }),
      "baseTexture"
    );
    finalPass.needsSwap = true;

    finalComposer = new EffectComposer(renderer);
    finalComposer.addPass(renderScene);
    finalComposer.addPass(finalPass);

    window.addEventListener("resize", onResize, false);
  });

  function moveToBloom(name: string) {
    const meshIndex = normalMeshes.findIndex((x) => x.name === name);
    if (meshIndex !== -1) {
      bloomMeshes.push(normalMeshes[meshIndex]);
      normalMeshes.splice(meshIndex, 1);
    }
  }

  function moveToNormal(name: string) {
    const meshIndex = bloomMeshes.findIndex((x) => x.name === name);
    if (meshIndex !== -1) {
      normalMeshes.push(bloomMeshes[meshIndex]);
      bloomMeshes.splice(meshIndex, 1);
    }
  }

  function animation(time: number) {
    controls.update();

    var date1 = new Date();
    var date2 = new Date("03/20/2023 22:24:28");
    // var date2 = new Date("03/20/2023 18:14:00");
    var delta = Math.floor(Math.abs(date2.getTime() - date1.getTime()) / 1000);

    if (delta < lastDelta) {
      var days = Math.floor(delta / 86400);
      delta -= days * 86400;

      var hours = Math.floor(delta / 3600) % 24;
      delta -= hours * 3600;

      var minutes = Math.floor(delta / 60) % 60;
      delta -= minutes * 60;

      var seconds = delta % 60;

      if (days == 0 && hours == 0 && minutes == 0 && seconds == 0) {
        canvas.style.display = "none";
        fireworksContainer.style.display = "block";
        onResize();
        const fireworks = new Fireworks(fireworksContainer);
        fireworks.start();
      } else {
        addNumberSet("second", seconds, -2, -2.5, false);
        addNumberSet("minute", minutes, -0.5, -1, true);
        addNumberSet("hour", hours, 1, 0.5, true);
        addNumberSet("day", days, 2.5, 2, true);
      }
      // console.log(secondLMesh);
    }

    lastDelta = delta;

    // function

    bloomMeshes.forEach((x) => {
      if (x.material) x.material.color.set(0xffffff);
    });
    scene.background = new THREE.Color("#000000");
    normalMeshes.forEach((x) => {
      x.material.color.set(0x000000);
    });
    composer.render();

    scene.background = new THREE.Color("#1e88a8");
    normalMeshes.forEach((x) => {
      x.material.color.set(0xffffff);
    });
    finalComposer.render();
    // renderer.render(scene, camera);
  }

  function addNumberSet(name: string, date: number, lPosition: number, rPosition: number, addColon: boolean) {
    var dateStr1 = Number.isNaN(parseInt(date.toString().charAt(0))) ? -1 : parseInt(date.toString().charAt(0));
    var dateStr2 = Number.isNaN(parseInt(date.toString().charAt(1))) ? -1 : parseInt(date.toString().charAt(1));

    var lMesh = scene.getObjectByName("l" + name);
    var rMesh = scene.getObjectByName("r" + name);

    if (lMesh) scene.remove(lMesh);
    if (rMesh) scene.remove(rMesh);

    // if (dateStr1 == -1 && dateStr2 == -1) return;

    lMesh = numberMeshes[dateStr2 ? dateStr1 : 0].clone();
    lMesh.name = "l" + name;
    lMesh.position.set(-1.8, 2.8, lPosition);
    lMesh.castShadow = false;
    lMesh.visible = true;
    scene.add(lMesh);

    console.log(dateStr1 + " " + dateStr2 + " " + date + " " + name);
    rMesh = numberMeshes[dateStr2 != -1 && dateStr1 != -1 ? dateStr2 : dateStr1 != -1 ? dateStr1 : 0].clone();
    rMesh.name = "r" + name;
    rMesh.position.set(-1.8, 2.8, rPosition);
    rMesh.castShadow = false;
    rMesh.visible = true;
    scene.add(rMesh);

    if (!addColon) return;
    var colonMesh = scene.getObjectByName("c" + name);
    if (colonMesh) scene.remove(colonMesh);

    colonMesh = numberMeshes[10].clone();
    colonMesh.name = "c" + name;
    colonMesh.position.set(-1.8, 2.8, rPosition - 0.55);
    colonMesh.castShadow = false;
    colonMesh.visible = true;
    scene.add(colonMesh);
  }

  onDestroy(() => {
    window.removeEventListener("resize", onResize);

    bloomMeshes.splice(0, bloomMeshes.length);
    normalMeshes.splice(0, normalMeshes.length);
  });

  function onResize() {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();

    renderer.setSize(window.innerWidth, window.innerHeight);
    fireworksContainer.width = window.innerWidth;
    fireworksContainer.height = window.innerHeight;
  }
</script>

<main>
  <canvas bind:this={canvas} />
  <canvas style="display: none;" bind:this={fireworksContainer} />
</main>

<style>
</style>
