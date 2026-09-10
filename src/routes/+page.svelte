<script lang="ts">
    import {GLTFLoader, type GLTF} from 'three/addons/loaders/GLTFLoader.js';
    import * as THREE from 'three';
    import { onMount } from 'svelte';
    import { PointerLockControls } from 'three/addons/controls/PointerLockControls.js';

    const SPAWN = [1.5, 1, 2]

    onMount(() => {
      const KEYS: Record<string, string> = {
        a: 'KeyA',
        s: 'KeyS',
        w: 'KeyW',
        d: 'KeyD',
      };

      class InputRouterAndActionTaker {
        // define type of properties
        // keys_: Record<string, boolean> = {};

        constructor() {
          this.initialize_();
        }

        initialize_() {
          this.keys_ = {}; //available keys property
          this.previousKeys_ = {}; //history property?

          document.addEventListener('keydown', (event: KeyboardEvent) => this.onKeyDown_(event), false);
          document.addEventListener('keyup', (event: KeyboardEvent) => this.onKeyUp_(event), false);
        };

        onKeyDown_(event: KeyboardEvent) {
          this.keys_[event.code] = true;
        };
        onKeyUp_(event: KeyboardEvent) {
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
        constructor(camera: THREE.PerspectiveCamera) {
          // we cannot define it in other places, because most of this shit is async.

          this.suggestedTranslation = {
            forward: new THREE.Vector3(),
            left: new THREE.Vector3()
          };
          this.camera_ = camera;
          this.controls_ = new PointerLockControls(camera, document.body)
          document.body.addEventListener('click', () => {
            this.controls_.lock();
          });
          this.input_ = new InputRouterAndActionTaker();

          this.translation_ = new THREE.Vector3(SPAWN[0], SPAWN[1], SPAWN[2]);
        }

        update(timeElapsedS) {
          this.updateTranslation(timeElapsedS);
        }

        updateTranslation(timeElapsedS) {
          // KEYS.w and KEYS.a as is the W and the A key
          const forwardVelocity = (this.input_.key(KEYS.w) ? 1 : 0) + (this.input_.key(KEYS.s) ? -1 : 0);
          const leftVelocity = (this.input_.key(KEYS.a) ? 1 : 0) + (this.input_.key(KEYS.d) ? -1 : 0);

          const forward = new THREE.Vector3();
          this.camera_.getWorldDirection(forward);

          forward.y = 0;
          forward.normalize();
          forward.multiplyScalar(forwardVelocity * timeElapsedS * 10)

          const left = new THREE.Vector3();
          left.crossVectors(forward, this.camera_.up).normalize();
          left.multiplyScalar(leftVelocity * timeElapsedS * 10);

          // add suggested translation here
          this.suggestedTranslation = {
            forward: forward,
            left: left
          };
        }
      }

      class ThreeJSScene {
        constructor() {
          this.initialize_();
        }

        initialize_() {
          console.log('[INIT] ThreeJSScene.initialize_');
          this.hitboxes = {};
          this.initializeRenderer_();
          this.importSceneIntoScene_();
          this.lightConfiguration_();
          this.controls_();
          this.player_camera_modification_();

          this.previousRAF_ = null;
          this.raf_(); // render animation frame

          window.addEventListener('resize', () => {
            this.camera_.aspect =
              window.innerWidth / window.innerHeight;

            this.camera_.updateProjectionMatrix();

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
          console.log('[RENDERER]', this.renderer, this.camera_, this.scene_);
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
            console.log('[RAF] frame', t);

            const suggestedTranslation = this.fpsCamera_.suggestedTranslation;

            const forward = collisionHandler(this.hitboxes, this.scene_, this.playerBox, suggestedTranslation.forward);
            const left = collisionHandler(this.hitboxes, this.scene_, this.playerBox, suggestedTranslation.left);

            // inside a statement on "if collision true" this.step_(t - this.previousRAF_);

            this.fpsCamera_.translation_.add(forward);
            this.fpsCamera_.translation_.add(left);
            this.camera_.position.copy(
              this.fpsCamera_.translation_
            );

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

          console.log('[CAMERA]', this.camera_.position, this.camera_.rotation);

          this.playerBox.setFromCenterAndSize(
            this.fpsCamera_.translation_,
            new THREE.Vector3(.5, 2, .5)
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
