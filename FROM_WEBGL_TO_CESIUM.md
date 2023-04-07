
# 从WebGL到Cesium

## 前言

Cesium中是可以实现一些酷炫的效果，如何实现这些效果，对于初学者而言有点无从下手，本文就是关于Cesium底层改造的一篇入门的文章

## Cesium简介

Cesium是一款基于WebGL的地图引擎，可以用于展示三维地球、地形和GIS数据。它是由美国宾夕法尼亚州的AGI公司开发的。Cesium不仅支持在线地球模型的创建和交互式3D可视化，还支持几乎所有地图投影和坐标系的展示，包括来自地球卫星、航空监测、UAV测绘、激光匹配和图像测量等多种数据源。

![webgl-2d](/assets/images/cesium.jpg)

## WebGL简介

WebGL是基于OpenGL ES的3D渲染技术，可以在浏览器中实现高性能的3D渲染和动画效果。WebGL可以利用<strong style='color:red'>GPU加速</strong>，支持硬件加速和图像处理，能够处理复杂的3D渲染效果。 WebGL的性能是明显优于其他浏览器中的图形绘制技术，svg、Canvas API。

![webgl-2d](/assets/images/webgl-water.jpg)

## 为什么要从WebGL到Cesium

本文想写的是一个Cesium底层开发的入门文章，就跳不过Cesium中最基础的东西WebGL。

有些东西开起来很复杂，当我们从底层原理一步一步理解后，那些看似复杂的问题，也许就不那么难了。

## WebGL

我们先通过一个简单的例子了解webGL长什么样子，以下是绘制一个简单正方形的步骤，通过这个例子我们会得到如图所示的结果。

![webgl-2d](/assets/images/webgl-rect.jpg)

绘制上面这个图形，需要下面9个步骤

1. 创建一个canvas元素：

```html
<canvas id="myCanvas"></canvas>
```

2. 获取WebGL上下文：

```javascript
var canvas = document.getElementById('myCanvas');
var gl = canvas.getContext('webgl'); //如果gl获取为空，就说明，这个浏览器不支持webgl
```

3. 创建<strong style='color:red'>顶点着色器</strong>：

```javascript
var vertexShaderSource = `
    attribute vec2 a_position;
    void main() {
        gl_Position = vec4(a_position, 0.0, 1.0);
    }
`;
```

顶点着色器接收属性a_position，并通过gl_Position将顶点位置传递给WebGL。

4. 创建<strong style='color:red'>片元着色器</strong>：

```javascript
var fragmentShaderSource = `
    precision mediump float;
    void main() {
        gl_FragColor = vec4(0.0, 0.0, 1.0, 1.0);
    }
`;
```

片元色器指定了形状的颜色。

5. 编译顶点着色器与片段着色器：

```javascript
var vertexShader = gl.createShader(gl.VERTEX_SHADER);
gl.shaderSource(vertexShader, vertexShaderSource);
gl.compileShader(vertexShader);

var fragmentShader = gl.createShader(gl.FRAGMENT_SHADER);
gl.shaderSource(fragmentShader, fragmentShaderSource);
gl.compileShader(fragmentShader);
```

6. 将顶点着色器与片元着色器连接，并创建一个程序：

```javascript
var program = gl.createProgram();
gl.attachShader(program, vertexShader);
gl.attachShader(program, fragmentShader);
gl.linkProgram(program);
gl.useProgram(program);
```

7. 创建一个buffer代表形状的点集：

```javascript
var positionBuffer = gl.createBuffer();
gl.bindBuffer(gl.ARRAY_BUFFER, positionBuffer);
var positions = [
    -0.5, 0.5,
    0.5, 0.5,
    -0.5, -0.5,
    0.5, -0.5,
];
gl.bufferData(gl.ARRAY_BUFFER, new Float32Array(positions), gl.STATIC_DRAW);
```

这里创建一个二维坐标系，将其四个端点储存到一个Float32Array中。

8. 启动顶点着色器中的a_position：

```javascript
var positionLocation = gl.getAttribLocation(program, "a_position");
gl.enableVertexAttribArray(positionLocation);
gl.vertexAttribPointer(positionLocation, 2, gl.FLOAT, false, 0, 0);
```

这里获取a_position的位置，启用它，并将它绑定到之前创建的buffer上。

9. 渲染图形：

```
gl.drawArrays(gl.TRIANGLE_STRIP, 0, 4);
```

可以看到，虽然只是画了一个矩形，却总共用到了9个步骤，这看起来是是有些繁琐，但包含了<strong style='color:red'>WebGL的关键步骤</strong>

其中片元着色器和顶点着色器的用的是GLSL（OpenGL Shading Language），一种基于C语言的语法，关于GLSL学习，可以前往这本经典的入门教材
<https://thebookofshaders.com/?lan=ch>

但是，如果只是画一个二维图形，Canvas API也可以实现，并且在代码实现上比WebGL简单多了，如下：

```javascript
const canvas = document.getElementById('canvas');
const ctx = canvas.getContext('2d');

ctx.fillStyle = 'blue';
ctx.fillRect(10, 10, 150, 100);
```

二维图形完全不能展示我们WebGL能力，所以我们现在，我们换个高级点的用法，用WebGL画一个三维的立方体。

![webgl-3d](/assets/images/webgl-cube.jpg)

具体代码请参考[2-webgl-cube.html]这个示例

这个是怎么实现的呢，WebGL本身只是提供了在二维平面上绘制的能力，要绘制三维图形，我们就需要到矩阵变换，矩阵变换可以将三维的坐标投影到一个二维平面上，如下图。

![webgl-3d](/assets/images/project-matrix.jpg)

矩阵变换的原理在这里我们不详细介绍
原因：

1. 需要一些线性代数的基础知识
2. 不用担心看不懂，通常三维引擎会帮我们把这一步做了，我们只需要使用API封装的相关方法即可

## Threejs

看完上面的两个例子，我们理所应当的就应该把threejs搬出来了，因为我们需要一个三维引擎，来避免直接接触繁琐的WebGL代码

看看threejs是如何实现一个立体

```javascript
    // Setup renderer and camera
      var renderer = new THREE.WebGLRenderer();
      renderer.setSize(window.innerWidth, window.innerHeight);
      document.body.appendChild(renderer.domElement);

      var camera = new THREE.PerspectiveCamera(
        75,
        window.innerWidth / window.innerHeight,
        0.1,
        1000
      );
      camera.position.z = 5;

      //创建一个场景
      var scene = new THREE.Scene();

      //创建一个立方体
      var geometry = new THREE.BoxGeometry(1, 1, 1);
      var material = new THREE.MeshBasicMaterial({
        color: 0x00ff00,
        wireframe: true,
      });
      var cube = new THREE.Mesh(geometry, material);
      scene.add(cube);

```

![webgl-3d](/assets/images/threejs-cube.jpg)

在threejs中，我们只需要创建一个BoxGeometry对象，把他添加到场景中，就可以渲染这个立方体了，threejs做的只是把BoxGeomtry翻译成原生的WebGL代码执行。

当然，一个三维引擎要做的不仅仅是这个。为了真实的还原三维显示场景，我们需要一个三维引擎提供如下能力

1. 现实中都是有光源的，所以三维引擎需要提供<strong style='color:red'>光照效果</strong>的能力，这样我们看到物体才是和现实世界一样是有明有暗的
2. 现实中有光就会有阴影，所以需要三维引擎能提供<strong style='color:red'>阴影效果</strong>
3. 现实中有些材质是会反射光源的，例如镜子，水面，金属材料等，这个时候我们就需要有<strong style='color:red'>光纤追踪</strong>
4. 我们需要和三维物体互动，可以点击<strong style='color:red'>拾取</strong>三维物体

以上4个能力都是三维引擎用算法计算出来的。我们不展开讲，在应用层面，我们只需要了解这个概念即可。对原理感兴趣的同学可以通过关键字搜索相关资源进行学习

下面的例子，就是展示threejs如何实现这些能力，例如，虽然计算光源的算法很复杂，但是在三维引擎中，我们并不需要关心具体的计算，只要定义光源的位置即可

* Threejs 一个在光源下的立方体并且有阴影

![webgl-3d](/assets/images/threejs-cube-shadow.jpg)

那既然threejs对WebGL进行了封装，是不是我们在基于threejs封装可以得到一个三维地图渲染引擎呢，理论上是的，harp.gl就是做了这个事情，但是目前主流的两个三维地图引擎并不是基于threejs或者其他三维引擎再次封装得到的，而是基于原生的WebGL封装，这样看来，了解WebGL看起来是比较划算的，了解了底层，就能了解好几个引擎了。

## 从三维引擎到Cesium

世面上常见的三维引擎，有threejs，babylonjs，还有国人之光cocos，他们更多的关注点是渲染的效果和交互，例如cocos主要拿来做游戏，都可以做出很酷炫的效果。但他们涉及的数据量都比较小。例如一个模型，一栋建筑，或者一个园区，当数据量大了之后，这些引擎是承载不住的，会出现掉帧甚至卡死的情况，三维地图引擎就是为了解决这个应运而生， 他们就是为了解决大数据量的问题。

例如：

1. 支持栅格瓦片和矢量瓦片数据
2. 支持按需加载地球上的物体，例如cesium的3dTiles，用于解决大量的三维模型加载
3. 倾斜摄影数据，点云数据的大批量加载

然后我们再拉回现实，解决大数据量的展示，确实用了一些牛逼的算法。但这些的前提，还是把数据如何翻译成WebGL执行。这就涉及到cesium底层对WebGL的封装了，这个封装就是<strong style='color:red'>DrawCommand</strong>，要了解Cesium的绘制原理，就要从DrawCommand开始看。

首先我们看看一个简单的的DrawCommand，长什么样子

```javascript

//WebGL中的program的封装
var shaderProgram = Cesium.ShaderProgram.fromCache({
        context: context,
        vertexShaderSource: vs,  //顶点着色器
        fragmentShaderSource: fs, //片元着色器
        attributeLocations: attributeLocations, //属性
    });

var drawCommand = new Cesium.DrawCommand({
        modelMatrix: modelMatrix,  //模型原点
        vertexArray: va, //顶点数组
        shaderProgram: shaderProgram, //着色器的Program
        uniformMap: uniformMap, //即WebGL中的uniform，用于存储给WebGL的常量
        renderState: renderState, //WebGL Program运行时需要用到的渲染参数
        pass: Cesium.Pass.OPAQUE, //运行的优先级。可参考Cesium.Pass中的常量
    });

```

![cesium-draw-command](/assets/images/cesium-draw-command.jpg)

一个DrawCommand就是一个WebGL中的program的封装。

其中两个类非常重要

一个是cesium的Geometry类，例如要创建一个和子我们就是用Cesium.BoxGeometry创建一个geometry对象。

```javascript
var box = new Cesium.BoxGeometry({
       vertexFormat:  Cesium.VertexFormat.POSITION_NORMAL_AND_ST,
       maximum: new Cesium.Cartesian3(250000.0,250000.0, 250000.0),
       minimum: new Cesium.Cartesian3(-250000.0,-250000.0, -250000.0),
    });
var geometry = Cesium.BoxGeometry.createGeometry(box);

```

这个里面我么需要注意，vertexFormat这个参数，geometry类不仅是会生成几何顶点的坐标，还会生成一些其他的参数，例如st是用于绘制纹理，normal是法线，这些在之后的计算中都可能会用到。Cesium的VertexFormat类中有更详细的介绍。

有了顶点参数，即WebGL中的attribute属性，把这个属性传给一个WebGL的Program，就可以完成绘制了

也就是Cesium.ShaderProgram这个类

```javascript


var program = Cesium.ShaderProgram.fromCache({
        context: context,
        vertexShaderSource: vs,  //顶点着色器
        fragmentShaderSource: fs, //片元着色器
        attributeLocations: attributeLocations, //属性
    });

```

最后，用DrawCommand，把顶点数据和program装起来

```javascript

     this.drawCommand = new Cesium.DrawCommand({
            modelMatrix: modelMatrix,
            vertexArray: va,
            shaderProgram: shaderProgram,
            uniformMap: uniformMap,
            renderState: renderState,
            pass: Cesium.Pass.OPAQUE,
          });

```

DrawCommand还需要交给Cesium的执行队列来执行，这时候就需要用到Primitive类了，Primitive类中主要使用到的是update方法，这个方法会在Cesium每一帧渲染的时候被调用。

```javascript

class MyPrimitive{

    createCommand(){
        //TODO  创建一个DrawCommand
    }


    /**
     * 实现Primitive接口，供Cesium内部在每一帧中调用
     * @param {Cesium.FrameState} frameState
     */
    update(frameState) {
    if (!this.drawCommand) {
        this.createCommand(frameState.context);
    }
    frameState.commandList.push(this.drawCommand);
    }
}

//创建Primitive实例
var primitive = new MyPrimitive(modelMatrix);
viewer.scene.primitives.add(primitive);

```

完整的例子见[6-cesium-draw-command.html]

## 总结和思考

* WebGL是三维引擎的基础，不论使用哪种引擎，要彻底搞清楚，学习WebGL是逃不掉的一环

* Cesium是暴露出来的DrawCommand，几乎相当于把原生WebGL暴露出来了，所以可扩展性是非常强的。例如，有一些有趣的shader网站，里面有各种各样炫酷的着色器
[https://www.shadertoy.com/](https://www.shadertoy.com/)
[http://glslsandbox.com/](http://glslsandbox.com/)
[https://shaderlab.com/](https://shaderlab.com/)

   我们可以在网站上找好看的shader，然后迁移到cesium

* 从文中的例子，我们可以看出，不论是Threejs还是Cesium，都是对WebGL的封装，也就是说，如果在threejs中看到了很好的功能，理论上讲，我们是可以从WebGL这个层面迁移过来的。

## 还有些有意思的东西

例如，Cesium中的地球是如何实现的，光照效果怎么实现的，阴影怎么实现的，如何拾取物体等等，这些都是基于WebGL实现，篇幅不够，需要另起一章。
