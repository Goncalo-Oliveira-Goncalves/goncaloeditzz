<script lang="ts">
    import {GLTFLoader, type GLTF} from 'three/addons/loaders/GLTFLoader.js';
    //import {FirstPersonControls } from 'three/addons/controls/FirstPersonControls.js';
    import * as THREE from 'three';
    import { onMount } from 'svelte';

    onMount(() => {
      const KEYS = {
        a: 'KeyA',
        s: 'KeyS',
        w: 'KeyW',
        d: 'KeyD',
      };

      function clamp(x, a, b) {
        return Math.min(Math.max(x, a), b);
      }

      class InputRouterAndActionTaker {
        constructor() {
          this.initialize_();
        }

        initialize_() {
          this.current_ = {
            leftButton: false,
            rightButton: false,
            mouseXDelta: 0,
            mouseYDelta: 0,
            mouseX: 0,
            mouseY: 0
          };
          this.previousMousePosition_ = null;
          this.keys_ = {}; //available keys property
          this.previousKeys_ = {}; //history property?

          document.addEventListener('mousedown', (event) => this.onMouseDown_(event), false);
          document.addEventListener('mouseup', (event) => this.onMouseUp_(event), false);
          document.addEventListener('mousemove', (event) => this.onMouseMove_(event), false);
          document.addEventListener('keydown', (event) => this.onKeyDown_(event), false);
          document.addEventListener('keyup', (event) => this.onKeyUp_(event), false);
        };

        onMouseDown_(event) {
          switch(event.button) {
            case 0: {
              this.current_.leftButton = true;
              break;
            }
            case 1: {
              this.current_.rightButton = true;
              break;
            }
          }
        };
        onMouseUp_(event) {
          switch(event.button) {
            case 0: {
              this.current_.leftButton = false;
              break;
            }
            case 1: {
              this.current_.rightButton = false;
              break;
            }
          }
        };
        onMouseMove_(event) {
          this.current_.mouseX = event.clientX - window.innerWidth/2;
          this.current_.mouseY = event.clientY - window.innerHeight/2;

          if (this.previousMousePosition_ === null) {
            this.previousMousePosition_ = {...this.current_}
          }

          this.current_.mouseXDelta = this.current_.mouseX - this.previousMousePosition_.mouseX;
          this.current_.mouseYDelta = this.current_.mouseY - this.previousMousePosition_.mouseY;
        };

        onKeyDown_(event) {
          this.keys_[event.code] = true;
        };
        onKeyUp_(event) {
          this.keys_[event.code] = false;
        };

        key(keyCode) {
          return !!this.keys_[keyCode];
        }

        update() {
          if (this.previousMousePosition_ !== null) {
            this.current_.mouseXDelta = this.current_.mouseX - this.previousMousePosition_.mouseX;
            this.current_.mouseYDelta = this.current_.mouseY - this.previousMousePosition_.mouseY;

            this.previousMousePosition_ = {...this.current_};
          }
        }
      }

      class FirstPersonCamera {
        constructor(camera) {
          this.camera_ = camera;
          this.input_ = new InputRouterAndActionTaker();
          this.rotation_ = new THREE.Quaternion();
          this.translation_ = new THREE.Vector3(10, 5, 10);
          this.phi_ = 0;
          this.theta_ = 0;
        }

        update(timeElapsedS) {
          this.updateRotation_(timeElapsedS);
          this.updateCamera_(timeElapsedS);
          this.updateTranslation(timeElapsedS);
          this.input_.update(timeElapsedS) // what does this mean...
        }

        updateRotation_(timeElapsedS) { // mouse controls rotation, so we need a seperate function for it.
          const x_axis_rotation = this.input_.current_.mouseXDelta / window.innerWidth;
          const y_axis_rotation = this.input_.current_.mouseYDelta / window.innerHeight;

          this.phi_ += -x_axis_rotation * 5;
          this.theta_ = clamp(this.theta_ + -y_axis_rotation * 5, -Math.PI / 3, Math.PI / 3);

          const quaternion_rotation_x_axis = new THREE.Quaternion();
          quaternion_rotation_x_axis.setFromAxisAngle(new THREE.Vector3(0, 1, 0), this.phi_);
          const quaternion_rotation_y_axis = new THREE.Quaternion();
          quaternion_rotation_y_axis.setFromAxisAngle(new THREE.Vector3(1, 0, 0), this.theta_);

          const quaternion = new THREE.Quaternion();
          quaternion.multiply(quaternion_rotation_x_axis);
          quaternion.multiply(quaternion_rotation_y_axis);

          this.rotation_.copy(quaternion);
        }

        updateTranslation(timeElapsedS) {
          // KEYS.w and KEYS.a as is the W and the A key
          const forwardVelocity = (this.input_.key(KEYS.w) ? 1 : 0) + (this.input_.key(KEYS.s) ? -1 : 0);
          const leftVelocity = (this.input_.key(KEYS.a) ? 1 : 0) + (this.input_.key(KEYS.d) ? -1 : 0);


          const quaternion_rotation_x_axis = new THREE.Quaternion();
          quaternion_rotation_x_axis.setFromAxisAngle(new THREE.Vector3(0, 1, 0), this.phi_);

          const forward = new THREE.Vector3(0, 0, -1);
          forward.applyQuaternion(quaternion_rotation_x_axis);
          forward.multiplyScalar(forwardVelocity * timeElapsedS * 10)

          const left = new THREE.Vector3(-1, 0, 0);
          left.applyQuaternion(quaternion_rotation_x_axis);
          left.multiplyScalar(leftVelocity * timeElapsedS * 10)

          this.translation_.add(forward);
          this.translation_.add(left);
        }

        updateCamera_(_) {
          this.camera_.quaternion.copy(this.rotation_);
          this.camera_.position.copy(this.translation_);
        }
      }

      class ThreeJSScene {
        constructor() {
          this.initialize_();
        }

        initialize_() {
          this.initializeRenderer_();
          this.importSceneIntoScene_();
          this.lightConfiguration_();
          this.controls_();
          this.player_camera_modification_();

          this.previousRAF_ = null;
          this.raf_();

          window.addEventListener('resize', () => {
            this.fpsCamera_.aspect = window.innerWidth / window.innerHeight;

            this.renderer.setSize( window.innerWidth, window.innerHeight );
          });
        }

        initializeRenderer_() {
          this.renderer = new THREE.WebGLRenderer();
          this.renderer.setPixelRatio( window.devicePixelRatio );
          this.renderer.setSize( window.innerWidth, window.innerHeight );
          document.body.appendChild(this.renderer.domElement);

          const fov = 60;
          const aspect = window.innerWidth / window.innerHeight;
          const near = 1.0;
          const far = 1000.0;
          this.camera_ = new THREE.PerspectiveCamera(fov, aspect, near, far);

          const position = [0, 0, 0]
          this.camera_.position.set(position[0], position[1], position[2]);
          this.camera_.lookAt(position[0]+.01, position[1]+.01, position[2]+.01);

          this.scene_ = new THREE.Scene();
        }

        importSceneIntoScene_() {
          const gltfLoader = new GLTFLoader();
          const url = 'maroom.gltf';
          gltfLoader.load(url, (gltf: GLTF) => { // test if it is a gltf or not later, I don't trust ai bro
            const root = gltf.scene;

            // to see the mesh while I have no textures...
            root.traverse((object) => {
              if (object instanceof THREE.Mesh) {
                  object.material = new THREE.MeshNormalMaterial();
              }
            });

            // add 3d scene to scene
            this.scene_.add(root);
          });
        }

        lightConfiguration_() {
          const color = 0xFFFFFF;
          const intensity = 1;
          const light = new THREE.AmbientLight(color, intensity);
          this.scene_.add(light);
        }

        controls_() {
          //skip
        }

        player_camera_modification_() {
          this.fpsCamera_ = new FirstPersonCamera(this.camera_/*, this.objects_ */);
        }

        raf_() {
          requestAnimationFrame((t) => {
            if (this.previousRAF_ === null) {
              this.previousRAF_ = t;
            }

            this.step_(t - this.previousRAF_);
            this.renderer.autoClear = true;
            this.renderer.render(this.scene_, this.camera_);
            this.previousRAF_ = t;
            this.raf_();
          });
        }

        step_(timeElapsed) {
          const timeElapsedS = timeElapsed * 0.001;

          // this.controls_.update(timeElapsedS);
          this.fpsCamera_.update(timeElapsedS);
        }
      }
      new ThreeJSScene();

      // controls


      // raycast
      // down the line I will need raycasting for object interactivityl, but not rn:
      // THREE.Raycaster


    });
</script>
