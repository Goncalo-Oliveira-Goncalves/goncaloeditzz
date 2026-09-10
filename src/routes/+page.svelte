<script lang="ts">
    import {GLTFLoader, type GLTF} from 'three/addons/loaders/GLTFLoader.js';
    import * as THREE from 'three';
    import { onMount } from 'svelte';
    import { PointerLockControls } from 'three/addons/controls/PointerLockControls.js';

    const SPAWN = [1.5, 1, 2]

    // TODO: HITBOXES: https://threejs.org/docs/#Box3.setFromObject (you will have to break the scene down into individual objects)

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
          // we cannot define it in other places, because most of this shit is async.

          this.suggestedTranslation = {
            forward: new THREE.Vector3(),
            left: new THREE.Vector3()
          };
          this.camera_ = camera;
          this.controls_ = new PointerLockControls(camera, document.body)
          this.input_ = new InputRouterAndActionTaker();
          this.rotation_ = new THREE.Quaternion();
          this.translation_ = new THREE.Vector3(SPAWN[0], SPAWN[1], SPAWN[2]);
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

          // add suggested translation here
          this.suggestedTranslation = {
            forward: forward,
            left: left
          };
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
          this.hitboxes = {};
          this.initializeRenderer_();
          this.importSceneIntoScene_();
          this.lightConfiguration_();
          this.controls_();
          this.player_camera_modification_();

          this.previousRAF_ = null;
          this.raf_(); // render animation frame

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
          const near = 0.01;
          const far = 100.0;
          this.camera_ = new THREE.PerspectiveCamera(fov, aspect, near, far);

          this.camera_.position.set(
            SPAWN[0],
            SPAWN[1],
            SPAWN[2]
          );

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
                const boundingbox = new THREE.Box3(new THREE.Vector3(), new THREE.Vector3());
                this.hitboxes[object.name] = boundingbox.setFromObject(object);
                console.log(object.name)

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

          this.playerBox = new THREE.Box3(
            new THREE.Vector3(),
            new THREE.Vector3()
          )
        }

        raf_() {
          function getCollisionNormal(playerBox, boundingBox) {
            const boxNormals = [
              new THREE.Vector3(1, 0, 0),   // +X
              new THREE.Vector3(-1, 0, 0),  // -X
              new THREE.Vector3(0, 1, 0),   // +Y
              new THREE.Vector3(0, -1, 0),  // -Y
              new THREE.Vector3(0, 0, 1),   // +Z
              new THREE.Vector3(0, 0, -1),  // -Z
            ];

            const overlapX = Math.min(
              playerBox.max.x - boundingBox.min.x,
              boundingBox.max.x - playerBox.min.x
            );

            const overlapY = Math.min(
              playerBox.max.y - boundingBox.min.y,
              boundingBox.max.y - playerBox.min.y
            );

            const overlapZ = Math.min(
              playerBox.max.z - boundingBox.min.z,
              boundingBox.max.z - playerBox.min.z
            );

            // Find the axis with the smallest overlap
            if (overlapX < overlapY && overlapX < overlapZ) {
              // Collision happened on X axis
              if (playerBox.getCenter(new THREE.Vector3()).x <
                  boundingBox.getCenter(new THREE.Vector3()).x) {
                return boxNormals[0]; // +X
              } else {
                return boxNormals[1]; // -X
              }
            }

            if (overlapY < overlapZ) {
              // Collision happened on Y axis
              if (playerBox.getCenter(new THREE.Vector3()).y <
                  boundingBox.getCenter(new THREE.Vector3()).y) {
                return boxNormals[2]; // +Y
              } else {
                return boxNormals[3]; // -Y
              }
            }

            // Collision happened on Z axis
            if (playerBox.getCenter(new THREE.Vector3()).z <
                boundingBox.getCenter(new THREE.Vector3()).z) {
              return boxNormals[4]; // +Z
            } else {
              return boxNormals[5]; // -Z
            }
          }

          function collisionHandler(
            hitboxes,
            scene,
            playerBox,
            suggestedTranslation
          ) {
            // If the wall normal is:
            // `normal = (1, 0, 0)`
            // and you're trying to move:
            // `velocity = (1, 0, 0)`
            // then:
            // Do not allow movement.
            // And if:
            // `velocity = (1, 0, 1)`
            // Allow.

            let modified_translation = suggestedTranslation;

            const futurePlayerBox = playerBox.clone();
            futurePlayerBox.translate(modified_translation);

            for (const [objectName, boundingBox] of Object.entries(hitboxes)) {
              console.log(scene.getObjectByName(objectName))
              if (futurePlayerBox.intersectsBox(boundingBox)) {
                // TODO: CONTINUE HANDING DOT MULT BASED COLLISSION & TRANSLATION
                // TODO: ADD UPDATE ON FPSCAMERA IF CERTAIN MOVEMENT IS UNALLOWED.
                const normal = getCollisionNormal(futurePlayerBox, boundingBox);

                const wallForce = modified_translation.dot(normal);
                const movementToWall = normal.clone().multiplyScalar(wallForce);
                modified_translation = modified_translation.sub(movementToWall);
              }
            }

            return modified_translation
          }

          requestAnimationFrame((t) => {
            if (this.previousRAF_ === null) {
              this.previousRAF_ = t;
            }

            this.step_(t - this.previousRAF_);

            const suggestedTranslation = this.fpsCamera_.suggestedTranslation;

            const forward = collisionHandler(this.hitboxes, this.scene_, this.playerBox, suggestedTranslation.forward);
            const left = collisionHandler(this.hitboxes, this.scene_, this.playerBox, suggestedTranslation.left);

            // inside a statement on "if collision true" this.step_(t - this.previousRAF_);

            this.fpsCamera_.translation_.add(forward);
            this.fpsCamera_.translation_.add(left);
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
          this.playerBox.setFromCenterAndSize(
            this.fpsCamera_.translation_,
            new THREE.Vector3(0.2, .1, .1)
          );
        }
      }
      new ThreeJSScene();

      // controls


      // raycast
      // down the line I will need raycasting for object interactivityl, but not rn:
      // THREE.Raycaster


    });
</script>
