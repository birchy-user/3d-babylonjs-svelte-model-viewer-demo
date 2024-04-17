<script>
    import { onMount } from 'svelte';

    import { 
        Animation,
        Engine,
        Scene,
        ArcRotateCamera,
        UniversalCamera,
        FreeCamera,
        Vector3,
        HemisphericLight,
        Mesh,
        MeshBuilder, 
        SceneLoader,
        Texture,
        PointerEventTypes,
        StandardMaterial,
        CubeTexture,
        Color3,
        ShadowGenerator,
        Vector2,
        AssetsManager
    } from "@babylonjs/core";
    
    // Automātiski reģistrē visus nepieciešamos failu lejupielādes moduļus (tai skaitā glTF un GLB)
    import '@babylonjs/loaders';

    // Oficiālais Babylon ūdens virsmas materiāls
    import { SkyMaterial, WaterMaterial } from '@babylonjs/materials';
    import { waterVertexShader } from '@babylonjs/materials/water/water.vertex';
  
    let canvas;
  
    onMount(() => {
        const engine = new Engine(canvas, true);
        const scene = new Scene(engine);

        // Definē rotācijas kameras objektu ar šādiem parametriem:
        //  *) Objekta nosaukums
        //  *) sākotnējā rotācija ap meridiānu (longitude) jeb horizontāli (radiānos)
        //  *) sākotnējā rotācija ap paralēli (latitude) jeb vertikāli (radiānos)
        //  *) sākotnējais kameras attālums no tās fokusa objekta ("target")
        //  *) 3D vektors (X, Y, Z), kas atbilst kameras fokusa objektam ("target")
        //  *) galvenās 3D ainas objekts
        // const camera = new ArcRotateCamera("Camera", 3 * Math.PI / 2, Math.PI / 4, 100, Vector3.Zero(), scene);
        const camera = new ArcRotateCamera("Camera", 3 * Math.PI / 2, 5 * Math.PI / 12, 20, Vector3.Zero(), scene);  // deg(5 * Math.PI / 12) = 75°
        camera.lowerRadiusLimit = 10;
        camera.upperRadiusLimit = 500;
        camera.useBouncingBehavior = true;
        // const camera = new FreeCamera("camera", new Vector3(0, 5, -10), scene);
        // const camera = new UniversalCamera("Camera", new Vector3(0, 0, -10), scene)
        // camera.setTarget(Vector3.Zero())
        camera.attachControl(canvas, true);
    
        const light = new HemisphericLight("light", new Vector3(0, 1, 0), scene)
        // light.intensity = 0.7;

        // const shadowGenerator = new ShadowGenerator(1024, light);

        let rotating = false;
        const rightDir = new Vector3();
        const upDir = new Vector3();
        const sensitivity = 0.005;

        let football;
        let footballMeshes = [];
      
        // failu atrašanās vietai jābūt `public` mapē, lai Babylon loader to uzreiz pamanītu:
        // SceneLoader.Append(
        //     '/assets/models/',
        //     'soccer_ball.glb',
        //     scene,
        //     (scene) => { 
        //         // `__root__` (scene.meshes[0]) ir pats galvenais objekts pašā 3D modelī:
        //         football = scene.getMeshByName("__root__");
        //         football.position.y = 15;
        //         football.isPickable = true;

        //         footballMeshes = [...scene.meshes];
        //         footballMeshes.forEach(mesh => mesh.isPickable = true);

        //         if (!football) {
        //             console.error("Galvenais futbola bumbas modelis nav atrasts");
        //             return;
        //         }

        //         console.log("football loaded: ", football);
        //         console.log("footballMeshes: ", footballMeshes);

        //         // // Vertical Oscillation Animation
        //         // let yAnimation = new Animation("yAnimation", "position.y", 20, Animation.ANIMATIONTYPE_FLOAT, Animation.ANIMATIONLOOPMODE_CYCLE);
        //         // let keyFrames = [];
        //         // keyFrames.push({ frame: 0, value: football.position.y });
        //         // keyFrames.push({ frame: 50, value: football.position.y + 2 }); // + 2 norāda izmaiņu par 2 px uz augšu (vertikāli) animācijas vidū
        //         // keyFrames.push({ frame: 100, value: football.position.y });
        //         // yAnimation.setKeys(keyFrames);
        //         // football.animations = [yAnimation];

        //         // // Curved Rotation Animation
        //         // let rotationAnimation = new Animation("rotationAnimation", "rotation.x", 20, Animation.ANIMATIONTYPE_FLOAT, Animation.ANIMATIONLOOPMODE_CYCLE);
        //         // let rotationFrames = [];
        //         // rotationFrames.push({ frame: 0, value: 0 });
        //         // rotationFrames.push({ frame: 50, value: Math.PI });  // Animācijas vidū tiek veikts pilns 180 grādu apgrieziens pa x asi
        //         // rotationFrames.push({ frame: 100, value: 0 });  // Return to start
        //         // rotationAnimation.setKeys(rotationFrames);
                
        //         // football.animations.push(rotationAnimation);

        //         // // Begin Animations
        //         // scene.beginAnimation(football, 0, 100, true);
        //     },
        //     undefined,
        //     (scene, message, exception) => {
        //         // Kļūdu apstrādes metode
        //         console.error("Error loading model: ", scene, message, exception);
        //     },
        //     '.glb'
        // );

        let footballGoal;

        const assetsManager = new AssetsManager(scene);
        const meshTask = assetsManager.addMeshTask("football goal task", "", "/assets/models/", "football_goal.glb", ".glb");
        meshTask.onSuccess = (task) => {
            footballGoal = task.loadedMeshes[0];
            footballGoal.scaling = new Vector3(5, 5, 5);
            footballGoal.position.y = 1;
            footballGoal.position.z = 20;
            console.log("success: ", footballGoal);
        };
        meshTask.onError = (task, message, exception) => {
            console.error(task, message, exception);
        };

        const meshTaskTwo = assetsManager.addMeshTask("soccer ball task", "", "/assets/models/", "soccer_ball.glb", ".glb");
        meshTaskTwo.onSuccess = (task) => {
            football = task.loadedMeshes[0];
            football.position.y = 1;
            console.log("success: ", footballGoal);
        };
        meshTaskTwo.onError = (task, message, exception) => {
            console.error(task, message, exception);
        };

        // Move the light with the camera
        // scene.registerBeforeRender(function () {
        //     light.position = camera.position;
        // });
        
        assetsManager.onFinish = (tasks) => {
            // Uzstāda debesis tā, lai tās atbilstu gaišam dienas laikam (kad saule atrodas tieši virs galvas), mainot tās materiāla (šajā gadījumā - saules) slīpumu (inclination)
            animateSky("material.inclination", skyboxMaterial.inclination, 0);
            engine.runRenderLoop(() => {
                scene.render();
            });
        };
        
        assetsManager.load();

        // Debesis kā kaste, kura iekļauj pārējos modeļus (tās materiāls - saule):
        const skyboxMaterial = new SkyMaterial("skyBox", scene);
        skyboxMaterial.backFaceCulling = false;
        // `CubeTexture` ļauj norādīt 6 bildes ar vienādu nosaukumu formātā `NOSAUKUMS_ab`, 
        // kur a un b var būt viena no šādām vērtībām: px, py, pz (pirmās trīs bildes), nx, ny, nz (nākošās trīs bildes)
        //  kur a - pirmās 3 bildes (x, y, z asu pozitīvajā virzienā), b - nākošās 3 bildes (x, y, z asu negatīvajā virzienā)
        // Šajā gadījumā tiks meklētas 6 bildes ar nosaukumu `TropicalSunnyDay_ab`
        // skyboxMaterial.reflectionTexture = new CubeTexture("assets/textures/TropicalSunnyDay", scene);
        // skyboxMaterial.reflectionTexture.coordinatesMode = Texture.SKYBOX_MODE;
        // skyboxMaterial.diffuseColor = new Color3(0, 0, 0);
        // skyboxMaterial.specularColor = new Color3(0, 0, 0);
        // skyboxMaterial.disableLighting = true;
        skyboxMaterial.fogEnabled = true;

        const skybox = MeshBuilder.CreateBox("skyBox", {size: 4000, width: 4000, height: 4000}, scene);
        skybox.material = skyboxMaterial;

        // Definē animāciju, kas izpildās uz noteiktu objektu 5 sekunžu laikā (parametrs 0.2, kas atbilst 1/5 no 1 sekundes) no sākuma punkta (from) uz beigu punktu (to):
        const animateSky = (propertyName, from, to) => {
            var keys = [
                { frame: 0, value: from },
                { frame: 100, value: to }
            ];
            
            var animation = new Animation("animation", propertyName, 100, Animation.ANIMATIONTYPE_FLOAT, Animation.ANIMATIONLOOPMODE_CONSTANT);
            animation.setKeys(keys);
            
            scene.stopAnimation(skybox);
            scene.beginDirectAnimation(skybox, [animation], 0, 100, false, 0.2);
        };

        // Zemes virsma (virs kuras attēlos ūdens virsmu):
        const groundMaterial = new StandardMaterial("groundMaterial", scene);
	    groundMaterial.diffuseTexture = new Texture("assets/textures/ground.jpg", scene);
        groundMaterial.diffuseTexture.uScale = groundMaterial.diffuseTexture.vScale = 6;
        groundMaterial.specularColor = new Color3(0, 0, 0);
        
        const ground = MeshBuilder.CreateGround("ground", { width: 2048, height: 2048, subdivisions: 32 }, scene);
        ground.position.y = -3;
        ground.material = groundMaterial;

        // Ūdens virsmas izveide un pievienošana ainai:
        const waterMaterial = MeshBuilder.CreateGround("underwaterSurface", { width: 2048, height: 2048, subdivisions: 32 }, scene);
        const water = new WaterMaterial("water", scene);

        // Floating examples:
        // https://www.babylonjs-playground.com/#1FXV44#0 - objekts, kas krītot uz leju pakāpeniski samazina ātrumu, līdz kamēr apstājas (varētu simulēt situāciju, kad iemet objektu ūdenī no augšas)
        // https://www.babylonjs-playground.com/#L76FB1#49 - parasta vertikāla svārstība uz augšu un uz leju
        // https://playground.babylonjs.com/#SBG0X8#2 - peldēšanas kustība uz riņķi (var eksperimentēt ar `windDirection`)
        // https://playground.babylonjs.com/#SBG0X8#8 - mainīga peldēšana uz augšu un uz leju
        // https://www.babylonjs-playground.com/#L76FB1#120 - sarežģītāks algoritms
        // Algoritma apraksts: 
        //  object y = 
        //      map object x z to watermesh space .. P
        //      then find nearest 4 vertices (in watermesh space) of P
        //      then bilinear interp that 4 vertices

        
        // Ūdens virsmas parametri:
        // Definē pacēluma karti ("bump map"), kas simulē pacēlumus ("bumps") un iespiedumus ("dents") uz renderētas virsmas
        // Tiek paņemts attēls, no kura tiek veidota parastā karte jeb "normal map", kas reprezentē tās struktūru
        // water.bumpTexture = new Texture("assets/textures/waternormals.jpg", scene);
        water.bumpTexture = new Texture("assets/textures/waterbump.png", scene);
        water.bumpHeight = 0.5; // Nosaka ūdens tekstūras pacēlumu ("bump") augstumu - jo lielāka šī vērtība, jo izteiktākas ir tekstūras atšķirīgās pazīmes (augstās / zemās virsmas)
        water.colorBlendFactor = 0.2; // Nosaka, kā ūdens krāsa tiek jaukta kopā ar atstaroto ("reflected") un lauzto ("refracted") Babylon ainā definētās pasaules gaismu
	    water.waterColor = new Color3(0, 0, 221 / 255); // Ūdens virsmas un atstarotās gaismas jauktā krāsa RGB (R, G, B) formātā
        water.waveHeight = 0.1; // Maksimālais viļņu augstums
        // water.waveLength = 0.1; // Jo mazāks viļņu garums, jo vairāk viļņi tiek ģenerēti
        water.windDirection = new Vector2(1, 1); // Vektors ar koordinātēm (X, Z), kas norāda virzienu, kurā pūš vējš (jo lielāka katra koordināte, jo stiprāk vējš pūš uz to pusi)
        water.windForce = -10; // Vēja stiprums
        // water.waveSpeed = 50.0;

        // // Ūdens virsmas parametri:
        // water.bumpHeight = 0.5; // Nosaka ūdens tekstūras pacēlumu ("bump") augstumu - jo lielāka šī vērtība, jo izteiktākas ir tekstūras atšķirīgās pazīmes (augstās / zemās virsmas)
        // water.colorBlendFactor = 0.3; // Nosaka, kā ūdens krāsa tiek jaukta kopā ar atstaroto ("reflected") un lauzto ("refracted") Babylon ainā definētās pasaules gaismu
        // water.waterColor = new Color3(0.1, 0.1, 0.6); // Ūdens virsmas un atstarotās gaismas jauktā krāsa RGB formātā
        // water.waveHeight = 0.1; // Maksimālais viļņu augstums
        // water.waveLength = 0.1; // Jo mazāks viļņu garums, jo vairāk viļņi tiek ģenerēti
        // water.windDirection = new Vector2(1, 1); // Vektors ar koordinātēm (X, Z), kas norāda virzienu, kurā pūš vējš (jo lielāka katra koordināte, jo stiprāk vējš pūš uz to pusi)
        // water.windForce = 5; // Vēja stiprums
        // // water.waveSpeed = 50.0;

        // Pievieno debesu un zemes objektus pie ūdens virsmas kā gaismas atstarošanas un laušanas objektus
        // Ja tas netiek darīts, tad ūdens virsma nereaģēs pret gaismu, kas nāk no šiem objektiem, bet tā vietā "peldēs" virs tiem kā necaurspīdīga virsma
        // Debesu (skybox) gadījumā ūdens virsma nebūs pietiekami detalizēta; zemes virsmas (ground) gadījumā to nevarēs redzēt zem ūdens
        water.addToRenderList(skybox);
        water.addToRenderList(ground);
        
        // Assign the water material
        waterMaterial.material = water;

        // const ground = MeshBuilder.CreateGround("ground", {width: 32, height: 32}, scene);
        // ground.isPickable = false;
        
        // const box = MeshBuilder.CreateBox("box", { size: 2 }, scene);
        // box.position.y = 3;
        // box.isPickable = true;

        // Mouse pointer notikumi (events), kas skatās, uz kuru virzienu rotēt objektu:
        // scene.onPointerObservable.add((pointerInfo) => {
        //     const pickedMesh = pointerInfo.pickInfo.pickedMesh;
        //     if (!pickedMesh) {
        //         return;
        //     }

        //     if (pointerInfo.type === PointerEventTypes.POINTERDOWN) {
        //         console.log("type === 1", pointerInfo.pickInfo);
        //         rotating = true;
        //         camera.detachControl();
        //     } else if (pointerInfo.type === PointerEventTypes.POINTERUP && rotating) {
        //         console.log("type === 2", pointerInfo.pickInfo);
        //         rotating = false;
        //         camera.attachControl();
        //     } else if (pointerInfo.type === PointerEventTypes.POINTERMOVE && rotating) {
        //         console.log("type === 4", pointerInfo.pickInfo);

        //         const matrix = camera.getWorldMatrix();
        //         rightDir.copyFromFloats(matrix.m[0], matrix.m[1], matrix.m[2]);
        //         upDir.copyFromFloats(matrix.m[4], matrix.m[5], matrix.m[6]);

        //         football.rotateAround(football.position, rightDir, pointerInfo.event.movementY * -1 * sensitivity);
        //         football.rotateAround(football.position, upDir, pointerInfo.event.movementX * -1 * sensitivity);
        //     }
        // });

  
        // engine.runRenderLoop(() => {
        //     scene.render();
        // });

        canvas.onresize = function() {
			engine.resize();
		};
		window.onresize = function() {
			engine.resize();
		};
    
        return () => {
            engine.dispose();
        }
    });
</script>

<canvas bind:this={canvas}></canvas>

<style>
    canvas {
        width: 100%;
        height: 100vh;
        display: block;
    }
</style>
  