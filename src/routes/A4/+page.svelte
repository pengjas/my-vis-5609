<script lang="ts">
    import { onMount } from "svelte";
    import * as THREE from "three";
    import { FontLoader, Font } from "three/addons/loaders/FontLoader.js";
    import { TextGeometry } from "three/addons/geometries/TextGeometry.js";
    import { FirstPersonControls } from "three/addons/controls/FirstPersonControls.js";

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
    let container: HTMLElement; // The container element where the Three.js canvas will be added to the DOM
    // let controls: any; // Controls for interacting with the scene (e.g., camera controls like OrbitControls or PointerLockControls)
    let camera: THREE.PerspectiveCamera; // The camera used to view the scene, typically a perspective camera for 3D rendering
    let scene: THREE.Scene; // The scene where all the objects are added
    let renderer: THREE.WebGLRenderer; // The renderer used to render the scene
    const FLOOR = -250; // The floor level of the scene

    const morphs: Array<THREE.Mesh> = []; // Array to store the morphs (animated models: horse, flamingo, parrot)
    let mixer: THREE.AnimationMixer; // The mixer used to animate the morphs

    const clock = new THREE.Clock(); // Clock to keep track of time for animations

    onMount(
        // once the component mounts, load data then initialize the Three.js scene
        async () => {
            // Load the real movie data before initializing the scene
            genreData = await loadMovieData();
            init(window.innerWidth, window.innerHeight);
        },
    );

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
            30, // Wider field of view
            SCREEN_WIDTH / SCREEN_HEIGHT,
            10,
            3000,
        );
        camera.position.set(0, 100, 2200); // Moved back and up for better view

        // Scene
        scene = new THREE.Scene();
        // scene.background = new THREE.Color(0x87ceeb); // Sky color (light blue)
        new THREE.TextureLoader().load("3d/sky.jpg", (texture) => {
            texture.repeat.set(0.8, 1); // repeat the texture
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

        // // CONTROLS, the scene will move with the mouse

        // controls = new FirstPersonControls(camera, renderer.domElement);

        // controls.lookSpeed = 0.0125;
        // controls.movementSpeed = 500;
        // controls.lookVertical = true;
        // controls.lookAt(scene.position);

        // GROUND (Grass)
        // addGround(scene, FLOOR, "3d/Grass-Texture.jpg");
        addGround(scene, FLOOR, "3d/grasslight-big.jpg");

        // Add text and bars
        const fontLoader = new FontLoader();
        fontLoader.load("3d/helvetiker_bold.typeface.json", (font: Font) => {
            const textGeo = new TextGeometry("Summer Movies - Top Genres", {
                font: font,
                size: 35,
                depth: 12,
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
            titleMesh.position.y = FLOOR + 520; // Raised higher to avoid bar occlusion
            titleMesh.position.z = -50; // Moved slightly back
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
        const maxHeight = 400; // Maximum height of the bars
        const barMaxWidth = 900; // Increased width to accommodate 8 genres

        const xScale = d3
            .scaleBand()
            .domain(Object.keys(data))
            .range([-barMaxWidth / 2, barMaxWidth / 2])
            .padding(0.2); // Increased padding for better spacing

        const yScale = d3
            .scaleLinear()
            .domain([0, Math.max(...Object.values(data).map((d) => d))])
            .range([0, maxHeight]);

        // Create bars for each genre
        Object.keys(data).forEach((genre, i) => {
            const barHeight = yScale(data[genre]);
            const bar = createOneBar(
                barHeight,
                xScale.bandwidth(),
            );
            bar.position.set(
                xScale(genre)! + xScale.bandwidth() / 2, // Center the bar
                FLOOR + barHeight / 2,
                0,
            );
            scene.add(bar);

            // Position labels in front of bars with alternating heights to avoid occlusion
            const labelYOffset = i % 2 === 0 ? 20 : 60;
            addLabelToBar(
                scene,
                `${genre}: ${data[genre]}`,
                xScale(genre)! + xScale.bandwidth() / 2 - 30,
                FLOOR + labelYOffset,
                xScale.bandwidth() / 2 + 50, // Position in front of bar
                font,
            );
        });
    }

    function createOneBar(height: number, width: number) {
        // const geometry = new THREE.BoxGeometry(width, height, width);

        const geometry = new THREE.CylinderGeometry(
            width / 2,
            width / 2,
            height,
            32,
        ); // cylinder

        // const material = new THREE.MeshPhongMaterial({ color: color }); // can be either custom color or texture
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
            size: 12,
            depth: 4,
        });

        const textMaterial = new THREE.MeshPhysicalMaterial({
            color: 0xffffff,
        }); // Three.js provide a list of mesh materials, you can choose the one you like
        const textMesh = new THREE.Mesh(textGeometry, textMaterial);

        // Position the text above the bar
        textMesh.position.set(x, y, z);
        textMesh.castShadow = true;
        textMesh.receiveShadow = false; // Text will not receive shadow to make it more readable

        scene.add(textMesh);
    }

    function animate() {
        // this function will be called every frame
        const delta = clock.getDelta();
        mixer.update(delta);

        // You can comment out the following lines if you don't want the morphs to move, which will also save some computation
        morphs.forEach((morph) => {
            morph.position.x += morph.speed * delta;
            // Reset position if it goes off screen
            if (morph.position.x > window.innerWidth / 2) {
                morph.position.x = -window.innerWidth / 2 - Math.random() * 200;
            }
        });

        renderer.render(scene, camera);
        // controls.update(delta);
    }
</script>

<div bind:this={container} class="container"></div>

<div class="reflection">
    <h2>Reflection: Pros and Cons of 3D Visualization</h2>

    <section>
        <h3>Pros: Advantages of 3D Visualization</h3>
        <ul>
            <li><strong>Enhanced Engagement and Immersion:</strong> The 3D bar chart creates a more immersive and engaging experience compared to 2D charts. The depth, shadows, textures (wood grain on bars), animated elements (horses, flamingos, parrots), and realistic sky background capture viewer attention and make data exploration more enjoyable. This can be particularly effective for presentations or educational contexts where maintaining audience interest is crucial.</li>

            <li><strong>Aesthetic Appeal:</strong> The 3D visualization offers richer visual aesthetics through materials, lighting, shadows, and environmental elements. The wood texture on cylindrical bars, the grass ground, and animated birds create an artistic representation that can make data more memorable and visually impactful than flat 2D charts.</li>

            <li><strong>Spatial Representation Potential:</strong> While not fully utilized in this bar chart, 3D visualizations can encode additional data dimensions using the z-axis. This capability could be valuable for datasets where three or more variables need to be represented simultaneously, such as showing genre popularity across both time and region.</li>

            <li><strong>Novelty Factor:</strong> 3D visualizations can stand out and attract attention in contexts where audiences are accustomed to traditional 2D charts. This novelty can be advantageous for marketing, exhibitions, or when trying to make data memorable.</li>
        </ul>
    </section>

    <section>
        <h3>Cons: Drawbacks of 3D Visualization</h3>
        <ul>
            <li><strong>Occlusion and Perceptual Distortion:</strong> One of the most significant issues with 3D bar charts is that elements can occlude (hide) each other. Bars in the back may be partially hidden by those in front. Additionally, perspective projection causes objects farther from the camera to appear smaller, making accurate value comparison difficult. In our visualization, the cylindrical bars at different positions can be hard to compare precisely.</li>

            <li><strong>Cognitive Load:</strong> 3D visualizations require viewers to mentally process depth and perspective, increasing cognitive load. Users must understand the 3D space orientation before interpreting the data. The animated elements (birds, horses), while engaging, can also distract from the actual data being presented.</li>

            <li><strong>Precision and Accuracy Issues:</strong> Reading exact values from 3D charts is inherently more difficult than from 2D charts. The perspective distortion means that bars of equal height may appear different sizes based on their position in the scene. This violates a fundamental principle of effective visualization: enabling accurate data comparison.</li>

            <li><strong>Accessibility Concerns:</strong> 3D visualizations may be less accessible to users with visual impairments or those using screen readers. The static 2D bar chart from A1 can be more easily converted to accessible formats with alt-text descriptions.</li>

            <li><strong>Performance and Compatibility:</strong> 3D visualizations using WebGL/Three.js require more computational resources and may not work on all devices or browsers. The 2D D3.js chart from A1 has broader compatibility and faster rendering.</li>
        </ul>
    </section>

    <section>
        <h3>Proposed Solutions for Occlusion</h3>
        <p>To address the occlusion problem, several practical solutions could be implemented:</p>
        <ul>
            <li><strong>Interactive Camera Controls:</strong> Enabling users to rotate, zoom, and pan the 3D scene would allow them to view the data from multiple angles, reducing occlusion issues. Users could move to positions where all bars are visible and comparable. This maintains the immersive 3D experience while giving users control over their viewing perspective.</li>

            <li><strong>2.5D or Isometric View:</strong> Using an isometric or parallel projection instead of perspective projection would eliminate size distortion based on distance. All bars would maintain their true proportional sizes regardless of position, making comparison more accurate while preserving the 3D aesthetic.</li>

            <li><strong>Transparent or Semi-Transparent Materials:</strong> Making bars semi-transparent would allow viewers to see bars positioned behind others. This could be combined with edge highlighting to maintain shape clarity while reducing complete occlusion.</li>

            <li><strong>Linked 2D View:</strong> Providing a synchronized 2D view alongside the 3D visualization would offer the best of both worlds - engagement from 3D and precision from 2D. Users could enjoy the immersive experience while having access to accurate value readings.</li>
        </ul>
    </section>
</div>

<style>
    div.container {
        width: 100%;
        height: 100vh;
    }

    div.reflection {
        padding: 40px;
        max-width: 900px;
        margin: 0 auto;
        font-family: system-ui, -apple-system, sans-serif;
        line-height: 1.6;
        background: #f9f9f9;
    }

    .reflection h2 {
        color: #2e6600;
        border-bottom: 3px solid #449900;
        padding-bottom: 10px;
        margin-bottom: 30px;
    }

    .reflection h3 {
        color: #449900;
        margin-top: 25px;
        margin-bottom: 15px;
    }

    .reflection section {
        margin-bottom: 30px;
        padding: 20px;
        background: white;
        border-radius: 8px;
        box-shadow: 0 2px 4px rgba(0,0,0,0.1);
    }

    .reflection ul {
        margin: 0;
        padding-left: 20px;
    }

    .reflection li {
        margin-bottom: 12px;
    }

    .reflection strong {
        color: #2e6600;
    }

    .reflection p {
        margin-bottom: 15px;
    }
</style>
