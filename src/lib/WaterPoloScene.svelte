<script>
    import { onMount } from 'svelte';

    // Havok Physics (no Babylon):
    // import { PhysicsEngine, HavokPlugin } from '@babylonjs/core/Physics';
    // import HavokPhysics from '@babylonjs/havok';

    import * as BABYLON from "@babylonjs/core";
    
    // Automātiski reģistrē visus nepieciešamos failu lejupielādes moduļus (tai skaitā glTF un GLB)
    import '@babylonjs/loaders';

    // Oficiālais Babylon ūdens virsmas materiāls
    import { SkyMaterial, WaterMaterial } from '@babylonjs/materials';
    import { waterVertexShader } from '@babylonjs/materials/water/water.vertex';

    // Ammo.js (fizikas bibliotēka):
    import Ammo from 'ammojs-typed';

    let canvas;

    let football;
    let footballGoal;

    let skybox;
    /** @type {WaterMaterial} */
    let water;
    let ground;

    let initializedHavok;

    // Ūdens vienmēr atradīsies y = 0 koordinātā:
    const waterLevel = 0;
    
    // Definē animāciju, kas izpildās uz noteiktu objektu 5 sekunžu laikā (parametrs 0.2, kas atbilst 1/5 no 1 sekundes) no sākuma punkta (from) uz beigu punktu (to):
    const animateObject = (object, propertyName, from, to, scene) => {
        var keys = [
            { frame: 0, value: from },
            { frame: 100, value: to }
        ];
        
        var animation = new BABYLON.Animation("animation", propertyName, 100, BABYLON.Animation.ANIMATIONTYPE_FLOAT, BABYLON.Animation.ANIMATIONLOOPMODE_CONSTANT);
        animation.setKeys(keys);
        
        scene.stopAnimation(object);
        scene.beginDirectAnimation(object, [animation], 0, 100, false, 0.2);
    };

    // Debesis kā kaste, kura iekļauj pārējos modeļus (tās materiāls - saule):
    const initSkybox = (scene) => {
        const skyboxMaterial = new SkyMaterial("skyBox", scene);
        skyboxMaterial.backFaceCulling = false;
        // `CubeTexture` ļauj norādīt 6 bildes ar vienādu nosaukumu formātā `NOSAUKUMS_ab`, 
        // kur a un b var būt viena no šādām vērtībām: px, py, pz (pirmās trīs bildes), nx, ny, nz (nākošās trīs bildes)
        //  kur a - pirmās 3 bildes (x, y, z asu pozitīvajā virzienā), b - nākošās 3 bildes (x, y, z asu negatīvajā virzienā)
        // Šajā gadījumā tiks meklētas 6 bildes ar nosaukumu `TropicalSunnyDay_ab`
        // skyboxMaterial.reflectionTexture = new BABYLON.CubeTexture("assets/textures/TropicalSunnyDay", scene);
        // skyboxMaterial.reflectionTexture.coordinatesMode = BABYLONTexture.SKYBOX_MODE;
        // skyboxMaterial.diffuseColor = new BABYLON.Color3(0, 0, 0);
        // skyboxMaterial.specularColor = new BABYLON.Color3(0, 0, 0);
        // skyboxMaterial.disableLighting = true;
        skyboxMaterial.fogEnabled = true;

        skybox = BABYLON.MeshBuilder.CreateBox("skyBox", {size: 4000, width: 4000, height: 4000}, scene);
        skybox.material = skyboxMaterial;
    };

    const initGround = (scene) => {
        // Zemes virsma (virs kuras attēlos ūdens virsmu):
        const groundMaterial = new BABYLON.StandardMaterial("groundMaterial", scene);
	    groundMaterial.diffuseTexture = new BABYLON.Texture("assets/textures/ground.jpg", scene);
        groundMaterial.diffuseTexture.uScale = groundMaterial.diffuseTexture.vScale = 6;
        groundMaterial.specularColor = new BABYLON.Color3(0, 0, 0);
        
        ground = BABYLON.MeshBuilder.CreateGround("ground", { width: 2048, height: 2048, subdivisions: 32 }, scene);
        ground.position.y = -10;
        ground.material = groundMaterial;
    };

    // Ūdens virsmas izveide un pievienošana ainai:
    const initBasicWater = (scene) => {
        const waterMaterial = BABYLON.MeshBuilder.CreateGround("underwaterSurface", { width: 2048, height: 2048, subdivisions: 32 }, scene);
        water = new WaterMaterial("water", scene);

        // Daži citi piemēri peldoša ķermeņa kustības vizualizācijai:
        // https://www.babylonjs-playground.com/#1FXV44#0 - objekts, kas krītot uz leju pakāpeniski samazina ātrumu, līdz kamēr apstājas (varētu simulēt situāciju, kad iemet objektu ūdenī no augšas)
        // https://www.babylonjs-playground.com/#L76FB1#49 - parasta vertikāla svārstība uz augšu un uz leju
        // https://playground.babylonjs.com/#SBG0X8#2 - peldēšanas kustība uz riņķi (var eksperimentēt ar `windDirection`)
        // https://playground.babylonjs.com/#SBG0X8#8 - mainīga peldēšana uz augšu un uz leju
        // https://www.babylonjs-playground.com/#L76FB1#120 - sarežģītāks algoritms (ideja, ar kuras palīdzību varētu simulēt objekta iekrišanu ūdenī no gaisa)

        // Ūdens virsmas parametri:
        // Definē pacēluma karti ("bump map"), kas simulē pacēlumus ("bumps") un iespiedumus ("dents") uz renderētas virsmas
        // Tiek paņemts attēls, no kura tiek veidota parastā karte jeb "normal map", kas reprezentē tās struktūru
        // water.bumpTexture = new Texture("assets/textures/waternormals.jpg", scene);
        water.backFaceCulling = true;
        water.bumpTexture = new BABYLON.Texture("assets/textures/waterbump.png", scene);
        water.bumpHeight = 0.5; // Nosaka ūdens tekstūras pacēlumu ("bump") augstumu - jo lielāka šī vērtība, jo izteiktākas ir tekstūras atšķirīgās pazīmes (augstās / zemās virsmas)
        water.colorBlendFactor = 0.0; // Nosaka, kā ūdens krāsa tiek jaukta kopā ar atstaroto ("reflected") un lauzto ("refracted") Babylon ainā definētās pasaules gaismu
	    water.waterColor = new BABYLON.Color3(0, 0, 221 / 255); // Ūdens virsmas un atstarotās gaismas jauktā krāsa RGB (R, G, B) formātā
        water.waveHeight = 0.5; // Maksimālais viļņu augstums
        // water.waveLength = 0.1; // Jo mazāks viļņu garums, jo vairāk viļņi tiek ģenerēti
        water.windDirection = new BABYLON.Vector2(1, 1); // Vektors ar koordinātēm (X, Z), kas norāda virzienu, kurā pūš vējš (jo lielāka katra koordināte, jo stiprāk vējš pūš uz to pusi)
        water.windForce = -10; // Vēja stiprums
        // water.waveSpeed = 50.0;

        // // Ūdens virsmas parametri v2:
        // water.bumpHeight = 0.5;
        // water.colorBlendFactor = 0.3;
        // water.waterColor = new BABYLON.Color3(0.1, 0.1, 0.6);
        // water.waveHeight = 0.1;
        // water.waveLength = 0.1;
        // water.windDirection = new BABYLON.Vector2(1, 1);
        // water.windForce = 5;

        // Pievieno visus ielādētos objektus pie ūdens virsmas kā gaismas atstarošanas un laušanas objektus
        // Ja tas netiek darīts, tad ūdens virsma nereaģēs pret gaismu, kas nāk no šiem objektiem, bet tā vietā "peldēs" virs tiem kā necaurspīdīga virsma
        // Debesu (skybox) gadījumā ūdens virsma nebūs pietiekami detalizēta; zemes virsmas (ground) gadījumā to nevarēs redzēt zem ūdens
        water.addToRenderList(skybox);
        water.addToRenderList(ground);
        water.addToRenderList(football);
        water.addToRenderList(footballGoal);
        
        // Assign the water material
        waterMaterial.material = water;

        // Registers a function to be called before every frame render
        // Funkcija, kas tiek izpildīta pirms katra renderētā 3D ainas kadra
        // Pamata ideja ir ar trigonometrisko funkciju palīdzību simulēt, kā laika gaitā objekta vertikālā atrašanās vieta mainās
        // atkarībā no ūdenim piemērotā viļņu ātruma, viļņu augstuma, vēja virziena ()
        // Rezultātā tiek iegūta kustība, kas vizuāli atdarina to, kā fizisks objekts varētu peldēt ūdenī
        const initialFootballGoalPositionY = footballGoal.position.y;
        scene.registerBeforeRender(() => {
            let time = water._lastTime / 100000;
            let x = footballGoal.position.x;
            let z = footballGoal.position.z;
            footballGoal.position.y = Math.abs(
                (Math.sin(((x / 0.9) + time * water.waveSpeed * 30)) * water.waveHeight * water.windDirection.x * 0.5) + // water.windDirection.x = vēja virziena x koordināta
                (Math.cos(((z / 0.05) +  time * water.waveSpeed * 30)) * water.waveHeight * water.windDirection.y * 2.0)  // water.windDirection.y = vēja virziena z koordināta
            );
        });

    };
    
    // Simulē ūdens fizikālās īpašības - piemēro nelielu spēku norādītajam `mesh` objektam, to pabīdot nedaudz uz augšu:
    const applyBuoyancyEffect = (mesh) => {
        const buoyancyForce = 10;
        const gravity = -9.81;
        mesh.physicsImpostor.applyForce(
            new BABYLON.Vector3(0, buoyancyForce + gravity * mesh.physicsImpostor.mass, 0),
            mesh.getAbsolutePosition()
        );
    };

    const initPhysics = async (scene) => {
        // Fizikālās īpašības:
        const ammoModule = await Ammo.call({});
        // const plugin = new AmmoJSPlugin(true, ammoModule)

        // const havokInstance = await HavokPhysics();
        // const plugin = new HavokPlugin(true, havokInstance);

        const plugin = new BABYLON.AmmoJSPlugin(true, ammoModule);
        // scene.enablePhysics(new BABYLON.Vector3(0, 0, 0), plugin);
        scene.enablePhysics(undefined, plugin);

        football.physicsImpostor = new BABYLON.PhysicsImpostor(football, BABYLON.PhysicsImpostor.SphereImpostor, { mass: 1, friction: 0.5, restitution: 0.9 }, scene);
	    ground.physicsImpostor = new BABYLON.PhysicsImpostor(ground, BABYLON.PhysicsImpostor.BoxImpostor, { mass: 0, friction: 0.5, restitution: 0.9 }, scene);

        football.physicsImpostor._physicsBody.linearDamping = 0.3;
        football.physicsImpostor._physicsBody.angularDamping = 0.3;

        const sphere = BABYLON.MeshBuilder.CreateSphere("sphere1", { segments: 16, diameter: 2 }, scene);
        
        sphere.position.y = 8;
        sphere.physicsImpostor = new BABYLON.PhysicsImpostor(sphere, BABYLON.PhysicsImpostor.SphereImpostor, { mass: 1, restitution: 0.9 }, scene);

        football.physicsImpostor.registerOnPhysicsCollide(ground.physicsImpostor, (main, collided) => {
            console.log("Football has collided with ground", main, collided);
        });

        football.physicsImpostor.registerOnPhysicsCollide(sphere.physicsImpostor, (main, collided) => {
            console.log("Football has collided with sphere", main, collided);
        });

        // Inicializē 'velc-un-nomet' animāciju ("drag and drop"):
        const sixDofDragBehavior = new BABYLON.SixDofDragBehavior();
        sixDofDragBehavior.dragDeltaRatio = 0.2; // Jo mazāka vērtība, jo gludāka vilkšanas animācija
        sixDofDragBehavior.zDragFactor = 2; // Nosaka, cik ātri objektam ir jākustas, ja to velk virzienā uz sevi jeb pa z asi (noderīgi, ja objekts, kas tiek vilkts, atrodas tālu no kameras)

        sixDofDragBehavior.onDragStartObservable.add((event) => {
            console.log("starting drag: ", event);
            // plugin.setGravity(new BABYLON.Vector3(0,0,0));
        });
        
        sixDofDragBehavior.onDragObservable.add((event) => {
            console.log("in the middle of drag: ", event);
            // football.position.x = event.position.x;
            // football.position.y = event.position.y;
            // football.position.z = event.position.z;
        });
        sixDofDragBehavior.onDragEndObservable.add((event) => {
            console.log("ending drag: ", event);
            // plugin.setGravity(new BABYLON.Vector3(0,-9.81,0));
        });
        
        football.addBehavior(sixDofDragBehavior);
    }
    
    const initScene = async () => {
        const engine = new BABYLON.Engine(canvas, true);
        const scene = new BABYLON.Scene(engine);

        // Definē rotācijas kameras objektu ar šādiem parametriem:
        //  *) Objekta nosaukums
        //  *) sākotnējā rotācija ap meridiānu (longitude) jeb horizontāli (radiānos)
        //  *) sākotnējā rotācija ap paralēli (latitude) jeb vertikāli (radiānos)
        //  *) sākotnējais kameras attālums no tās fokusa objekta ("target")
        //  *) 3D vektors (X, Y, Z), kas atbilst kameras fokusa objektam ("target")
        //  *) galvenās 3D ainas objekts
        // const camera = new BABYLON.ArcRotateCamera("Camera", 3 * Math.PI / 2, Math.PI / 4, 100, BABYLON.Vector3.Zero(), scene);
        const camera = new BABYLON.ArcRotateCamera("Camera", 3 * Math.PI / 2, 5 * Math.PI / 12, 40, BABYLON.Vector3.Zero(), scene);  // deg(5 * Math.PI / 12) = 75°
        camera.lowerRadiusLimit = 10;
        camera.upperRadiusLimit = 500;
        camera.useBouncingBehavior = true;
        // const camera = new BABYLON.FreeCamera("camera", new BABYLON.Vector3(0, 5, -10), scene);
        // const camera = new BABYLON.UniversalCamera("Camera", new BABYLON.Vector3(0, 0, -10), scene)
        // camera.setTarget(BABYLON.Vector3.Zero())
        camera.attachControl(canvas, true);
    
        const light = new BABYLON.HemisphericLight("light", new BABYLON.Vector3(0, 1, 0), scene)
        // light.intensity = 0.7;

        // failu atrašanās vietai jābūt `public` mapē, lai Babylon loader to uzreiz pamanītu:
        const assetsManager = new BABYLON.AssetsManager(scene);
        const meshTask = assetsManager.addMeshTask("football goal task", "", "/assets/models/", "football_goal.glb", ".glb");
        meshTask.onSuccess = (task) => {
            footballGoal = task.loadedMeshes[0];
            footballGoal.scaling = new BABYLON.Vector3(5, 5, 5);
            footballGoal.position.y = 1;
            footballGoal.position.z = 20;
            console.log("football goal loaded: ", footballGoal);
        };
        meshTask.onError = (task, message, exception) => {
            console.error(task, message, exception);
        };
        
        const meshTaskTwo = assetsManager.addMeshTask("football task", "", "/assets/models/", "football.gltf", ".gltf");
        meshTaskTwo.onSuccess = (task) => {
            football = task.loadedMeshes[0];
            console.log("football loaded: ", football);
            
            football.position.x = 4;
            football.position.y = 15;
        };
        meshTaskTwo.onError = (task, message, exception) => {
            console.error(task, message, exception);
        };
        
        assetsManager.onFinish = async (tasks) => {
            initSkybox(scene);
        
            initGround(scene);

            initBasicWater(scene);
            // Uzstāda debesis tā, lai tās atbilstu gaišam dienas laikam (kad saule atrodas tieši virs galvas), mainot tās materiāla (šajā gadījumā - saules) slīpumu (inclination)
            animateObject(skybox, "material.inclination", skybox.material.inclination, 0, scene);
                        
            await initPhysics(scene);

            engine.runRenderLoop(() => {
                if (football.position.y <= waterLevel && football.position.y >= ground.position.y) {
                    console.log("Football is in the water");
                    applyBuoyancyEffect(football);
                } else if (football.position.y > waterLevel) {

                }
                scene.render();
            });
        };
        
        assetsManager.load();

        canvas.onresize = function() {
			engine.resize();
		};
		window.onresize = function() {
			engine.resize();
		};
    
        return () => {
            engine.dispose();
        };
    };
  
    onMount(async () => {
        await initScene();
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
  