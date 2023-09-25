<!-- Notas: Usaré las palabras "shader" en vez de "sombreado", "pipeline" en vez de "canalización" y "array" en vez de "vector", "target" en vez de "objetivo" -->

<!-- TODO: Meta -->
Title: Fundamentos de WebGPU
Description: Los conceptos más básicos de WebGPU
TOC: Fundamentals

Este artículo tratará de explicarte los conceptos más básicos de WebGPU!

<div class="warn">
Se da por hecho que ya conoces JavaScript antes de leer este artículo. Conceptos como
<!-- TODO: asegurar que 'mapeado de arrays' es correcto -->
<a href="https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/map">mapeado de arrays</a>,
<a href="https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Destructuring_assignment">desestructuración</a>,
<a href="https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Spread_syntax">sintaxis extendida</a>,
<a href="https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function">métodos asíncronos</a> o
<a href="https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Modules">módulos de ES6</a>
serán usados con frecuencia.
</div>

<div class="warn">Si ya sabes cómo funciona WebGL, <a href="webgpu-from-webgl.html">lee esto</a>.</div>

WebGPU es una API que permite hacer dos cosas:

1. [Dibujar triángulos, puntos y líneas en texturas](#a-drawing-triangles-to-textures)

2. [Correr código en una GPU](#a-run-computations-on-the-gpu)

Nada más!

El resto va a depender de ti. Lo mismo sucede al aprender un nuevo lenguaje de programación
como JavaScript, Rust o C++. Comienzas por entender los conceptos básicos, y después está
en tus manos aplicarlos para solucionar problemas.

WebGPU es una API de bajo nivel. Aunque puedas crear pequeños ejemplos, será habitual
acabar con proyectos que requieran mucho código y formas rigurosas de organizar datos.
Como ejemplo, existe [three.js](https://threejs.org), una librería muy popular que hace uso de WebGPU.
Tras ser minificada, su tamaño ronda los 600 kilobytes, incluyendo únicamente las partes esenciales y
dejando de lado funcionalidades como loaders, controladores o post procesado.
Sucede igual con [TensorFlow usando WebGPU en el backend](https://github.com/tensorflow/tfjs/tree/master/tfjs-backend-webgpu),
pesando alrededor de 500 kilobytes.

Por tanto, si tu objetivo es mostrar algo en pantalla es más recomendable utilizar una librería
que ya proporcione esa gran cantidad de código, en lugar de tener que escribirlo por ti mismo.

Sin embargo, es posible que requieras una solución personalizada cuando las librerías no te sirvan.
También puede ser que desees realizar cambios en alguna de estas librerías o simplemente tengas
curiosidad acerca del funcionamiento de WebGPU. Si te encuentras en cualquiera de estas situaciones,
¡continúa leyendo!

# Primeros pasos

Es complicado decidir dónde empezar. En cierta forma WebGPU es una herramienta sencilla,
todo lo que hace es correr 3 tipos de métodos:
Vertex Shaders, Fragment Shaders y Compute Shaders.

Un Vertex Shader calcula vértices, devolviendo sus posiciones. Para cada grupo de 3 vertices
se devuelve un triangulo formado por esas 3 posiciones [^primitives]

<!-- TODO: comprueba que esto está bien descrito! -->
[^primitives]: Realmente hay 5 modos:

    * `'point-list'`: dibuja un punto por cada posición
    * `'line-list'`: dibuja una linea por cada 2 posiciones
    * `'line-strip'`: dibuja lineas que conecten cada nuevo punto con su anterior
    * `'triangle-list'`: dibuja un triangulo por cada 3 posiciones (**usado por defecto**)
    * `'triangle-strip'`: dibuja un triangulo para cada nueva posición y sus 2 últimas posiciones

Un Fragment Shader calcula colores [^fragment-output]. La GPU llama al Fragment Shader en cada píxel
comprendido por el triangulo calculado en el Vertex Shader, cada llamada al Fragment Shader devuelve
el color que le corresponde a ese píxel.

<!-- TODO: repasa esta descripción -->
[^fragment-output]: Los Fragment Shader escriben datos en texturas. Estos datos pueden ser más cosas
a parte de colores. Un ejemplo es la dirección de la superficie que el pixel representa.

Un Compute Shader es un método genérico. A diferencia de los dos anteriores, este shader corre la función que
queramos tantas veces como sea necesario. Para cada llamada la GPU asigna un índice de iteración, de esta forma
es posible hacer algo distinto cada vez.

Podemos imaginar los Shaders como algo similar a las funciones
[`array.forEach`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/forEach)
o
[`array.map`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/map)
que usamos en JavaScript.
La diferencia es que corren en la GPU, mientras que en JavaScript utilizamos la CPU. Por este motivo,
necesitamos copiar todos los datos necesarios en forma de búfers y texturas, que luego retornarán datos
exclusivamente a esos mismos búfers y texturas.
Por un lado es necesario definir en la función los Bindings y Locations que se utilizarán para acceder a los datos.
Por el otro, en JavaScript se definen las relaciones entre los búfers y texturas con los Bindings y Locations.
Finalmente se pide a la GPU ejecutar la función.

<a id="a-draw-diagram"></a>Quizás una imagen ayude a entenderlo mejor. El siguiente diagrama muestra de forma
simplificada la configuración para dibujar un triangulo mediante un vertex y fragment shader.

<div class="webgpu_center"><img src="resources/webgpu-draw-diagram.svg" style="width: 960px;"></div>

Cosas que destacar sobre el diagrama:

* Hay un **Pipeline**. Contiene el vertex y fragment shader que correrá la GPU.
  Es posible tener también un pipeline con compute shaders dentro.

<!-- TODO: indirectamente? -->
* Los recursos utilizados en los shaders como búfers, texturas o samplers se
  referencian a través de **Bind Groups**

<!-- TODO: indirectamente? -->
* El pipeline define atributos que referencian búfers a través del estado interno.

* Los atributos obtienen los datos mediante los búfers, luego los facilita al vertex shader.

* El vertex shader podría pasar datos al fragment shader.

<!-- TODO: indirectamente? descriptores de render pass? -->
* El fragment shader escribe en texturas a través de descriptores de render pass.

Al ejecutar shaders en una GPU, es necesario inicializar todos los recursos y configurar el estado tal y como aparece en el diagrama. El proceso es relativamente sencillo.
Algo a destacar sobre los recursos en WebGPU es que puedas cambiar su contenido, pero no modificar el tamaño, uso, formato y otras propiedades. Para hacerlo no queda otra que eliminar y volver a crear esos recursos.

Parte del estado se prepara creando y ejecutando búfers de comandos.
Son literalmente búfers que contienen comandos, el nombre no tiene pérdida.
Primero se crean los codificadores, estos codificadores codifican los comandos al nuevo búfer. Al **acabar** devolverán el búfer listo, luego podrá ser **enviado** para que WebGPU corra los comandos que contiene.

El siguiente ejemplo muestra en seudocódigo cómo se codifica un búfer de comandos, a su lado se representa qué aspecto tendría.

<div class="webgpu_center side-by-side"><div style="min-width: 300px; max-width: 400px; flex: 1 1;"><pre class="prettyprint lang-javascript"><code>{{#escapehtml}}
encoder = device.createCommandEncoder()
// draw something
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
// draw something else
{
  pass = encoder.beginRenderPass(...)
  pass.setPipeline(...)
  pass.setVertexBuffer(0, …)
  pass.setBindGroup(0, …)
  pass.draw(...)
  pass.end()
}
// compute something
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

Una vez que hayas creado un búfer de comandos en WebGPU, el siguiente paso es *enviarlo* para su ejecución

```js
device.submit([commandBuffer]);
```

El diagrama de encima representa el estado de un comando "draw" dentro del búfer
de comandos. Al ejecutar estos comandos, se configura el *estado interno* de la GPU.
En concreto, el comando "draw" instruye a la GPU a ejecutar un sombreador de vértices
y, de forma indirecta, un sombreador de fragmentos. 
Por otro lado, el comando `dispatchWorkgroup` se utiliza para indicar a la GPU que ejecute
un sombreador de cómputo.

Espero que eso te haya dado una imagen mental del estado que necesitas configurar.
Como se mencionó anteriormente, WebGPU tiene 2 cosas básicas que puede hacer:

1. [Dibujar triángulos/puntos/líneas en texturas](#a-drawing-triangles-to-textures)

2. [Ejecutar cálculos en la GPU](#a-run-computations-on-the-gpu)

Exploraremos un pequeño ejemplo de cómo hacer cada una de esas cosas.
Otros artículos mostrarán las diversas formas de proporcionar datos a estas funciones.
Ten en cuenta que esto será muy básico. Necesitamos construir una base con estos
conceptos fundamentales. Más adelante, mostraremos cómo utilizarlos para realizar tareas
que las personas suelen hacer con las GPUs, como gráficos 2D, gráficos 3D, etc.

# <a id="a-drawing-triangles-to-textures"></a>Dibujando triángulos en texturas

WebGPU puede dibujar triángulos en [texturas](webgpu-textures.html). Para el propósito de
este artículo, una textura es un rectángulo 2D de píxeles.[^textures] El elemento `<canvas>`
se utiliza para representar una textura en una página web. En WebGPU, podemos solicitar al lienzo (`<canvas>`) la textura para luego renderizar en ella.

[^textures]: Las texturas también pueden ser rectángulos 3D de píxeles, mapas de cubos (6 cuadrados de píxeles que forman un cubo) y algunas otras cosas, pero las texturas más comunes son rectángulos 2D de píxeles.

Para dibujar triángulos con WebGPU, debemos proporcionar 2 sombreadores. Una vez más, los sombreadores son funciones que se ejecutan en la GPU. Estos 2 sombreadores son:

1. Sombreadores de Vértices

    Los sombreadores de vértices son funciones que calculan las posiciones de los vértices
    para dibujar triángulos/líneas/puntos.

2. Sombreadores de Fragmentos

    Los sombreadores de fragmentos son funciones que calculan el color (u otros datos) para
    cada píxel que se va a dibujar/rasterizar al dibujar triángulos/líneas/puntos.

Comencemos con un pequeño programa de WebGPU para dibujar un triángulo.

Necesitamos un lienzo para mostrar nuestro triángulo.

```html
<canvas></canvas>
```

También una etiqueta `<script>` para contener nuestro JavaScript.

```html
<canvas></canvas>
+<script type="module">

... el código de javascript se escribirá aquí ...

+</script>
```

Todo el código JavaScript se ubicará dentro de la etiqueta `<script>`.

WebGPU es una API asíncrona, por lo que será más cómodo usarla dentro de una `async function`.
Comenzamos solicitando un adaptador (adapter) y luego solicitamos un dispositivo (device) del adaptador.

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

El código anterior es bastante autoexplicativo. Primero solicitamos un adaptador utilizando el
[`?.` operador de encadenamiento opcional](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Optional_chaining).
De esta forma, si `navigator.gpu` resulta ser undefined, entonces `adapter` también lo será. Sin el
operador `?.` intentaríamos acceder a una función que no existe.
Si existe, llamaremos a `requestAdapter`. Este devuelve sus resultados de forma asíncrona, por lo que necesitamos utilizar `await`. El adaptador representa una GPU específica. Algunos dispositivos tienen múltiples GPUs.

A través del adaptador solicitamos el dispositivo utilizando `?.` una vez más, para que si el adaptador resulta ser indefinido, el dispositivo también lo será.

Si no se encontrase ningún dispositivo, es bastante probable que el usuario esté usando un navegador antiguo.

Next up we look up the canvas and create a `webgpu` context for it. This will
let us get a texture to render to that will be used to render the canvas in the
webpage.

El siguiente paso será buscar el elemento canvas y crear un contexto `webgpu` para él. Esto nos permitirá obtener una textura en la que renderizar.

```js
  // Get a WebGPU context from the canvas and configure it
  const canvas = document.querySelector('canvas');
  const context = canvas.getContext('webgpu');
  const presentationFormat = navigator.gpu.getPreferredCanvasFormat();
  context.configure({
    device,
    format: presentationFormat,
  });
```

Una vez más, el código es bastante autoexplicativo. Obtenemos un contexto
`"webgpu"` a partir del elemento canvas. Definimos cuál es el formato preferido de 
lienzo, que será `"rgba8unorm"` o `"bgra8unorm"`. No importa demasiado cuál es,
pero consultarlo optimizará el proceso.

Pasamos el valor retornado como `format` al contexto del lienzo de webgpu llamando a `configure`.
También pasamos el `device`, que asocia este lienzo con el dispositivo que acabamos de crear.

A continuación, creamos un módulo de sombreador (shader module). Un módulo de sombreador
contiene una o más funciones de sombreador. En nuestro caso, crearemos una función de
sombreador de vértices y una función de sombreador de fragmentos.

```js
  const module = device.createShaderModule({
    label: 'código para los sombreadores del triángulo rojo',
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

Los shaders se escriben en un lenguaje llamado [WebGPU Shading Language  (WGSL)](https://gpuweb.github.io/gpuweb/wgsl/). WGSL es un lenguaje tipado del cual intentaremos abordar más detalles en [otro artículo](webgpu-wgsl.html). Por ahora, con una breve explicación mostraremos algunos conceptos básicos.

En el fragmento anterior, podemos ver una función llamada `vs` que se declara con el atributo `@vertex`. Declararla así la designa como una función de sombreado de vértices.

```wgsl
      @vertex fn vs(
        @builtin(vertex_index) vertexIndex : u32
      ) -> @builtin(position) vec4f {
         ...
```

Esta función acepta un parámetro que hemos llamado `vertexIndex`. `vertexIndex` es un `u32`, lo que significa un *entero sin signo de 32 bits*. Obtiene su valor del elemento incorporado (@builtin) llamado `vertex_index`. `vertex_index` es similar a un número de iteración, muy parecido a `index` en la función `Array.map` de JavaScript. Si le decimos a la GPU que ejecute esta función 10 veces llamando a `draw`, la primera vez que se ejecute, `vertex_index` sería `0`, la segunda vez sería `1`, la tercera vez sería `2`, y así sucesivamente...[^indices]

[^indices]: También podemos usar un búfer de índices para especificar el `índice_de_vertice`.
Esto se explica en [el artículo sobre búfers de vértices](webgpu-vertex-buffers.html#a-index-buffers).

Nuestra función `vs` se declara devolviendo un `vec4f`, que es un vector de cuatro
valores de punto flotante de 32 bits. Piénsalo como un vector de 4 valores o un objeto
con 4 propiedades como `{x: 0, y: 0, z: 0, w: 0}`. El valor devuelto se asignará a la
variable `position`. En el modo "triangle-list", cada vez que se ejecute el **sombreador
de vértices** se dibujará un triángulo conectando los 3 valores de `position` que devolvemos.

Positions in WebGPU need to be returned in *clip space* where X goes from -1.0
on the left to +1.0 on the right, Y goes from -1.0 at the bottom to +1.0 at the
top. This is true regardless of the size of the texture we are drawing to.

Las posiciones en WebGPU deben devolverse en *espacio de recorte* (*clip space*). Para ambas dimensiones, los
valores se comprenden entre -1.0 a +1.0. 
En el eje X, -1.0 representa la parte más a la izquierda, y +1.0 la parte más a la derecha.
En el eje Y, -1.0 representa la parte más inferior y +1.0 la parte más superior.
Esto es cierto independientemente del tamaño de la textura a la que estamos dibujando.

<div class="webgpu_center"><img src="resources/clipspace.svg" style="width: 500px"></div>

La función `vs` declara un vector de 3 `vec2f`, donde cada `vec2f` consta de dos valores de punto flotante de 32 bits. Luego, el código llena ese vector con 3 `vec2f`.

```wgsl
        let pos = array(
          vec2f( 0.0,  0.5),  // arriba y centrado horizontalmente
          vec2f(-0.5, -0.5),  // abajo a la izquierda
          vec2f( 0.5, -0.5)   // abajo a la derecha
        );
```

Finalmente, utiliza `vertexIndex` para devolver uno de los 3 valores del conjunto. Dado que la
función requiere 4 valores de punto flotante como tipo de retorno (`vec4f`), y `pos` es un conjunto
de tipo `vec2f`, el código añade `0.0` y `1.0` para los 2 valores restantes.

```wgsl
        return vec4f(pos[vertexIndex], 0.0, 1.0);
```

El módulo de sombreador también declara una función llamada `fs` que se declara con el atributo
`@fragment`, lo que la convierte en una función de sombreador de fragmentos.

```wgsl
      @fragment fn fs() -> @location(0) vec4f {
```

Esta función no toma parámetros y devuelve un `vec4f` en `location(0)`. Esto significa que escribirá
en el primer destino de renderizado (render target). Más adelante, haremos que el primer destino de renderizado sea
nuestro lienzo.

```wgsl
        return vec4f(1, 0, 0, 1);
```

El código devuelve `1, 0, 0, 1`, es decir, el color rojo. En WebGPU, los colores se especifican 
como valores de punto flotante de `0.0` a `1.0`, donde cada uno de los valores corresponden a
rojo, verde, azul y alfa respectivamente.

Cuando la GPU rasteriza el triángulo (lo dibuja con píxeles), llama al sombreador de fragmentos
para determinar de qué color pintar cada píxel. En nuestro caso, simplemente estamos devolviendo rojo.

Otra cosa importante es la etiqueta `label`. Casi todos los objetos que puedes crear en WebGPU pueden
tener una etiqueta. Las etiquetas son completamente opcionales, pero se considera una *buena práctica*. 
De esta manera, cuando obtienes un error, las implementaciones de WebGPU imprimirán un mensaje que incluye
las etiqueta relacionada con el error.

En una aplicación normal, tendrías cientos o miles de búferes, texturas, módulos de sombreador, canalizaciones, etc. Si obtienes un error como `"Error de sintaxis WGSL en shaderModule en la línea 10"`, ¿cuál de los 100 módulos de sombreador causó el error? Si etiquetas el módulo, obtendrás un mensaje de error más útil como `"Error de sintaxis WGSL en shaderModule('nuestros sombreadores de triángulo rojo codificados') en la línea 10"`, lo cual es mucho más informativo y te ahorrará mucho tiempo investigando la causa del problema.

Ahora que hemos creado un módulo de sombreador, lo siguiente que necesitamos hacer es crear un pipeline (canalización en Castellano) de renderizado.

```js
  const pipeline = device.createRenderPipeline({
    label: 'código para la canalización del triángulo rojo',
    layout: 'auto',
    vertex: {
      module,
      entryPoint: 'vs',
    },
    fragment: {
      module,
      entryPoint: 'fs',
      targets: [{ format: presentationFormat }],
    },
  });
```

En este caso, no hay mucho que ver. Configuramos `layout` como `'auto'`, lo que significa
que WebGPU definirá un diseño para todos los recursos (búfers, texturas y demás) en función
de lo que haya escrito en los shaders.

Luego, indicamos al render pipeline que utilice la función `vs` de nuestro módulo de shaders
como un shader de vértices, y la función `fs` como nuestro shader de fragmentos.
Además, especificamos el formato del primer render objetivo. Cuando hablamos de "render objetivo",
nos referimos a la textura en la que finalmente renderizamos.
Cuando creamos un pipeline, debemos especificar el formato de la(s) textura(s) que utilizaremos
en el pipeline para renderizar.
El elemento 0 del array `targets` corresponde a la posición 0, tal y como se especificó en el
valor de retorno del shader de fragmentos. Más adelante, configuraremos ese target para que sea una textura para el lienzo.

A continuación, preparamos un `GPURenderPassDescriptor` que describe qué texturas queremos dibujar y cómo utilizarlas.

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

Un `GPURenderPassDescriptor` tiene un array llamado `colorAttachments`, que enumera las texturas
se renderizarán y cómo tratarlas. Esperaremos para especificar qué textura queremos representar realmente. Por ahora, configuramos el valor de `clearValue`, `loadOp` y `storeOp`.

- `loadOp: 'clear'` especifica que se borre la textura al valor de `clearValue` antes de dibujar. La otra opción es `'load'`, que cargaría el contenido existente de la textura en la GPU para que podamos dibujar sobre lo que ya está allí.
- `storeOp: 'store'` significa almacenar el resultado de lo que dibujamos. También podríamos usar `'discard'`, lo que descartaría lo que dibujamos. Comentaremos cuándo podríamos querer hacer eso en otro artículo.

Ahora es el momento de escribir la función para el render.

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

Primero llamamos a `context.getCurrentTexture()` para obtener una textura que aparecerá en el
lienzo. Llamar a `createView` obtiene una vista en una parte específica de una textura, pero
sin parámetros, devolverá la parte predeterminada, que es lo que queremos en este
caso.
En este caso, nuestra única `colorAttachment` es una vista de textura de nuestro
lienzo, que obtenemos a través del contexto que creamos al principio. Nuevamente, el elemento 0 de
el array `colorAttachments` corresponde a `location(0)`, tal y se especificó para
el valor de retorno del fragment shader.

A continuación, creamos un codificador de comandos. Un codificador de comandos se utiliza para crear un búfer de comandos. Se utiliza para codificar comandos y luego "enviar" el búfer que creado para que se ejecuten los comandos.

<!-- codificador de "render pass" ??? -->
Luego utilizamos el codificador de comandos para crear un codificador de "render pass" llamando a `beginRenderPass`. Un codificador de render pass es un codificador específico para crear comandos relacionados con el renderizado. Le pasamos nuestro `renderPassDescriptor` para indicarle a qué textura queremos renderizar.

Codificamos el comando `setPipeline` para establecer nuestra pipeline y luego le indicamos que ejecute nuestro shader de vértices 3 veces llamando a `draw` con un valor de entrada 3. Por defecto, cada vez que se ejecuta nuestro shader de vértices 3 veces, se dibujará un triángulo conectando los 3 valores devueltos.

Finalizamos el render pass y luego concluimos el codificador. Esto devuelve un búfer de comandos que representa los pasos que acabamos de especificar. Finalmente, enviamos el búfer de comandos para que se ejecuten los comandos.

Al ejecutar `draw`, este será nuestro estado:

<div class="webgpu_center"><img src="resources/webgpu-simple-triangle-diagram.svg" style="width: 723px;"></div>

En el estado no hay textures, ni búfers, ni bindGroups pero sí hay un pipeline, un vertex shader y un fragment shader y un descriptor de render pass que especifica al shader la textura donde debe renderizar.

El resultado:

{{{example url="../webgpu-simple-triangle.html"}}}

It's important to emphasize that all of these functions we called
like `setPipeline`, and `draw` only add commands to a command buffer.
They don't actually execute the commands. The commands are executed
when we submit the command buffer to the device queue.

So, now we've seen a very small working WebGPU example. It should be pretty
obvious that hard coding a triangle inside a shader is not very flexible. We
need ways to provide data and we'll cover those in the following articles. The
points to take away from the code above,

* WebGPU just runs shaders. Its up to you to fill them with code to do useful things
* Shaders are specified in a shader module and then turned into a pipeline
* WebGPU can draw triangles
* WebGPU draws to textures (we happened to get a texture from the canvas)
* WebGPU works by encoding commands and then submitting them.

# <a id="a-run-computations-on-the-gpu"></a>Run computations on the GPU

Let's write a basic example for doing some computation on the GPU

We start off with the same code to get a WebGPU device

```js
async function main() {
  const adapter = await gpu?.requestAdapter();
  const device = await adapter?.requestDevice();
  if (!device) {
    fail('need a browser that supports WebGPU');
    return;
  }
```

When we create a shader module

```js
  const module = device.createShaderModule({
    label: 'doubling compute module',
    code: `
      @group(0) @binding(0) var<storage, read_write> data: array<f32>;

      @compute @workgroup_size(1) fn computeSomething(
        @builtin(global_invocation_id) id: vec3<u32>
      ) {
        let i = id.x;
        data[i] = data[i] * 2.0;
      }
    `,
  });
```

First we declare a variable called `data` of type `storage` that we want to be
able to both read from and write to.

```wgsl
      @group(0) @binding(0) var<storage, read_write> data: array<f32>;
```

We declare its type as `array<f32>` which means an array of 32bit floating point
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
later. For now we'll just set it to 1 with the attribute `@workgroup_size(1)`.
We declare it to have one parameter `id` which uses a `vec3u`. A `vec3u` is
three unsigned 32 integer values. Like our vertex shader above, this is the
iteration number. It's different in that compute shader iteration numbers are 3
dimensional (have 3 values). We declare `id` to get its value from the built-in
`global_invocation_id`.

You can *kind of* think of a compute shaders as running like this. This is an over
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
  const {x: width, y: height, z: depth} = workgroup.size;
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

Since we set `@workgroup_size(1)`, effectively the pseudo code above becomes

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

Finally we use the `x` property of `id` to index `data` and multiply each value
by 2

```wgsl
        let i = id.x;
        data[i] = data[i] * 2.0;
```

Above, `i` is just the first of the 3 iteration numbers.

Now that we've created the shader we need to create a pipeline

```js
  const pipeline = device.createComputePipeline({
    label: 'doubling compute pipeline',
    layout: 'auto',
    compute: {
      module,
      entryPoint: 'computeSomething',
    },
  });
```

Here we just tell it we're using a `compute` stage from the shader `module` we
created and we want to call the `computeSomething` function. `layout` is
`'auto'` again, telling WebGPU to figure out the layout from the shaders. [^layout-auto]

[^layout-auto]: `layout: 'auto'` is convenient but, it's impossible to share bind groups
across pipelines using `layout: 'auto'`. Most of the examples on this site
never use a bind group with multiple pipelines . We'll cover explicit layouts in [another article](webgpu-drawing-multiple-things.html).

Next we need some data

```js
  const input = new Float32Array([1, 3, 5]);
```

That data only exists in JavaScript. For WebGPU to use it we need to make a
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

Above we call `device.createBuffer` to create a buffer. `size` is the size in
bytes, in this case it will be 12 because size in bytes of a `Float32Array` of 3
values is 12. If you're not familiar with `Float32Array` and typed arrays then
see [this article](webgpu-memory-layout.html).

Every WebGPU buffer we create has to specify a `usage`. There are a bunch of
flags we can pass for usage but not all of them can be used together. Here we
say we want this buffer to be usable as `storage` by passing
`GPUBufferUsage.STORAGE`. This makes it compatible with `var<storage,...>` from
the shader. Further, we want to able to copy data to this buffer so we include
the `GPUBufferUsage.COPY_DST` flag. And finally we want to be able to copy data
from the buffer so we include `GPUBufferUsage.COPY_SRC`.

Note that you can not directly read the contents of a WebGPU buffer from
JavaScript. Instead you have to "map" it which is another way of requesting
access to the buffer from WebGPU because the buffer might be in use and because
it might only exist on the GPU.

WebGPU buffers that can be mapped in JavaScript can't be used for much else. In
other words, we can not map the buffer we just created above and if we try to add
the flag to make it mappable we'll get an error that that is not compatible with
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

In order to tell our shader about the buffer we want it to work on we need to
create a bindGroup

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

We get the layout for the bindGroup from the pipeline. Then we setup bindGroup
entries. The 0 in `pipeline.getBindGroupLayout(0)` corresponds to the
`@group(0)` in the shader. The `{binding: 0 ...` of the `entries` corresponds to
the `@group(0) @binding(0)` in the shader.

Now we can start encoding commands

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
this case we pass it `input.length` which is `3` telling WebGPU to run the
compute shader 3 times. We then end the pass.

Here's what the situation will be when `dispatchWorkgroups` is executed

<div class="webgpu_center"><img src="resources/webgpu-simple-compute-diagram.svg" style="width: 553px;"></div>

After the computation is finished we ask WebGPU to copy from `buffer` to
`resultBuffer`

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

We then map the results buffer and get a copy of the data

```js
  // Read the results
  await resultBuffer.mapAsync(GPUMapMode.READ);
  const result = new Float32Array(resultBuffer.getMappedRange());

  console.log('input', input);
  console.log('result', result);

  resultBuffer.unmap();
```

To map the results buffer we call `mapAsync` and have to `await` for it to
finish. Once mapped, we can call `resultBuffer.getMappedRange()` which with no
parameters will return an `ArrayBuffer` of entire buffer. We put that in a
`Float32Array` typed array view and then we can look at the values. One
important detail, the `ArrayBuffer` returned by `getMappedRange` is only valid
until we called `unmap`. After `unmap` its length with be set to 0 and its data
no longer accessible.

Running that we can see we got the result back, all the numbers have been
doubled.

{{{example url="../webgpu-simple-compute.html"}}}

We'll cover how to really use compute shaders in other articles. For now, you
hopefully have gleaned some understanding of what WebGPU does. EVERYTHING ELSE
IS UP TO YOU! Think of WebGPU as similar to other programming languages. It
provides a few basic features, and leaves the rest to your creativity.

What makes WebGPU programming special is these functions, vertex shaders,
fragment shaders, and compute shaders, run on your GPU. A GPU could have over
10000 processors which means they can potentially do more than 10000
calculations in parallel which is likely 3 or more orders of magnitude than your
CPU can do in parallel.

## Simple Canvas Resizing

Before we move on, let's go back to our triangle drawing example and add some
basic support for resizing a canvas. Sizing a canvas is actually a topic that
can have many subtleties so [there is an entire article on it](webgpu-resizing-the-canvas.html).
For now though let's just add some basic support

First we'll add some CSS to make our canvas fill the page

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
the resolution of the canvas itself so you might notice if you make the example below
large, like if you click the full screen button, you'll see the edges of the triangle
are blocky.

{{{example url="../webgpu-simple-triangle-with-canvas-css.html"}}}

`<canvas>` tags, by default, have a resolution of 300x150 pixels. We'd like to
adjust the canvas resolution of the canvas to match the size it is displayed.
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

In the code above we go over all the entries but there should only ever be one
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

In the following articles we'll cover various ways to pass data into shaders.

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
Textures are most complex as they have tons types and options.

I'm a little bit worried these article will be boring at first. Feel free to
jump around if you'd like. Just remember if you don't understand something you
probably need to read or review these basics. Once we get the basics down we'll
start going over actual techniques.

One other thing. All of the example programs can be edited live in the webpage.
Further, they can all easily be exported to [jsfiddle](https://jsfiddle.net) and [codepen](https://codepen.io)
and even [stackoverflow](https://stackoverflow.com). Just click "Export".

<div class="webgpu_bottombar">
<p>
The code above gets a WebGPU device in very terse way. A more verbose
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
