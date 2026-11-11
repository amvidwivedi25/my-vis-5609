<script lang="ts">
    import { onMount } from "svelte";
    import * as THREE from "three";
    import { FontLoader, Font } from "three/addons/loaders/FontLoader.js";
    import { TextGeometry } from "three/addons/geometries/TextGeometry.js";
    import { GLTFLoader } from "three/addons/loaders/GLTFLoader.js";
    import type { TMovie } from "../../types";
    import { addGround, onWindowResize, loadModels } from "$lib/Helper-3D";
    import { base } from "$app/paths";
    import * as d3 from "d3";

    let movies: TMovie[] = $state([]);

    // Stacked data: each year has counts per genre
    type TStackRow = {
        year: string;
        Comedy: number;
        Romance: number;
        Drama: number;
    };

    let stackData: TStackRow[] = $state([]);
    const genres = ["Comedy", "Romance", "Drama"];
    const genreColors: Record<string, number> = {
        Comedy: 0xf4a261,  // warm orange
        Romance: 0xe76f51,  // coral red
        Drama: 0x2a9d8f,   // teal
    };

    // === TODO-1: Load Data ===
    // Currently using hardcoded dummy data.
    // TODO: Load the summer_movies.csv, then aggregate
    //   the real movie data into stackData.
    //   - Group movies by year (movie.year.getFullYear())
    //   - Count how many movies belong to each genre (Comedy, Romance, Drama)
    //   - Build a TStackRow[] array sorted by year
    async function loadCsv() {
        const data = await d3.csv(`${base}/summer_movies.csv`);
        movies = [];
        for (let d of data) {
            if (d.year === "NA" || !d.year) {
                continue;
            }
            movies.push({
                tconst: d.tconst,
                title_type: d.title_type,
                primary_title: d.primary_title,
                original_title: d.original_title,
                year: new Date(Number(d.year), 0, 1),
                runtime_minutes: Number(d.runtime_minutes),
                genres: d.genres.split(","),
                average_rating: Number(d.average_rating),
                num_votes: Number(d.num_votes)
            });
        }
        const years: number[] = [];
        for (let movie of movies) {
            const year = movie.year.getFullYear();
            if (!years.includes(year)) {
                years.push(year);
            }
        }
        years.sort((a, b) => a - b);
        stackData = [];
        for (let year of years) {
            let comedy = 0;
            let romance = 0;
            let drama = 0;
            for (let movie of movies) {
                if (movie.year.getFullYear() === year) {
                    if (movie.genres.includes("Comedy")) {
                        comedy++;
                    }

                    if (movie.genres.includes("Romance")) {
                        romance++;
                    }

                    if (movie.genres.includes("Drama")) {
                        drama++;
                    }
                }
            }
            stackData.push({
                year: String(year),
                Comedy: comedy,
                Romance: romance,
                Drama: drama
            });
        }
        console.log("Stack data:", stackData);
    }
    // === END TODO-1 ===

    let container: HTMLElement;
    let camera: THREE.PerspectiveCamera;
    let scene: THREE.Scene;
    let renderer: THREE.WebGLRenderer;
    const FLOOR = -250;

    const morphs: Array<THREE.Mesh> = [];
    let mixer: THREE.AnimationMixer;
    const clock = new THREE.Clock();

    const horseMorphs: Array<THREE.Mesh> = [];
    let horseMixer: THREE.AnimationMixer;
    const ORBIT_RX = 850;
    const ORBIT_RZ = 400;


    let animating = $state(false);

    onMount(async () => {
        await loadCsv();
        init(window.innerWidth, window.innerHeight);
    });

    function init(SCREEN_WIDTH: number, SCREEN_HEIGHT: number) {
        // Renderer
        renderer = new THREE.WebGLRenderer({ antialias: true });
        renderer.setPixelRatio(window.devicePixelRatio);
        renderer.setSize(SCREEN_WIDTH, SCREEN_HEIGHT);
        renderer.shadowMap.enabled = true;
        renderer.shadowMap.type = THREE.PCFShadowMap;
        container.appendChild(renderer.domElement);

        // Camera
        camera = new THREE.PerspectiveCamera(
            23,
            SCREEN_WIDTH / SCREEN_HEIGHT,
            10,
            5000,
        );
        camera.position.set(0, 100, 2800);

        // Scene
        scene = new THREE.Scene();
        new THREE.TextureLoader().load(`${base}/3d/sky.jpg`, (texture) => {
            texture.repeat.set(0.8, 1);
            scene.background = texture;
        });

        // Ambient Light
        const ambient = new THREE.AmbientLight(0xffffff);
        scene.add(ambient);

        // Directional Light
        const light = new THREE.DirectionalLight(0xffffff, 3);
        light.position.set(0, 1500, 1000);
        light.castShadow = true;
        Object.assign(light.shadow.camera, {
            top: 2000,
            bottom: -2000,
            left: -2000,
            right: 2000,
            near: 1200,
            far: 2500,
        });
        light.shadow.bias = 0.0001;
        light.shadow.mapSize.width = 2048;
        light.shadow.mapSize.height = 1024;
        scene.add(light);

        // Ground
        addGround(scene, FLOOR, `${base}/3d/grasslight-big.jpg`);

        // Title + Stacked Bars
        const fontLoader = new FontLoader();
        fontLoader.load(`${base}/3d/helvetiker_bold.typeface.json`, (font: Font) => {
            // Title
            const textGeo = new TextGeometry("summer movies", {
                font: font,
                size: 30,
                depth: 10,
            });
            textGeo.computeBoundingBox();
            const centerOffset =
                -0.5 *
                (textGeo!.boundingBox!.max.x - textGeo!.boundingBox!.min.x);
            const textMaterial = new THREE.MeshStandardMaterial({
                color: 0x449900,
            });
            const titleMesh = new THREE.Mesh(textGeo, textMaterial);
            titleMesh.position.x = centerOffset;
            titleMesh.position.y = FLOOR + 500;
            titleMesh.castShadow = true;
            scene.add(titleMesh);

            createUnits(scene, font);
            createLegend(scene, font);
        });

        // === TODO-2: Add Expressive 3D Objects ===
        // Make the scene more expressive, engaging and fun by changing or adding
        // at least one 3D object! You do NOT need to replace all existing objects —
        // modifying or adding just one is sufficient.
        // Animation is required (e.g., rotation, bobbing, or using a model's built-in animation clips).
        //
        // Where to find free GLB models:
        //   - https://poly.pizza (search, download as .glb)
        //   - https://sketchfab.com (filter by "Downloadable" + "glTF")
        //   - https://kenney.nl/assets (game-style assets, some 3D)
        //   - https://www.fab.com/search?q=3d+object&asset_formats=glb&asset_formats=converted-files&is_free=1
        //
        // Tips:
        //   - Place your .glb files in the /static/3d/ folder
        //   - Adjust the `scale` value if your model is too big or small
        //   - If a model has no built-in animations, add a custom one
        //     (e.g., rotate or float the mesh in the animate() function)
        //   - If a model has no animations, guard the animation code:
        //     if (gltf.animations.length > 0) { ... }
        //   - Feel free to add extra static models as decoration (trees, buildings, etc.)!

        // Load bird models (linear motion)
        const birdModels = [
            {
                path: `${base}/3d/Flamingo.glb`, // TODO: replace with your model
                speed: 350,
                duration: 1,
                x: 300 - Math.random() * 500,
                y: FLOOR + 550,
                z: 100,
                scale: 0.5,
            },
            {
                path: `${base}/3d/Flamingo.glb`, // TODO: replace with your model
                speed: 350,
                duration: 1,
                x: 300 - Math.random() * 500,
                y: FLOOR + 550,
                z: -100,
                scale: 0.5,
            },
            {
                path: `${base}/3d/Parrot.glb`, // TODO: replace with your model
                speed: 350,
                duration: 0.5,
                x: 500 - Math.random() * 500,
                y: FLOOR + 500,
                z: 700,
                scale: 0.5,
            },
        ];
        mixer = loadModels(birdModels, scene, mixer, morphs);

        // Load horse models (orbit around data)
        horseMixer = new THREE.AnimationMixer(scene);
        const gltfLoader = new GLTFLoader();
        const horseConfigs = [
            { startAngle: 0, orbitSpeed: 0.4 },
            { startAngle: Math.PI, orbitSpeed: 0.4 },
        ];
        horseConfigs.forEach((config) => {
            gltfLoader.load(`${base}/3d/Horse.glb`, (gltf) => { // TODO: replace with your model
                const mesh = gltf.scene.children[0].clone() as THREE.Mesh;
                (mesh as any).material = (mesh as any).material.clone();
                mesh.scale.set(0.5, 0.5, 0.5);
                mesh.castShadow = true;

                horseMixer
                    .clipAction(gltf.animations[0], mesh)
                    .setDuration(1)
                    .startAt(-Math.random())
                    .play();

                mesh.userData.angle = config.startAngle;
                mesh.userData.orbitSpeed = config.orbitSpeed;
                mesh.position.set(
                    ORBIT_RX * Math.cos(config.startAngle),
                    FLOOR,
                    ORBIT_RZ * Math.sin(config.startAngle),
                );

                scene.add(mesh);
                horseMorphs.push(mesh);
            });
        });
        // === END TODO-2 ===

        // Handle resize
        window.addEventListener("resize", () =>
            onWindowResize(
                camera,
                renderer,
                window.innerWidth,
                window.innerHeight,
            ),
        );

        renderer.setAnimationLoop(animate);
    }

    // === TODO-3: Unit Visualization ===
    //
    // TODO-3a: Define a unique 3D shape and material per genre.
    //   Docs: https://threejs.org/docs/#api/en/geometries
    //   Docs: https://threejs.org/docs/#api/en/materials/MeshPhysicalMaterial
    
    function createGenreGeometry(genre: string): THREE.BufferGeometry {
        if (genre === "Comedy") {
            return new THREE.BoxGeometry(8, 8, 8);
        }

        if (genre === "Romance") {
            return new THREE.SphereGeometry(5, 12, 8);
        }

        return new THREE.ConeGeometry(5, 10, 8);
    }

    function createGenreMaterial(genre: string): THREE.Material {
        return new THREE.MeshPhysicalMaterial({
            color: genreColors[genre]
        });
    }

    function createUnits(scene: THREE.Scene, font: Font) {
        if (stackData.length === 0) return;

        const barMaxWidth = Math.min(stackData.length * 80, 1400);
        const unitSize = 12;

        const xScale = d3
            .scaleBand()
            .domain(stackData.map((d) => d.year))
            .range([-barMaxWidth / 2, barMaxWidth / 2])
            .padding(0.1);

        const bandwidth = xScale.bandwidth();

        stackData.forEach((row) => {
            const xPos = xScale(row.year)! + bandwidth / 2;
            let currentY = FLOOR;

            genres.forEach((genre) => {
                const count = row[genre as keyof TStackRow] as number;

                for (let i = 0; i < count; i++) {
                    const geo = createGenreGeometry(genre);
                    const mat = createGenreMaterial(genre);
                    const mesh = new THREE.Mesh(geo, mat);

                    mesh.position.set(xPos, currentY + unitSize / 2, 0);
                    mesh.castShadow = true;
                    mesh.receiveShadow = true;
                    scene.add(mesh);

                    currentY += unitSize;
                }
            });
        });

        stackData.forEach((row) => {
            const x = xScale(row.year)! + bandwidth / 2;

            const textGeo = new TextGeometry(row.year, {
                font: font,
                size: 6,
                depth: 2
            });

            textGeo.computeBoundingBox();

            const textWidth =
                textGeo.boundingBox!.max.x - textGeo.boundingBox!.min.x;

            const textMat = new THREE.MeshPhysicalMaterial({
                color: 0x000000
            });

            const textMesh = new THREE.Mesh(textGeo, textMat);

            textMesh.position.set(
                x - textWidth / 2,
                FLOOR + 5,
                15
            );

            textMesh.castShadow = true;
            scene.add(textMesh);
        });
    }

    function createLegend(scene: THREE.Scene, font: Font) {
        const legendX = 180;
        const legendStartY = FLOOR + 420;
        const spacing = 40;

        genres.forEach((genre, i) => {
            const y = legendStartY - i * spacing;

            const geo = createGenreGeometry(genre);
            const mat = createGenreMaterial(genre);
            const mesh = new THREE.Mesh(geo, mat);

            mesh.position.set(legendX, y, 0);
            mesh.scale.set(1.5, 1.5, 1.5);
            mesh.castShadow = true;
            scene.add(mesh);

            const textGeo = new TextGeometry(genre, {
                font: font,
                size: 10,
                depth: 3
            });

            const textMat = new THREE.MeshPhysicalMaterial({
                color: 0x000000
            });

            const textMesh = new THREE.Mesh(textGeo, textMat);

            textMesh.position.set(legendX + 18, y - 5, 0);
            textMesh.castShadow = true;
            scene.add(textMesh);
        });
    }

    function animate() {
        const delta = clock.getDelta();

        if (animating) {
            mixer.update(delta);
            horseMixer.update(delta);

            // TODO: Customize the animation behavior for your own models.
            // Birds - linear motion
            morphs.forEach((morph) => {
                morph.position.x += morph.speed * delta;
                if (morph.position.x > window.innerWidth / 2) {
                    morph.position.x = -window.innerWidth / 2 - Math.random() * 200;
                }
            });

            // TODO: Customize the animation for your own models.
            // Horses - orbit around the data
            horseMorphs.forEach((horse) => {
                horse.userData.angle += horse.userData.orbitSpeed * delta;
                const angle = horse.userData.angle;
                horse.position.x = 650 * Math.sin(angle);
                horse.position.z = 250 * Math.sin(angle * 2);
                horse.position.y = FLOOR + 10;
                horse.scale.set(0.7, 0.7, 0.7);
                const nextAngle = angle + 0.05;
                horse.lookAt(
                    650 * Math.sin(nextAngle),
                    horse.position.y,
                    250 * Math.sin(nextAngle * 2)
                );
            });
        }
        renderer.render(scene, camera);
    }
</script>

<div bind:this={container} class="container"></div>

<button class="toggle-btn" onclick={() => animating = !animating}>
    {animating ? "Pause" : "Play"} Animation
</button>

<style>
    div.container {
        width: 100%;
        height: 100%;
    }

    .toggle-btn {
        position: fixed;
        bottom: 10px;
        right: 10px;
        padding: 8px 10px;
        font-size: 12px;
        border: none;
        border-radius: 20px;
        background: rgba(0, 0, 0, 0.30);
        color: white;
        cursor: pointer;
        z-index: 10;
    }

    .toggle-btn:hover {
        background: rgba(0, 0, 0, 0.8);
    }
</style>
