# 背景

我自从接触到可视化相关的工作，就一直想更进一步的学习可视化相关的其他内容，特别是3D可视化。这个方向其实对大多数人来说门槛相比普通的前端开而言是有一定门槛的，作为以后的一个职业发展方向，个人认为也是一个很好的选择。当然前提是需要持续学习和积累，这个方向简单入个门虽然很容易，但如果要想实现更多的复杂场景以及各种应用，还是需要大量的练习和训练。之前有趣看了《WebGL开发指南》，对于3D可视化基础有了一定了解，最近工作之余想学Three.js，后面会根据自己的学习进度同步自己的总结文章。

# 我了解的Three.js

关于Three.js，我大致的理解，它就是一个流行的开源JS库，这个库主要是用来创建交互式3D图形和动画。它的底层其实是WebGL，提供一套相对WebGL更简单的API，可以让开发者留下更多的精力在设计场景本身，目前很多领域都有广泛的应用，比如：游戏、可视化数据、房屋全景、模型等等。

那么，这些炫酷场景背后到底是如何实现的呢？

# 使用Three.js实现3D基础流程

Three.js它的三个核心就是，**虚拟场景**、**虚拟相机**、**几何体**、**渲染器**，通过这几个必备的元素，生成最后的**投影图**。你可以想象自己是一个摄影师，自己先租一个摄影棚，准备一个摄像机，还需要邀请模特，最后拍出了成片。这就能看成一个最基础的Three.js实现3D流程。接着作为一名更专业的摄影师，还需要根据不同的主题给模特穿不同面料的衣服(**材质**)，并且需要配合**灯光**，不同材质不同角度的光照拍出的效果会不一样。

## 场景（Scene）

```js
const scene = new THREE.Scene();
```

这里的`Scene`对象就是虚拟3D场景。

## 相机（Camera）

相机定义了观察者在场景中看到的视角。Three.js提供了多种类型的相机，包括透视相机（`PerspectiveCamera`）和正交相机（`OrthographicCamera`），这次就用透视相机。

```JS
const camera = new THREE.PerspectiveCamera();
```

相机不同视角，包括相机的位置和相机的观察目标，好比摄像，作为摄影师该将镜头摆在哪里，调整到一个什么样的方向，对于最后形成的图像都是有重要影响的。

1.  相机的位置
    `camera.position.set(x,y,z) `
2.  相机的观察方向
    `camera.lookAt(x, y, z)`

如果还不理解的话，可以看下图：

![WechatIMG636.jpeg](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a37e4801ff354ea69820f614b816a81c~tplv-k3u1fbpfcp-watermark.image?)


## 几何体（Geometry）

几何体是3D对象的基本构建块，Three.js提供了很多种几何体API，如长方体、球体、圆柱等。

```js
//BoxGeometry：长方体
const geometry = new THREE.BoxGeometry(100, 100, 100);
// SphereGeometry：球体
const geometry = new THREE.SphereGeometry(50);
// CylinderGeometry：圆柱
const geometry = new THREE.CylinderGeometry(50,50,100);
```

## 材质（Material）

上面有提及，一个物体表面的材质不同，在相同的光照下会展示不同的效果。Three.js提供的**网格材质**，有的受光照影响，有的不受光照影响。

*   基础网格材质`MeshBasicMaterial`
    不受光的影响
*   漫反射网格材质`MeshLambertMaterial`
    受光的影响
    
    还有其他受光照影响的材质，比如高光、物理材质，这些会根据材质的区分，针对光照的影响程度会有所不同。

为什么会有这样的区别？

### 光的反射

*   一道光照射到物体的表面，照射在物体上面的光线部分会被吸收，部分会被反射。如果这个物体是透明的，有些光还会被折射。
*   反射部分的多少确定物体的颜色和亮度。
*   物体表面在白光上看起来是红色的，就是因为光线中的红色分量被反射，而其他的颜色被吸收掉了。
*   反射光被反射的方式又会根据物体表面的光滑程度定向确定。如果物体表面光滑，物体就会显得明亮，如果比较粗糙，那么就显得暗淡。这里就能解释在漫反射网格材质中为什么还做了其他材质的区分。

材质不同（光滑、粗糙）会影响最后的反射效果，那么如果光线的照射方向不同产生不同大小的入射角，这对最后的效果会有怎么样的影响？

1.  **反射方向**：入射角度越大，光线在表面上的反射方向越偏离入射角。这意味着当光线以较大的角度入射时，反射光线将更接近表面的切线方向而不是垂直于表面。
2.  **反射强度**：入射角度越大，反射光线的强度通常会减弱。这是因为入射角度越大，光线与表面的接触面积减小，从而导致反射光线的能量损失。
3.  **反射模式**：在较小的入射角度下，反射可能会更加镜面化，即反射光线具有明确的方向性和反射角度。然而，随着入射角度的增加，反射可能会变得更加漫反射，即反射光线以更广泛的角度散射。
4.  **表面形貌**：几何形状和纹理也会对光照的反射效果产生影响。不同入射角度下，表面形貌的微观细节对光线的反射和散射方式可能会有所不同。

## 灯光（Light）

光源在实现3D场景，是一个很重的一部分，就好比摄影，一个成片的好坏很大程度上取决于光打得到不到位。Three.js中主要分为下面四类光源:

*   环境光
*   点光源
*   聚光灯光源
*   平行光

其中环境光相对特殊，它没有具体的位置，它是就整个场景的一个光照情况，就好比是摄影棚里一日时间的光照情况。其他三种光源都可以设置位置，而且这几种光源对应光的散发方向会有区别：

![WechatIMG632.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/24821bd2fe204a65b8e577e85808ea37~tplv-k3u1fbpfcp-watermark.image?)

## 渲染器（Renderer）

渲染器负责将场景和相机中的3D对象渲染到浏览器窗口中--`WebGLRenderer`

```js
// 创建渲染器对象
const renderer = new THREE.WebGLRenderer();
// 设置渲染区域的尺寸
renderer.setSize(width, height);
// 执行渲染(scene是之前创建的场景、camera是之前创建的相机)
renderer.render(scene, camera); 
```

但是最后这个渲染器只是执行了渲染操作，那么如何显示在浏览器上呢？

### 渲染器Canvas画布属性.domElement

渲染器`WebGLRenderer`通过属性`.domElement`可以获得渲染方法`.render()`生成的Canvas画布，**.domElement本质上就是一个HTML元素**，我们可以将这个元素插入到自己提前定义好的一个div中，这样最后的渲染效果就会显示在浏览器中。

```js
<div id="webgl" style="margin-top: 200px;margin-left: 100px;"></div>
document.getElementById('webgl').appendChild(renderer.domElement);
```

现在大致了解了实现3D的几个重要概念，下面可以根据上面的几个重要部分进行组合，输出一个简单的立方体3D图形。在此之前，先说明一下Three.js中的三维坐标，如图：

![WechatIMG635.jpeg](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bf8a2fe5f3e6492dba93af2f332935b4~tplv-k3u1fbpfcp-watermark.image?)

这里注意在Three.js中默认Y轴是向上的。

## 实现一个简单的3D

### 准备部分

提前下载Three.js模块，这里直接用html、js的基础方式实现，可以通过`type="importmap"`引入。

```html
<script type="importmap">
    {
        "imports": {
            "three": "../lib/three-js/three.module.js",
            "three/addons/": "../lib/three-js/jsm/"
        }
    }
</script>
```

### 具体实现

```html
<div id="webgl" style="margin-top: 200px;margin-left: 100px;"></div>
<script type="module">
    import * as THREE from 'three';
    // 创建虚拟场景
    const scene = new THREE.Scene();
    // 创建物体集合
    const geometry = new THREE.BoxGeometry(100, 60, 20);
    // 设置物体表面材质
    const material = new THREE.MeshBasicMaterial({
        color: 0x0000ff, //设置材质颜色
        transparent:true,//开启透明
        opacity:0.5,//设置透明度
    });
    // 创建物体模型
    const mesh = new THREE.Mesh(geometry, material);
    // 设置物体模型在场景中的位置
    mesh.position.set(100,0,0);
    // 将模型添加到虚拟场景中
    scene.add(mesh);

     // 没有特定方向，整体改变场景的光照明暗
    const ambient = new THREE.AmbientLight(0xffffff, 0.4);
    scene.add(ambient);

    // 平行光
    const directionalLight = new THREE.DirectionalLight(0xffffff, 1);
    directionalLight.position.set(100, 60, 50);
    // 方向光指向对象网格模型mesh，可以不设置，默认的位置是0,0,0
    directionalLight.target = mesh;
    scene.add(directionalLight);

    // 定义画布的尺寸
    const width = 800;
    const height = 500;

    // 创建一个透视投影相机实例
    const camera = new THREE.PerspectiveCamera(30, width / height, 1, 3000);
    // 设置相机位置
    camera.position.set(200,200,100);
    // 设置相机视点,这里就是物体位置
    camera.lookAt(mesh.position);

    // 创建一个渲染器
    const renderer = new THREE.WebGLRenderer();
    // 设置渲染器渲染区域尺寸
    renderer.setSize(width, height);
    renderer.render(scene, camera);
    document.getElementById('webgl').appendChild(renderer.domElement);
</script>
```

最后效果如下：

![WechatIMG6015.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d9cf0d6547c2475c8fd7b8e71dcd70ab~tplv-k3u1fbpfcp-watermark.image?)

这张图其实如果不是很熟悉坐标体系，一下子没法想象这个物体所在的位置以及整体的一个三维空间。我们可以借助Three.js给我们提供的`AxesHelper`辅助观察的坐标系，可以方便定位。点光源和平行光源分别提供了各自的辅助观察，方便我们清楚知道光源在不同的点不同的观察位置对最后的成像会产生怎么样的差异。

```js
// AxesHelper：辅助观察的坐标系
const axesHelper = new THREE.AxesHelper(150);
scene.add(axesHelper);

const pointLight = new THREE.PointLight(0xffffff, 1.0);
pointLight.position.set(100, 60, 50);
scene.add(pointLight);
// 点光源辅助观察
const pointLightHelper = new THREE.PointLightHelper(pointLight, 10);
scene.add(pointLightHelper);

const directionalLight = new THREE.DirectionalLight(0xffffff, 1);
directionalLight.position.set(100, 60, 50);
directionalLight.target = mesh;
scene.add(directionalLight);
// 平行光辅助观察
const directionalLightHelper = new THREE.DirectionalLightHelper(directionalLight, 10, 0x0000ff);
scene.add(directionalLightHelper);
```

效果如下：

![WechatIMG633.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d4f9fc0c609940ef9b76b011f84fc151~tplv-k3u1fbpfcp-watermark.image?)

这样还不够生动形象，我需要可以通过鼠标对当前进行旋转、平移、放大缩小来查看不同角度看上去的效果是怎么样的。这里就需要用到Three.js提供的相机控件轨道控制器`OrbitControls`。这个相机控件轨道控制器内部的原理也很简单：

**就是通过改变相机的位置、观察方向，通过设置更远的位置，看到更小的视图效果**，原理如图：

![WechatIMG634.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/50bd703a08da4f2f8c14357fb1bb4efb~tplv-k3u1fbpfcp-watermark.image?)

```js
const controls = new OrbitControls(camera, renderer.domElement);
controls.addEventListener('change', function () {
    renderer.render(scene, camera); 
});
```

以上就是整个Three.js进行3D渲染的基本实现，通过这次的实现，我对于Threejs实现的几个重要概念有了很好的理解，熟悉了常用的几个API。因为之前看过相关WebGL书籍，所以深有感触，Three.js提供的API是真的简单，基本做一些事情一两行代码就可以实现，让自己更多的去考虑自己到底要实现一个什么样的场景上。当然WebGL很重要，如果想要达到进阶的程度，就需要去啃WebGL相关书籍了。最后还有一个最重要的一点，线性代数！！！因为之前在学习WebGL过程中很多都会涉及矩阵计算，如果自己的数学基础相对好的话，这些东西学习起来就没那么困难了（这里叹口气）😢

未完待续...
