Title: Les fondamentaux de WebGPU 
Description: Les principes fondamentaux de WebGPU
TOC: Fondamentaux

Cet article va vous apprendre les fondamentaux de WebGPU.

<div class="warn">
Pour cet article, il est supposé que vous connaissez déjà JavaScript. Des concepts comme
<a href="https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/map">le mapping de tableaux</a>,
<a href="https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Destructuring_assignment">l'affectation par destructuration</a>,
<a href="https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Spread_syntax">la décomposition de valeurs</a>,
<a href="https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function">async/await</a>,
<a href="https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Modules">les modules es6</a>,
et plus encore seront utilisés dans ce cours. Si vous ne connaissez pas encore JavaScript et que vous voulez en apprendre davantage,vous pouvez regarder
<a href="https://javascript.info/">JavaScript.info</a>, <a href="https://eloquentjavascript.net/">Eloquent JavaScript</a>,
et/ou <a href="https://www.codecademy.com/learn/introduction-to-javascript">CodeCademy</a>.
</div>

<div class="warn">Si vous connaissez déjà WebGL, <a href="webgpu-from-webgl.html">lisez ceci</a>.</div>

WebGPU est une API qui vous laisse faire 2 choses de base.

1. [Dessiner des triangles/points/lignes dans des textures](#a-drawing-triangles-to-textures)

2. [Lancez des calculs sur le GPU](#a-run-computations-on-the-gpu)

C'est tout !

Tout ce qui concerne WebGPU après cela dépend de vous. C'est comme apprendre un langage informatique comme JavaScript, 
Rust, ou C++. 
D'abord, vous apprenez les bases, puis c'est à vous de les utiliser de manière créative pour résoudre votre problème.

WebGPU est une API extrêmement bas niveau. Bien que vous puissiez faire de petits exemples,
pour de nombreuses applications, il faudra probablement une grande quantité de code et une organisation sérieuse des données.
Par exemple, [three.js](https://threejs.org) qui supporte WebGPU consiste en ~600k de JavaScript minifié, et ce n'est que sa
librairie de base. Cela n'inclut pas les loaders, les contrôles, le post-processing, et d'autres fonctionnalités.
De la même manière, [TensorFlow avec le backend WebGPU](https://github.com/tensorflow/tfjs/tree/master/tfjs-backend-webgpu)
est ~500k de JavaScript minifié.


Dans le cas général, si vous voulez simplement afficher quelque chose à l'écran, il vaut mieux choisir une librairie 
qui fournit la grande quantité de code que vous auriez dû écrire vous-même.

Si vous avez un cas d'utilisation personnalisé, si vous voulez modifier une bibliothèque 
existante ou si vous êtes simplement curieux de savoir comment tout cela fonctionne, dans ce cas, lisez la suite !

# Pour commencer

C'est difficile de savoir où commencer.

Fondamentalement, WebGPU est un système très simple. Tout ce qu'il fait, c'est lancer 3 type de fonctions sur le GPU. 
Les Vertex Shaders, les Fragment Shaders, et les Compute Shaders.

Un Vertex Shader calculent les sommets (d'une géométrie). Le shader retourn les positions des sommets. 
Pour chaque groupe de 3 sommets, la fonction du vertex shader retourne un triangle qui est dessiné entre ses 3 positions [^primitives]

[^primitives]: Il y a en réalité 5 modes.

    * `'point-list'`: pour chaque position, dessine un point
    * `'line-list'`: pour chaque deux positions, dessine une ligne
    * `'line-strip'`: dessine des lignes connectant le nouveau point et le précédent
    * `'triangle-list'`: pour chaque 3 positions, dessine un triangle (**par défaut**)
    * `'triangle-strip'`: pour chaque nouvelle position, dessine un triangle de cette position et des deux précédentes

Un Fragment Shader calcule les couleurs [^fragment-output]. Quand un triangle est dessiné, pour dessiner chaque pixel le
GPU appelle votre fragment shader. Le fragment shader retourne une couleur.

[^fragment-output]: Les fragments shaders écrivent indirectement des données dans des textures. 
Ces données n'ont pas besoin d'être des couleurs.
Par exemple, il est courant de retourner la direction de la surface que le pixel représente.

Un Compute Shader est plus générique. C'est essentiellement une fonction que vous appelez et vous lui demandez "exécute cette fonction N fois".
Le GPU passe le numéro d'itération à chaque fois qu'il appelle votre fonction, vous pouvez donc l'utiliser pour faire quelque chose d'unique à chaque itération.

En y regardant de plus près, on peut penser que ces fonctions sont similaires aux fonctions à passer à
[`array.forEach`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/forEach)
ou
[`array.map`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/map).
Les fonctions que vous exécutez sur le GPU ne sont que des fonctions, tout comme les fonctions en JavaScript.
La partie qui diffère est qu'elles s'exécutent sur le GPU, et pour les exécuter,
vous devez copier toutes les données que vous voulez qu'elles accèdent sur le GPU sous forme de buffers (tampons) et de textures
et elles n'écrivent que dans ces buffers et textures.
Vous devez spécifier dans les fonctions les bindings (liaisons) ou les emplacements où la fonction cherchera les données. 
Et de retour en JavaScript, vous devez attacher les buffers et les textures contenant vos données aux bindings et aux bons emplacements.
Une fois que vous avez fait cela, vous dites au GPU d'exécuter la fonction.

<a id="a-draw-diagram"></a>
Peut-être qu'une image vous aidera. Voici un diagramme simplifié de la configuration de WebGPU pour dessiner des triangles 
en utilisant un vertex shader et un fragment shader.

<div class="webgpu_center"><img src="resources/webgpu-draw-diagram.svg" style="width: 960px;"></div>

Ce qu'il faut remarquer sur ce diagramme:

* Il y a un **Pipeline**. Il contient le vertex shader et le fragment shader que le GPU exécutera. 
  Vous pourriez également avoir un pipeline avec un compute shader.

* Les shaders référencent des ressources (buffers, textures, samplers) indirectement à travers les **Bind Groups**

* Le pipeline définit des attributs qui référencent des buffers indirectement à travers l'état interne

* Les attributs extraient des données et les injectent dans le vertex shader.

* Le vertex shader peut injecter des données dans le fragment shader

* Le fragment shader écrit dans des textures indirectement à travers la description de la passe de rendu

Pour exécuter des shaders sur le GPU, vous devez créer toutes ces ressources et configurer cet état. 
La création de ressources est relativement simple. 
Une chose intéressante est que la plupart des ressources WebGPU ne peuvent pas être modifiées après leur création. 
Vous pouvez modifier leur contenu mais pas leur taille, leur utilisation, leur format, etc... 
Si vous souhaitez modifier l'une de ces choses, vous devez créer une nouvelle ressource et détruire l'ancienne.


Une partie de l'état est mis en place en créant puis en exécutant des command buffers. 
Les command buffers sont littéralement ce que leur nom suggère. 
Il s'agit d'un tampon de commandes. Vous créez des encodeurs de commandes. 
Les encodeurs codent les commandes dans le command buffer. 
Vous *finissez* ensuite l'encodeur et il vous donne le command buffer qu'il a créé. 
Vous pouvez alors *soumettre* ce command buffer pour que WebGPU exécute les commandes.

Voici du pseudo-code pour encoder un tampon de commande suivi d'une représentation du command buffer qui a été créé.

<div class="webgpu_center side-by-side"><div style="min-width: 300px; max-width: 400px; flex: 1 1;"><pre class="prettyprint lang-javascript"><code>{{#escapehtml}}
encoder = device.createCommandEncoder()
// dessiner quelque chose
{
  pass = encoder.beginRenderPass(...)
  pass.setPipeline(...)
  pass.setVertexBuffer(0, …)
  pass.setVertexBuffer(1, …)
  pass.setIndexBuffer(...)
  pass.setBindGroup(0, …)
  pass.setBindGroup(1, …)
  pass.draw(...)
  pass.end()
}
// dessiner quelque chose d'autre
{
  pass = encoder.beginRenderPass(...)
  pass.setPipeline(...)
  pass.setVertexBuffer(0, …)
  pass.setBindGroup(0, …)
  pass.draw(...)
  pass.end()
}
// calculer quelque chose
{
  pass = encoder.beginComputePass(...)
  pass.beginComputePass(...)
  pass.setBindGroup(0, …)
  pass.setPipeline(...)
  pass.dispatchWorkgroups(...)
  pass.end();
}
commandBuffer = encoder.finish();
{{/escapehtml}}</code></pre></div>
<div><img src="resources/webgpu-command-buffer.svg" style="width: 300px;"></div>
</div>

Une fois que vous avez créé un command buffer, vous pouvez le *soumettre* pour qu'il soit exécuté

```js
device.queue.submit([commandBuffer]);
```

Le diagramme ci-dessus représente l'état d'une commande `draw` dans le command buffer. 
L'exécution des commandes configurera l'*état interne*, puis la commande *draw* indiquera au GPU d'exécuter un
vertex shader (et indirectement un fragment shader). 
La commande `dispatchWorkgroup` demandera au GPU d'exécuter un compute shader.

J'espère que cela vous a donné une image mentale de l'état que vous devez mettre en place. 
Comme mentionné plus haut, WebGPU peut faire 2 choses de base:

1. [Dessiner des triangles/points/lignes sur des textures](#a-drawing-triangles-to-textures)

2. [Exécuter des calculs sur le GPU](#a-run-computations-on-the-gpu)

Nous examinerons un petit exemple de ce qu'il faut faire pour chacune de ces choses.
D'autres articles montreront les différentes façons de fournir des données à ces éléments.
Notez que cet article est très basique. Nous devons construire une base de ces éléments fondamentaux.
Plus tard, nous montrerons comment les utiliser pour faire des choses que les gens font typiquement avec les GPUs
comme les graphismes 2D, 3D, etc...

# <a id="a-drawing-triangles-to-textures"></a>Dessiner des triangles dans des textures

WebGPU peut dessiner des triangles dans des [textures](webgpu-textures.html). 
Dans le cadre de cet article, une texture est un rectangle 2D de pixels.[^textures] L'élément `<canvas>`
représente une texture sur une page web. Dans WebGPU, nous pouvons demander une texture au canvas 
et effectuer le rendu dans cette texture.

[^textures]: Les textures peuvent également être des rectangles de pixels en 3D, "cube maps" (6 carrés de pixels formant un cube),
et quelques autres choses, mais les textures les plus courantes sont des rectangles de pixels en 2D.

Pour dessiner des triangles avec WebGPU nous devons fournir 2 "shaders". 
Encore une fois, les shaders sont des fonctions qui s'exécutent sur le GPU. Ces 2 shaders sont

1. Vertex Shaders

   Vertex shaders sont des fonctions qui calculent la position des sommets pour dessiner des triangles, des lignes ou des points.

2. Fragment Shaders

   Fragment shaders sont des fonctions qui calculent la couleur (ou d'autres données)
   pour chaque pixel à dessiner/rastériser lors du dessin de triangles/lignes/points

Commençons par un tout petit programme WebGPU pour dessiner un triangle.

Nous avons besoin d'un canvas pour afficher notre triangle

```html
<canvas></canvas>
```

Ensuite, nous avons besoin d'une balise `<script>` pour contenir notre JavaScript.

```html
<canvas></canvas>
+<script type="module">

... le javascript va ici ...

+</script>
```

Tout le JavaScript ci-dessous sera placé à l'intérieur de cette balise script

WebGPU est une API asynchrone, il est donc plus facile de l'utiliser dans une fonction asynchrone. 
Nous commençons par demander un adaptateur, puis un dispositif (device) à partir de l'adaptateur.

```js
async function main() {
  const adapter = await navigator.gpu?.requestAdapter();
  const device = await adapter?.requestDevice();
  if (!device) {
    fail('need a browser that supports WebGPU');
    return;
  }
}
main();
```

Le code ci-dessus est assez explicite. Tout d'abord, nous demandons un adaptateur en utilisant
[l'opérateur optionel de chaîne `?.`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Optional_chaining).
Si `navigator.gpu` n'existe pas alors `adapter` va valoir undefined.
Et s'il existe alors nous appelons `requestAdapter`. Il renvoie son résultat de manière asynchrone
donc on doit utiliser l'opérateur `await`. L'adaptateur représente un GPU spécifique. Certains appareils disposent de plusieurs GPU.
À partir de l'adaptateur, nous demandons le dispositif, mais nous utilisons à nouveau ` ?.` pour que si l'adaptateur n'est pas défini,
le dispositif ne le soit pas non plus.

Si le `device` n'est pas défini, il est probable que l'utilisateur utilise un ancien navigateur.

Ensuite, nous récupérons le canvas et créons un contexte `webgpu` pour lui. Cela va nous donner une texture dans lequel nous allons pouvoir faire le rendu.
C'est cette texture qui va être utilisée pour afficher le canvas dans la page web.


```js
  // On obtient un contexte WebGPU à partir du canvas et on le configure
  const canvas = document.querySelector('canvas');
  const context = canvas.getContext('webgpu');
  const presentationFormat = navigator.gpu.getPreferredCanvasFormat();
  context.configure({
    device,
    format: presentationFormat,
  });
```

Again, the code above is pretty self-explanatory. We get a `"webgpu"` context
from the canvas. We ask the system what the preferred canvas format is. This
will be either `"rgba8unorm"` or `"bgra8unorm"`. It's not really that important
what it is but querying it will make things faster for the user's system.

We pass that as `format` into the webgpu canvas context by calling `configure`.
We also pass in the `device` which associates this canvas with the device we just
created.

Next, we create a shader module. A shader module contains one or more shader
functions. In our case, we'll make 1 vertex shader function and 1 fragment shader
function.

```js
  const module = device.createShaderModule({
    label: 'our hardcoded red triangle shaders',
    code: `
      @vertex fn vs(
        @builtin(vertex_index) vertexIndex : u32
      ) -> @builtin(position) vec4f {
        let pos = array(
          vec2f( 0.0,  0.5),  // top center
          vec2f(-0.5, -0.5),  // bottom left
          vec2f( 0.5, -0.5)   // bottom right
        );

        return vec4f(pos[vertexIndex], 0.0, 1.0);
      }

      @fragment fn fs() -> @location(0) vec4f {
        return vec4f(1.0, 0.0, 0.0, 1.0);
      }
    `,
  });
```

Shaders are written in a language called
[WebGPU Shading Language (WGSL)](https://gpuweb.github.io/gpuweb/wgsl/) which is
often pronounced wig-sil. WGSL is a strongly typed language
which we'll try to go over in more detail in [another article](webgpu-wgsl.html).
For now, I'm hoping with a little explanation you can infer some basics.

Above we see a function called `vs` is declared with the `@vertex` attribute.
This designates it as a vertex shader function.

```wgsl
      @vertex fn vs(
        @builtin(vertex_index) vertexIndex : u32
      ) -> @builtin(position) vec4f {
         ...
```

It accepts one parameter we named `vertexIndex`. `vertexIndex` is a `u32` which
means a *32-bit unsigned integer*. It gets its value from the builtin called
`vertex_index`. `vertex_index` is like an iteration number, similar to `index` in
JavaScript's `Array.map(function(value, index) { ... })`. If we tell the GPU to
execute this function 10 times by calling `draw`, the first time `vertex_index` would be `0`, the
2nd time it would be `1`, the 3rd time it would be `2`, etc...[^indices]

[^indices]: We can also use an index buffer to specify `vertex_index`.
This is covered in [the article on vertex-buffers](webgpu-vertex-buffers.html#a-index-buffers).

Our `vs` function is declared as returning a `vec4f` which is a vector of four
32-bit floating point values. Think of it as an array of 4 values or an object
with 4 properties like `{x: 0, y: 0, z: 0, w: 0}`. This returned value will be
assigned to the `position` builtin. In "triangle-list" mode, every 3 times the
vertex shader is executed a triangle will be drawn connecting the 3 `position`
values we return.

Positions in WebGPU need to be returned in *clip space* where X goes from -1.0
on the left to +1.0 on the right, and Y goes from -1.0 at the bottom to +1.0 at the
top. This is true regardless of the size of the texture we are drawing to.

<div class="webgpu_center"><img src="resources/clipspace.svg" style="width: 500px"></div>

The `vs` function declares an array of 3 `vec2f`s. Each `vec2f` consists of two
32-bit floating point values.

```wgsl
        let pos = array(
          vec2f( 0.0,  0.5),  // top center
          vec2f(-0.5, -0.5),  // bottom left
          vec2f( 0.5, -0.5)   // bottom right
        );
```

Finally it uses `vertexIndex` to return one of the 3 values from the array.
Since the function requires 4 floating point values for its return type, and
since `pos` is an array of `vec2f`, the code supplies `0.0` and `1.0` for
the remaining 2 values.

```wgsl
        return vec4f(pos[vertexIndex], 0.0, 1.0);
```

The shader module also declares a function called `fs` that is declared with
`@fragment` attribute making it a fragment shader function.

```wgsl
      @fragment fn fs() -> @location(0) vec4f {
```

This function takes no parameters and returns a `vec4f` at `location(0)`.
This means it will write to the first render target. We'll make the first
render target our canvas texture later.

```wgsl
        return vec4f(1, 0, 0, 1);
```

The code returns `1, 0, 0, 1` which is red. Colors in WebGPU are usually
specified as floating point values from `0.0` to `1.0` where the 4 values above
correspond to red, green, blue, and alpha respectively.

When the GPU rasterizes the triangle (draws it with pixels), it will call
the fragment shader to find out what color to make each pixel. In our case,
we're just returning red.

One more thing to note is the `label`. Nearly every object you can create with
WebGPU can take a `label`. Labels are entirely optional but it's considered
*best practice* to label everything you make. The reason is that when you get an
error, most WebGPU implementations will print an error message that includes the
labels of the things related to the error.

In a normal app, you'd have 100s or 1000s of buffers, textures, shader modules,
pipelines, etc... If you get an error like `"WGSL syntax error in shaderModule
at line 10"`, if you have 100 shader modules, which one got the error? If you
label the module then you'll get an error more like `"WGSL syntax error in
shaderModule('our hardcoded red triangle shaders') at line 10` which is a way
more useful error message and will save you a ton of time tracking down the
issue.

Now that we've created a shader module, we next need to make a render pipeline

```js
  const pipeline = device.createRenderPipeline({
    label: 'our hardcoded red triangle pipeline',
    layout: 'auto',
    vertex: {
      entryPoint: 'vs',
      module,
    },
    fragment: {
      entryPoint: 'fs',
      module,
      targets: [{ format: presentationFormat }],
    },
  });
```

In this case, there isn't much to see. We set `layout` to `'auto'` which means
to ask WebGPU to derive the layout of data from the shaders. We're not using
any data though.

We then tell the render pipeline to use the `vs` function from our shader module
for a vertex shader and the `fs` function for our fragment shader. Otherwise, we
tell it the format of the first render target. "render target" means the texture
we will render to. When we create a pipeline
we have to specify the format for the texture(s) we'll use this pipeline to
eventually render to.

Element 0 for the `targets` array corresponds to location 0 as we specified for
the fragment shader's return value. Later, we'll set that target to be a texture
for the canvas.

One shortcut, for each shader stage, `vertex` and `fragment`, if there is only one
function of the corresponding type then we don't need to specify the `entryPoint`.
WebGPU will use the sole function that matches the shader stage. So we can shorten
the code above to

```js
  const pipeline = device.createRenderPipeline({
    label: 'our hardcoded red triangle pipeline',
    layout: 'auto',
    vertex: {
-      entryPoint: 'vs',
      module,
    },
    fragment: {
-      entryPoint: 'fs',
      module,
      targets: [{ format: presentationFormat }],
    },
  });
```

Next up we prepare a `GPURenderPassDescriptor` which describes which textures
we want to draw to and how to use them.

```js
  const renderPassDescriptor = {
    label: 'our basic canvas renderPass',
    colorAttachments: [
      {
        // view: <- to be filled out when we render
        clearValue: [0.3, 0.3, 0.3, 1],
        loadOp: 'clear',
        storeOp: 'store',
      },
    ],
  };  
```

A `GPURenderPassDescriptor` has an array for `colorAttachments` which lists
the textures we will render to and how to treat them.
We'll wait to fill in which texture we actually want to render to. For now,
we set up a clear value of semi-dark gray, and a `loadOp` and `storeOp`.
`loadOp: 'clear'` specifies to clear the texture to the clear value before
drawing. The other option is `'load'` which means load the existing contents of
the texture into the GPU so we can draw over what's already there. 
`storeOp: 'store'` means store the result of what we draw. We could also pass `'discard'`
which would throw away what we draw. We'll cover why we might want to do that in
[another article](webgpu-multisampling.html).

Now it's time to render. 

```js
  function render() {
    // Get the current texture from the canvas context and
    // set it as the texture to render to.
    renderPassDescriptor.colorAttachments[0].view =
        context.getCurrentTexture().createView();

    // make a command encoder to start encoding commands
    const encoder = device.createCommandEncoder({ label: 'our encoder' });

    // make a render pass encoder to encode render specific commands
    const pass = encoder.beginRenderPass(renderPassDescriptor);
    pass.setPipeline(pipeline);
    pass.draw(3);  // call our vertex shader 3 times
    pass.end();

    const commandBuffer = encoder.finish();
    device.queue.submit([commandBuffer]);
  }

  render();
```

First, we call `context.getCurrentTexture()` to get a texture that will appear in the
canvas. Calling `createView` gets a view into a specific part of a texture but
with no parameters, it will return the default part which is what we want in this
case. For now, our only `colorAttachment` is a texture view from our
canvas which we get via the context we created at the start. Again, element 0 of
the `colorAttachments` array corresponds to `@location(0)` as we specified for
the return value of the fragment shader.

Next, we create a command encoder. A command encoder is used to create a command
buffer. We use it to encode commands and then "submit" the command buffer it
created to have the commands executed.

We then use the command encoder to create a render pass encoder by calling `beginRenderPass`. A render
pass encoder is a specific encoder for creating commands related to rendering.
We pass it our `renderPassDescriptor` to tell it which texture we want to
render to.

We encode the command, `setPipeline`, to set our pipeline and then tell it to
execute our vertex shader 3 times by calling `draw` with 3. By default, every 3
times our vertex shader is executed a triangle will be drawn by connecting the 3
values just returned from the vertex shader.

We end the render pass, and then finish the encoder. This gives us a
command buffer that represents the steps we just specified. Finally, we submit
the command buffer to be executed.

When the `draw` command is executed, this will be our state.

<div class="webgpu_center"><img src="resources/webgpu-simple-triangle-diagram.svg" style="width: 723px;"></div>

We've got no textures, no buffers, no bindGroups but we do have a pipeline, a
vertex and fragment shader, and a render pass descriptor that tells our shader
to render to the canvas texture.

The result.

{{{example url="../webgpu-simple-triangle.html"}}}

It's important to emphasize that all of these functions we called
like `setPipeline`, and `draw` only add commands to a command buffer.
They don't actually execute the commands. The commands are executed
when we submit the command buffer to the device queue.

<a id="a-rasterization"></a>WebGPU takes every 3 vertices we return from our vertex shader and uses
them to rasterize a triangle. It does this by determining which pixels'
centers are inside the triangle. It then calls our fragment shader for
each pixel to ask what color to make it.

Imagine the texture we are rendering
to was 15x11 pixels. These are the pixels that would be drawn to

<div class="webgpu_center">
  <div data-diagram="clip-space-to-texels" style="display: inline-block; max-width: 500px; width: 100%"></div>
  <div>drag the vertices</div>
</div>

So, now we've seen a very small working WebGPU example. It should be pretty
obvious that hard coding a triangle inside a shader is not very flexible. We
need ways to provide data and we'll cover those in the following articles. The
points to take away from the code above,

* WebGPU just runs shaders. It's up to you to fill them with code to do useful things
* Shaders are specified in a shader module and then turned into a pipeline
* WebGPU can draw triangles
* WebGPU draws to textures (we happened to get a texture from the canvas)
* WebGPU works by encoding commands and then submitting them.

# <a id="a-run-computations-on-the-gpu"></a>Run computations on the GPU

Let's write a basic example for doing some computation on the GPU.

We start off with the same code to get a WebGPU device.

```js
async function main() {
  const adapter = await navigator.gpu?.requestAdapter();
  const device = await adapter?.requestDevice();
  if (!device) {
    fail('need a browser that supports WebGPU');
    return;
  }
```

Then we create a shader module.

```js
  const module = device.createShaderModule({
    label: 'doubling compute module',
    code: `
      @group(0) @binding(0) var<storage, read_write> data: array<f32>;

      @compute @workgroup_size(1) fn computeSomething(
        @builtin(global_invocation_id) id: vec3u
      ) {
        let i = id.x;
        data[i] = data[i] * 2.0;
      }
    `,
  });
```

First, we declare a variable called `data` of type `storage` that we want to be
able to both read from and write to.

```wgsl
      @group(0) @binding(0) var<storage, read_write> data: array<f32>;
```

We declare its type as `array<f32>` which means an array of 32-bit floating point
values. We tell it we're going to specify this array on binding location 0 (the
`binding(0)`) in bindGroup 0 (the `@group(0)`).

Then we declare a function called `computeSomething` with the `@compute`
attribute which makes it a compute shader. 

```wgsl
      @compute @workgroup_size(1) fn computeSomething(
        @builtin(global_invocation_id) id: vec3u
      ) {
        ...
```

Compute shaders are required to declare a workgroup size which we will cover
later. For now, we'll just set it to 1 with the attribute `@workgroup_size(1)`.
We declare it to have one parameter `id` which uses a `vec3u`. A `vec3u` is
three unsigned 32-bit integer values. Like our vertex shader above, this is the
iteration number. It's different in that compute shader iteration numbers are 3
dimensional (have 3 values). We declare `id` to get its value from the built-in
`global_invocation_id`.

You can *kind of* think of compute shaders as running like this. This is an over
simplification but it will do for now.

```js
// pseudo code
function dispatchWorkgroups(width, height, depth) {
  for (z = 0; z < depth; ++z) {
    for (y = 0; y < height; ++y) {
      for (x = 0; x < width; ++x) {
        const workgroup_id = {x, y, z};
        dispatchWorkgroup(workgroup_id)
      }
    }
  }
}

function dispatchWorkgroup(workgroup_id) {
  // from @workgroup_size in WGSL
  const workgroup_size = shaderCode.workgroup_size;
  const {x: width, y: height, z: depth} = workgroup_size;
  for (z = 0; z < depth; ++z) {
    for (y = 0; y < height; ++y) {
      for (x = 0; x < width; ++x) {
        const local_invocation_id = {x, y, z};
        const global_invocation_id =
            workgroup_id * workgroup_size + local_invocation_id;
        computeShader(global_invocation_id)
      }
    }
  }
}
```

Since we set `@workgroup_size(1)`, effectively the pseudo-code above becomes

```js
// pseudo code
function dispatchWorkgroups(width, height, depth) {
  for (z = 0; z < depth; ++z) {
    for (y = 0; y < height; ++y) {
      for (x = 0; x < width; ++x) {
        const workgroup_id = {x, y, z};
        dispatchWorkgroup(workgroup_id)
      }
    }
  }
}

function dispatchWorkgroup(workgroup_id) {
  const global_invocation_id = workgroup_id;
  computeShader(global_invocation_id)
}
```

Finally, we use the `x` property of `id` to index `data` and multiply each value
by 2.

```wgsl
        let i = id.x;
        data[i] = data[i] * 2.0;
```

Above, `i` is just the first of the 3 iteration numbers.

Now that we've created the shader, we need to create a pipeline.

```js
  const pipeline = device.createComputePipeline({
    label: 'doubling compute pipeline',
    layout: 'auto',
    compute: {
      module,
    },
  });
```

Here we just tell it we're using a `compute` stage from the shader `module` we
created and since there is only one `@compute` entry point WebGPU knows we want to call it. `layout` is
`'auto'` again, telling WebGPU to figure out the layout from the shaders. [^layout-auto]

[^layout-auto]: `layout: 'auto'` is convenient but it's impossible to share bind groups
across pipelines using `layout: 'auto'`. Most of the examples on this site
never use a bind group with multiple pipelines. We'll cover explicit layouts in [another article](webgpu-drawing-multiple-things.html).

Next, we need some data.

```js
  const input = new Float32Array([1, 3, 5]);
```

That data only exists in JavaScript. For WebGPU to use it, we need to make a
buffer that exists on the GPU and copy the data to the buffer.

```js
  // create a buffer on the GPU to hold our computation
  // input and output
  const workBuffer = device.createBuffer({
    label: 'work buffer',
    size: input.byteLength,
    usage: GPUBufferUsage.STORAGE | GPUBufferUsage.COPY_SRC | GPUBufferUsage.COPY_DST,
  });
  // Copy our input data to that buffer
  device.queue.writeBuffer(workBuffer, 0, input);
```

Above, we call `device.createBuffer` to create a buffer. `size` is the size in
bytes. In this case, it will be 12 because the size in bytes of a `Float32Array` of 3
values is 12. If you're not familiar with `Float32Array` and typed arrays then
see [this article](webgpu-memory-layout.html).

Every WebGPU buffer we create has to specify a `usage`. There are a bunch of
flags we can pass for usage but not all of them can be used together. Here we
say we want this buffer to be usable as `storage` by passing
`GPUBufferUsage.STORAGE`. This makes it compatible with `var<storage,...>` from
the shader. Further, we want to be able to copy data to this buffer so we include
the `GPUBufferUsage.COPY_DST` flag. And finally, we want to be able to copy data
from the buffer so we include `GPUBufferUsage.COPY_SRC`.

Note that you can not directly read the contents of a WebGPU buffer from
JavaScript. Instead, you have to "map" it which is another way of requesting
access to the buffer from WebGPU because the buffer might be in use and because
it might only exist on the GPU.

WebGPU buffers that can be mapped in JavaScript can't be used for much else. In
other words, we can not map the buffer we just created above and if we try to add
the flag to make it mappable, we'll get an error that it is not compatible with
usage `STORAGE`.

So, in order to see the result of our computation, we'll need another buffer.
After running the computation, we'll copy the buffer above to this result buffer
and set its flags so we can map it.

```js
  // create a buffer on the GPU to get a copy of the results
  const resultBuffer = device.createBuffer({
    label: 'result buffer',
    size: input.byteLength,
    usage: GPUBufferUsage.MAP_READ | GPUBufferUsage.COPY_DST
  });
```

`MAP_READ` means we want to be able to map this buffer for reading data.

In order to tell our shader about the buffer we want it to work on, we need to
create a bindGroup.

```js
  // Setup a bindGroup to tell the shader which
  // buffer to use for the computation
  const bindGroup = device.createBindGroup({
    label: 'bindGroup for work buffer',
    layout: pipeline.getBindGroupLayout(0),
    entries: [
      { binding: 0, resource: { buffer: workBuffer } },
    ],
  });
```

We get the layout for the bindGroup from the pipeline. Then we set up bindGroup
entries. The 0 in `pipeline.getBindGroupLayout(0)` corresponds to the
`@group(0)` in the shader. The `{binding: 0 ...` of the `entries` corresponds to
the `@group(0) @binding(0)` in the shader.

Now we can start encoding commands.

```js
  // Encode commands to do the computation
  const encoder = device.createCommandEncoder({
    label: 'doubling encoder',
  });
  const pass = encoder.beginComputePass({
    label: 'doubling compute pass',
  });
  pass.setPipeline(pipeline);
  pass.setBindGroup(0, bindGroup);
  pass.dispatchWorkgroups(input.length);
  pass.end();
```

We create a command encoder. We start a compute pass. We set the pipeline, then
we set the bindGroup. Here, the `0` in `pass.setBindGroup(0, bindGroup)`
corresponds to `@group(0)` in the shader. We then call `dispatchWorkgroups` and in
this case, we pass it `input.length` which is `3` telling WebGPU to run the
compute shader 3 times. We then end the pass.

Here's what the situation will be when `dispatchWorkgroups` is executed.

<div class="webgpu_center"><img src="resources/webgpu-simple-compute-diagram.svg" style="width: 553px;"></div>

After the computation is finished we ask WebGPU to copy from `workBuffer` to
`resultBuffer`.

```js
  // Encode a command to copy the results to a mappable buffer.
  encoder.copyBufferToBuffer(workBuffer, 0, resultBuffer, 0, resultBuffer.size);
```

Now we can `finish` the encoder to get a command buffer and then submit that
command buffer.

```js
  // Finish encoding and submit the commands
  const commandBuffer = encoder.finish();
  device.queue.submit([commandBuffer]);
```

We then map the results buffer and get a copy of the data.

```js
  // Read the results
  await resultBuffer.mapAsync(GPUMapMode.READ);
  const result = new Float32Array(resultBuffer.getMappedRange());

  console.log('input', input);
  console.log('result', result);

  resultBuffer.unmap();
```

To map the results buffer, we call `mapAsync` and have to `await` for it to
finish. Once mapped, we can call `resultBuffer.getMappedRange()` which with no
parameters will return an `ArrayBuffer` of the entire buffer. We put that in a
`Float32Array` typed array view and then we can look at the values. One
important detail, the `ArrayBuffer` returned by `getMappedRange` is only valid
until we call `unmap`. After `unmap`, its length will be set to 0 and its data
no longer accessible.

Running that we can see we got the result back, all the numbers have been
doubled.

{{{example url="../webgpu-simple-compute.html"}}}

We'll cover how to really use compute shaders in other articles. For now, you
hopefully have gleaned some understanding of what WebGPU does. EVERYTHING ELSE
IS UP TO YOU! Think of WebGPU as similar to other programming languages. It
provides a few basic features and leaves the rest to your creativity.

What makes WebGPU programming special is these functions, vertex shaders,
fragment shaders, and compute shaders, run on your GPU. A GPU could have over
10000 processors which means they can potentially do more than 10000
calculations in parallel which is likely 3 or more orders of magnitude than your
CPU can do in parallel.

## <a id="a-resizing"></a> Simple Canvas Resizing

Before we move on, let's go back to our triangle drawing example and add some
basic support for resizing a canvas. Sizing a canvas is actually a topic that
can have many subtleties so [there is an entire article on it](webgpu-resizing-the-canvas.html).
For now though let's just add some basic support.

First, we'll add some CSS to make our canvas fill the page.

```html
<style>
html, body {
  margin: 0;       /* remove the default margin          */
  height: 100%;    /* make the html,body fill the page   */
}
canvas {
  display: block;  /* make the canvas act like a block   */
  width: 100%;     /* make the canvas fill its container */
  height: 100%;
}
</style>
```

That CSS alone will make the canvas get displayed to cover the page but it won't change
the resolution of the canvas itself so you might notice, if you make the example below
large, like if you click the full-screen button, you'll see the edges of the triangle
are blocky.

{{{example url="../webgpu-simple-triangle-with-canvas-css.html"}}}

`<canvas>` tags, by default, have a resolution of 300x150 pixels. We'd like to
adjust the resolution of the canvas to match the size it is displayed.
One good way to do this is with a `ResizeObserver`. You create a
`ResizeObserver` and give it a function to call whenever the elements you've
asked it to observe change their size. You then tell it which elements to
observe.

```js
    ...
-    render();

+    const observer = new ResizeObserver(entries => {
+      for (const entry of entries) {
+        const canvas = entry.target;
+        const width = entry.contentBoxSize[0].inlineSize;
+        const height = entry.contentBoxSize[0].blockSize;
+        canvas.width = Math.max(1, Math.min(width, device.limits.maxTextureDimension2D));
+        canvas.height = Math.max(1, Math.min(height, device.limits.maxTextureDimension2D));
+        // re-render
+        render();
+      }
+    });
+    observer.observe(canvas);
```

In the code above, we go over all the entries but there should only ever be one
because we're only observing our canvas. We need to limit the size of the canvas
to the largest size our device supports otherwise WebGPU will start generating
errors that we tried to make a texture that is too large. We also need to make
sure it doesn't go to zero or again we'll get errors. 
[See the longer article for details](webgpu-resizing-the-canvas.html).

We call `render` to re-render the
triangle at the new resolution. We removed the old call to `render` because
it's not needed. A `ResizeObserver` will always call its callback at least once
to report the size of the elements when they started being observed.

The new size texture is created when we call `context.getCurrentTexture()` 
inside `render` so there's nothing left to do.

{{{example url="../webgpu-simple-triangle-with-canvas-resize.html"}}}

In the following articles, we'll cover various ways to pass data into shaders.

* [inter-stage variables](webgpu-inter-stage-variables.html)
* [uniforms](webgpu-uniforms.html)
* [storage buffers](webgpu-storage-buffers.html)
* [vertex buffers](webgpu-vertex-buffers.html)
* [textures](webgpu-textures.html)
* [constants](webgpu-constants.html)

Then we'll cover [the basics of WGSL](webgpu-wgsl.html).

This order is from the simplest to the most complex. Inter-stage variables
require no external setup to explain. We can see how to use them using nothing
but changes to the WGSL we used above. Uniforms are effectively global variables
and as such are used in all 3 kinds of shaders (vertex, fragment, and compute).
Going from uniform buffers to storage buffers is trivial as shown at the top of
the article on storage buffers. Vertex buffers are only used in vertex shaders.
They are more complex because they require describing the data layout to WebGPU.
Textures are the most complex as they have tons of types and options.

I'm a little bit worried these articles will be boring at first. Feel free to
jump around if you'd like. Just remember if you don't understand something you
probably need to read or review these basics. Once we get the basics down, we'll
start going over actual techniques.

One other thing. All of the example programs can be edited live in the webpage.
Further, they can all easily be exported to [jsfiddle](https://jsfiddle.net) and [codepen](https://codepen.io)
and even [stackoverflow](https://stackoverflow.com). Just click "Export".

<div class="webgpu_bottombar">
<p>
The code above gets a WebGPU device in a very terse way. A more verbose
way would be something like
</p>
<pre class="prettyprint showmods">{{#escapehtml}}
async function start() {
  if (!navigator.gpu) {
    fail('this browser does not support WebGPU');
    return;
  }

  const adapter = await navigator.gpu.requestAdapter();
  if (!adapter) {
    fail('this browser supports webgpu but it appears disabled');
    return;
  }

  const device = await adapter?.requestDevice();
  device.lost.then((info) => {
    console.error(`WebGPU device was lost: ${info.message}`);

    // 'reason' will be 'destroyed' if we intentionally destroy the device.
    if (info.reason !== 'destroyed') {
      // try again
      start();
    }
  });
  
  main(device);
}
start();

function main(device) {
  ... do webgpu ...
}
{{/escapehtml}}</pre>
<p>
<code>device.lost</code> is a promise that starts off unresolved. It will resolve if and when the
device is lost. A device can be lost for many reasons. Maybe the user ran a really intensive
app and it crashed their GPU. Maybe the user updated their drivers. Maybe the user has
an external GPU and unplugged it. Maybe another page used a lot of GPU, your
tab was in the background and the browser decided to free up some memory by
losing the device for background tabs. The point to take away is that for any serious
apps you probably want to handle losing the device.
</p>
<p>
Note that <code>requestDevice</code> always returns a device. It just might start lost.
WebGPU is designed so that, for the most part, the device will appear to work,
at least from an API level. Calls to create things and use them will appear
to succeed but they won't actually function. It's up to you to take action
when the <code>lost</code> promise resolves.
</p>
</div>

<!-- keep this at the bottom of the article -->
<script type="module" src="webgpu-fundamentals.js"></script>
