Three.js helper to create multi passes post processing effects.

Based on this [excellent article](https://medium.com/@luruke/simple-postprocessing-in-three-js-91936ecadfb7) by [luruke](https://github.com/luruke), this allows to easily add multiple passes post processing to your three.js renders.

<h2>Installation</h2>
    
```html
<script src="three.multipass.post.processing.min.js"></script>
```

There's also an ESM class based module available in the dist folder: three.multipass.post.processing.esm.min.js

<h2>Initializing</h2>

```javascript
// assuming 'renderer' is a THREE.WebGLRenderer class object
const multiPostFX = new MultiPostFX({
    renderer: renderer,
    passes: [
        {
            fragmentShader: `
                precision highp float;
                uniform sampler2D uScene;
                uniform vec2 uResolution;
                void main() {
                    // get uvs
                    vec2 uv = gl_FragCoord.xy / uResolution.xy;
                    vec4 color = texture2D(uScene, uv);
                    // Do your cool postprocessing here
                    color.r += sin(uv.x * 50.0);
                    gl_FragColor = color;
                }
            `
        }
    ]
});

// ...

// in your rendering loop, instead of using 'renderer.render(scene, camera)'
multiPostFX.render(scene, camera);
```

<h2>Parameters</h2>

`renderer`: your `THREE.WebGLRenderer` object used to render your scene initially

`passes`: an object where each pass is an object with these parameters:

| Parameter  | Type | Default | Description |
| --- | --- | --- | --- |
| vertexShader  | String | see below | The vertex shader used by your pass |
| fragmentShader | String | see below | The fragment shader used by your pass (this is where you'll do most of your postprocessing stuff) |
| uniforms | object | null | Additional uniforms to use in your shaders. Will be merged with the built-in uniforms. Should respect [three.js Uniform object structure](https://threejs.org/docs/#api/en/core/Uniform) |
| format | [THREE texture constants format](https://threejs.org/docs/#api/en/constants/Textures) | THREE.RGBAFormat | The texture format property (whether to use alpha for example) |

<h2>Default built-in shaders and uniforms</h2>

<h3>Default uniforms</h3>

There are two default uniforms that your passes will always use:

`uScene` (three.js [Texture](https://threejs.org/docs/#api/en/textures/Texture)): a texture containing the scene to which post processing will be applied.
`uResolution` (three.js [Vector2](https://threejs.org/docs/#api/en/math/Vector2)): your renderer parameter sizes, used to calculate the UV in your fragment shader. 

<h3>Built-in shaders</h3>

<h4>Vertex shader</h4>

If you don't want to pass any varyings from your vertex shader to your fragment shader, don't specify any vertexShader property and your post processing pass will use the default one:

```glsl
precision highp float;
attribute vec2 position;
void main() {
    gl_Position = vec4(position, 1.0, 1.0);
}
```

<h4>Fragment shader</h4>

If you don't specify any fragment shader, then your scene will be rendered as is by using this fragment shader (note how the UV are calculated thanks to the `uResolution` uniform):

```glsl
precision highp float;
uniform sampler2D uScene;
uniform vec2 uResolution;
void main() {
    vec2 uv = gl_FragCoord.xy / uResolution.xy;
    gl_FragColor = texture2D(uScene, uv);
}
```

<h2>Methods</h2>

| Method  | parameters | Description |
| --- | --- | --- |
| render  | scene, camera | Renders your scene with all the post processing passes applied |
| resize  | - | Resize the passes (use it after having resized your scene and updated your camera aspect and matrix) |

<h3>Example</h3>

```javascript
let ww = window.innerWidth;
let wh = window.innerHeight;

let scene = new THREE.Scene();
let camera = new THREE.PerspectiveCamera(45, ww / wh, 0.1, 2000);
camera.position.set(0, 0, 3);
camera.lookAt(new THREE.Vector3(0, 0, 0));

scene.add(camera);

let renderer = new THREE.WebGLRenderer({
    alpha: true,
    antialias: false // post processing will disable default antialiasing anyway
});
renderer.setSize(ww, wh);
document.body.appendChild(renderer.domElement);

// add a simple point light and a cube
let light = new THREE.PointLight(0xffffff, 1, 0);
light.position.set(10, 10, 10);
scene.add(light);

let box = new THREE.Mesh(
    new THREE.BoxGeometry(1, 1),
    new THREE.MeshPhongMaterial({
        color: 0x156289,
        emissive: 0x072534,
    })
);

scene.add(box);


let passes = {
    distortion: {
        uniforms: {
            uTime: {
                value: 0
            }
        },
        fragmentShader: `
            precision highp float;
            uniform sampler2D uScene;
            uniform vec2 uResolution;
            uniform float uTime;
            void main() {
                vec2 uv = gl_FragCoord.xy / uResolution.xy;
                // displace UV based on time uniform
                uv += vec2(
                    sin(uv.y * 25.0) * cos(uv.x * 25.0) * (cos(uTime / 60.0)) / 50.0,
                    cos(uv.y * 25.0) * sin(uv.x * 25.0) * (sin(uTime / 60.0)) / 50.0
                );
                gl_FragColor = texture2D(uScene, uv);
            }
        `
    },
    invertHalfScreenColors: {
        fragmentShader: `
            precision highp float;
            uniform sampler2D uScene;
            uniform vec2 uResolution;
            
            void main() {
                vec2 uv = gl_FragCoord.xy / uResolution.xy;
                vec4 color = texture2D(uScene, uv);
                // invert screen's right half colors 
                if(uv.x >= 0.5) {
                    color.rgb = (1.0 - color.rgb) * color.a;
                }
                gl_FragColor = color;
            }
        `,
    },
    // you can add another pass like FXAA if you want...
};

let postFX = new MultiPostFX({
    renderer: renderer,
    passes: passes
});

// handle resize
window.addEventListener("resize", () => this.onResize())

// render everything
animate();

function animate() {
    // continuously rotate our cube
    box.rotation.x += 0.005;
    box.rotation.y += 0.005;

    // increase our distortion pass' time uniform
    postFX.passes.distortion.material.uniforms.uTime.value++;
    // render everything
    postFX.render(scene, camera);

    requestAnimationFrame(() => this.animate());
}

function onResize() {
    ww = window.innerWidth;
    wh = window.innerHeight;

    camera.aspect = ww / wh;
    camera.updateProjectionMatrix();

    renderer.setSize(ww, wh);

    postFX.resize();
}
```