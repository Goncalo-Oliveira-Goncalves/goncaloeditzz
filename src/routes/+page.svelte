<script lang="ts">
    import {GLTFLoader, type GLTF} from 'three/addons/loaders/GLTFLoader.js';
    import { FirstPersonControls } from 'three/addons/controls/FirstPersonControls.js';
    import * as THREE from 'three';
    import { onMount } from 'svelte';

    onMount(() => {
      const scene = new THREE.Scene();

      const renderer = new THREE.WebGLRenderer();
      renderer.setPixelRatio( window.devicePixelRatio );
      renderer.setSize( window.innerWidth, window.innerHeight );
      document.body.appendChild(  renderer.domElement );

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
        scene.add(root);
      });

      // lights
      const color = 0xFFFFFF;
      const intensity = 1;
      const light = new THREE.AmbientLight(color, intensity);
      scene.add(light);

      // camera
      var camera = new THREE.PerspectiveCamera( 75, window.innerWidth / window.innerHeight, 0.1, 1000 );

      /// controls
      THREE.FirstPersonControls = function ( camera, MouseMoveSensitivity = 0.002, speed = 800.0, jumpHeight = 350.0, height = 30.0) {
        var scope = this;

        scope.MouseMoveSensitivity = MouseMoveSensitivity;
        scope.speed = speed;
        scope.height = height;
        scope.jumpHeight = scope.height + jumpHeight;
        scope.click = false;

        var moveForward = false;
        var moveBackward = false;
        var moveLeft = false;
        var moveRight = false;
        var canJump = false;
        var run = false;

        var velocity = new THREE.Vector3();
        var direction = new THREE.Vector3();

        var prevTime = performance.now();

        camera.rotation.set( 0, 0, 0 );

        var pitchObject = new THREE.Object3D();
        pitchObject.add( camera );

        var yawObject = new THREE.Object3D();
        yawObject.position.y = 10;
        yawObject.add( pitchObject );

        var PI_2 = Math.PI / 2;

        var onMouseMove = function ( event ) {

          if ( scope.enabled === false ) return;

          var movementX = event.movementX || event.mozMovementX || event.webkitMovementX || 0;
          var movementY = event.movementY || event.mozMovementY || event.webkitMovementY || 0;

          yawObject.rotation.y -= movementX * scope.MouseMoveSensitivity;
          pitchObject.rotation.x -= movementY * scope.MouseMoveSensitivity;

          pitchObject.rotation.x = Math.max( - PI_2, Math.min( PI_2, pitchObject.rotation.x ) );

        };

        var onKeyDown = (function ( event ) {

          if ( scope.enabled === false ) return;

          switch ( event.keyCode ) {
            case 38: // up
            case 87: // w
              moveForward = true;
              break;

            case 37: // left
            case 65: // a
              moveLeft = true;
              break;

            case 40: // down
            case 83: // s
              moveBackward = true;
              break;

            case 39: // right
            case 68: // d
              moveRight = true;
              break;

            case 32: // space
              if ( canJump === true ) velocity.y += run === false ? scope.jumpHeight : scope.jumpHeight + 50;
              canJump = false;
              break;

            case 16: // shift
              run = true;
              break;

          }

        }).bind(this);

        var onKeyUp = (function ( event ) {

          if ( scope.enabled === false ) return;

          switch ( event.keyCode ) {

            case 38: // up
            case 87: // w
              moveForward = false;
              break;

            case 37: // left
            case 65: // a
              moveLeft = false;
              break;

            case 40: // down
            case 83: // s
              moveBackward = false;
              break;

            case 39: // right
            case 68: // d
              moveRight = false;
              break;

            case 16: // shift
              run = false;
              break;

          }

        }).bind(this);

        var onMouseDownClick= (function ( event ) {
          if ( scope.enabled === false ) return;
          scope.click = true;
        }).bind(this);

        var onMouseUpClick= (function ( event ) {
          if ( scope.enabled === false ) return;
          scope.click = false;
        }).bind(this);

        scope.dispose = function() {
          document.removeEventListener( 'mousemove', onMouseMove, false );
          document.removeEventListener( 'keydown', onKeyDown, false );
          document.removeEventListener( 'keyup', onKeyUp, false );
          document.removeEventListener( 'mousedown', onMouseDownClick, false );
          document.removeEventListener( 'mouseup', onMouseUpClick, false );
        };

        document.addEventListener( 'mousemove', onMouseMove, false );
        document.addEventListener( 'keydown', onKeyDown, false );
        document.addEventListener( 'keyup', onKeyUp, false );
        document.addEventListener( 'mousedown', onMouseDownClick, false );
        document.addEventListener( 'mouseup', onMouseUpClick, false );

        scope.enabled = false;

        scope.getObject = function () {

          return yawObject;

        };

        scope.update = function () {

          var time = performance.now();
          var delta = ( time - prevTime ) / 1000;

          velocity.y -= 9.8 * 100.0 * delta;
          velocity.x -= velocity.x * 10.0 * delta;
          velocity.z -= velocity.z * 10.0 * delta;

          direction.z = Number( moveForward ) - Number( moveBackward );
          direction.x = Number( moveRight ) - Number( moveLeft );
          direction.normalize();

          var currentSpeed = scope.speed;
          if (run && (moveForward || moveBackward || moveLeft || moveRight)) currentSpeed = currentSpeed + (currentSpeed * 1.1);

          if ( moveForward || moveBackward ) velocity.z -= direction.z * currentSpeed * delta;
          if ( moveLeft || moveRight ) velocity.x -= direction.x * currentSpeed * delta;

          scope.getObject().translateX( -velocity.x * delta );
          scope.getObject().translateZ( velocity.z * delta );

          scope.getObject().position.y += ( velocity.y * delta );

          if ( scope.getObject().position.y < scope.height ) {

            velocity.y = 0;
            scope.getObject().position.y = scope.height;

            canJump = true;
          }
          prevTime = time;
        };
      };
      controls = new THREE.FirstPersonControls( camera );
      controls.update();

      // raycast
      // down the line I will need raycasting for object interactivityl, but not rn:
      // THREE.Raycaster


      window.addEventListener('resize', () => {
        camera.aspect = window.innerWidth / window.innerHeight;
        camera.updateProjectionMatrix();

        renderer.setSize( window.innerWidth, window.innerHeight );
      });

      // action
      function animate(time: number) {
        console.log(time);

        controls.update();
        renderer.render( scene, camera );
      }
      renderer.setAnimationLoop( animate );
    });
</script>
