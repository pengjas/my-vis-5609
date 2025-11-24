<script lang="ts">
    import { onMount } from "svelte";
    import * as THREE from "three";
    import { FontLoader, Font } from "three/addons/loaders/FontLoader.js";
    import { TextGeometry } from "three/addons/geometries/TextGeometry.js";

    import { addGround, onWindowResize, loadModels } from "$lib/Helper-3D";

    import * as d3 from "d3";

    // Genre counts from real summer movie data
    type TGenreData = {
        [key: string]: number;
    };
    let genreData: TGenreData = {};

    // Load and process the summer movies CSV data
    async function loadMovieData(): Promise<TGenreData> {
        const csvUrl = "./summer_movies.csv";
        const movies = await d3.csv(csvUrl);

        const genreCounts: TGenreData = {};
        movies.forEach((movie) => {
            if (movie.genres) {
                const genres = movie.genres.split(",");
                genres.forEach((genre) => {
                    const trimmedGenre = genre.trim();
                    genreCounts[trimmedGenre] = (genreCounts[trimmedGenre] || 0) + 1;
                });
            }
        });

        // Get top 8 genres to avoid visual clutter
        const sortedGenres = Object.entries(genreCounts)
            .sort((a, b) => b[1] - a[1])
            .slice(0, 8);

        const topGenres: TGenreData = {};
        sortedGenres.forEach(([genre, count]) => {
            topGenres[genre] = count;
        });

        return topGenres;
    }

    // global variables for defining and controlling the 3D scene
    let container: HTMLElement;
    let camera: THREE.PerspectiveCamera;
    let scene: THREE.Scene;
    let renderer: THREE.WebGLRenderer;
    const FLOOR = -250;

    const morphs: Array<THREE.Mesh> = [];
    let mixer: THREE.AnimationMixer;

    const clock = new THREE.Clock();

    onMount(async () => {
        // Load the real movie data before initializing the scene
        genreData = await loadMovieData();
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

        // Camera - adjusted FOV and position for wider bar chart with 8 genres
        camera = new THREE.PerspectiveCamera(
            30,
            SCREEN_WIDTH / SCREEN_HEIGHT,
            10,
            3000,
        );
        camera.position.set(0, 100, 2200);

        // Scene
        scene = new THREE.Scene();
        new THREE.TextureLoader().load("3d/sky.jpg", (texture) => {
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

        // GROUND
        addGround(scene, FLOOR, "3d/grasslight-big.jpg");

        // Add text and bars
        const fontLoader = new FontLoader();
        fontLoader.load("3d/helvetiker_bold.typeface.json", (font: Font) => {
            const textGeo = new TextGeometry("Summer Movies - Top Genres", {
                font: font,
                size: 35,
                depth: 5,
            });
            textGeo.computeBoundingBox();
            const centerOffset =
                -0.5 *
                (textGeo!.boundingBox!.max.x - textGeo!.boundingBox!.min.x);
            
            // Title Material
            const textMaterial = new THREE.MeshStandardMaterial({
                color: 0x1a3c00, // Dark Green
            });
            const titleMesh = new THREE.Mesh(textGeo, textMaterial);
            titleMesh.position.x = centerOffset;
            titleMesh.position.y = FLOOR + 550; 
            titleMesh.position.z = -50;
            titleMesh.castShadow = true;
            scene.add(titleMesh);

            createBars(scene, font, genreData);
        });

        // Load Models
        const models = [
            {
                path: "3d/Horse.glb",
                speed: 300,
                duration: 1,
                x: 100 - Math.random() * 1000,
                y: FLOOR,
                z: 300,
                scale: 0.5,
            },
            {
                path: "3d/Horse.glb",
                speed: 300,
                duration: 1,
                x: 100 - Math.random() * 1000,
                y: FLOOR,
                z: 450,
                scale: 0.5,
            },
            {
                path: "3d/Flamingo.glb",
                speed: 350,
                duration: 1,
                x: 300 - Math.random() * 500,
                y: FLOOR + 550,
                z: 100,
                scale: 0.5,
            },
            {
                path: "3d/Flamingo.glb",
                speed: 350,
                duration: 1,
                x: 300 - Math.random() * 500,
                y: FLOOR + 550,
                z: 200,
                scale: 0.5,
            },
            {
                path: "3d/Parrot.glb",
                speed: 350,
                duration: 0.5,
                x: 500 - Math.random() * 500,
                y: FLOOR + 500,
                z: 700,
                scale: 0.5,
            },
        ];
        mixer = loadModels(models, scene, mixer, morphs);

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

    function createBars(scene: THREE.Scene, font: Font, data: TGenreData) {
        const maxHeight = 400; 
        const barMaxWidth = 900; 

        const xScale = d3
            .scaleBand()
            .domain(Object.keys(data))
            .range([-barMaxWidth / 2, barMaxWidth / 2])
            .padding(0.2); 

        const yScale = d3
            .scaleLinear()
            .domain([0, Math.max(...Object.values(data).map((d) => d))])
            .range([0, maxHeight]);

        Object.keys(data).forEach((genre, i) => {
            const barHeight = yScale(data[genre]);
            const bar = createOneBar(
                barHeight,
                xScale.bandwidth(),
            );
            bar.position.set(
                xScale(genre)! + xScale.bandwidth() / 2,
                FLOOR + barHeight / 2,
                0,
            );
            scene.add(bar);

            // Place labels ON TOP of the bars to avoid occlusion
            const labelYPosition = FLOOR + barHeight + 15; 

            addLabelToBar(
                scene,
                `${genre}: ${data[genre]}`,
                xScale(genre)! + xScale.bandwidth() / 2 - 40, 
                labelYPosition, 
                50, 
                font,
            );
        });
    }

    function createOneBar(height: number, width: number) {
        const geometry = new THREE.CylinderGeometry(
            width / 2,
            width / 2,
            height,
            32,
        ); 

        const material = new THREE.MeshStandardMaterial({
            map: new THREE.TextureLoader().load("./3d/wood-texture.jpg"),
        });
        const bar = new THREE.Mesh(geometry, material);
        bar.castShadow = true;
        bar.receiveShadow = true;
        return bar;
    }

    function addLabelToBar(
        scene: THREE.Scene,
        text: string,
        x: number,
        y: number,
        z: number,
        font: Font,
    ) {
        const textGeometry = new TextGeometry(text, {
            font: font,
            size: 14, 
            depth: 2, 
        });

        const textMaterial = new THREE.MeshPhysicalMaterial({
            color: 0x1a3c00, // Dark Green
        }); 
        const textMesh = new THREE.Mesh(textGeometry, textMaterial);

        textMesh.position.set(x, y, z);
        textMesh.castShadow = false; 
        textMesh.receiveShadow = false;

        scene.add(textMesh);
    }

    function animate() {
        const delta = clock.getDelta();
        mixer.update(delta);

        morphs.forEach((morph) => {
            morph.position.x += morph.speed * delta;
            if (morph.position.x > window.innerWidth / 2) {
                morph.position.x = -window.innerWidth / 2 - Math.random() * 200;
            }
        });

        renderer.render(scene, camera);
    }
</script>

<div bind:this={container} class="container"></div>

<style>
    div.container {
        width: 100%;
        height: 100vh;
        overflow: hidden; /* Prevents scrollbars */
        margin: 0;
        padding: 0;
    }
</style>