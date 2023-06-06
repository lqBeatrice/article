# 背景

前段时间，公司后台编辑平台需要出一个可以编辑文章的功能，在线编辑器目前网上有很多插件，当时我们选择的是BraftEditor。选择的理由基础功能用起来相对简单，相比其他编辑器开发效率更高。但是后面又新增了一个需求，需要我们拓展一个支持显示边框的功能，前提是在这个编辑器的基础上做一些拓展性的工作。当时翻过`BraftEditor`的官网demo和github，里面确实有一些已经实现的可集成的拓展控件，比如`Table`，最后就决定自己研究一个新的显示控件。

# 实现流程

## 1.定义Block类型和名称

整体需求就是给文本或者图片自动添加一个边框的功能，设置的类型和名称如下：

```js
{
    type: 'block',
    name: 'border-block',
}
```

名称这里可以自己随便定义一个通俗易懂且唯一的字符串，但是类型这里需要注意我们要从编辑器拓展支持的类型里面选择一个符合要求的。以下是目前支持的几种`extension`类型：
```js
control 控件
inline-style 内联样式
entity 实体 也是内联
block 块
decorator 装饰器
prop-interception 拦截
```
上述比较符合的就是**block类型**。（PS 我看了一下里面的源码，有两种类型atomic、block-style还是待开发状态，到目前为止没有更新，看来是已经搁置了😢）

## 2.定义渲染renderMap
即定义在编辑器内部展示的dom标签结构。

```js
import { Map } from 'immutable';
const getBorderBlockRenderMap = (options) => (props) => {
  return Map({
    'border-block': {
      element: 'div',
      wrapper: <BorderBlock {...options} {...props} />
    }
  })
}

{
    type: 'block',
    name: 'border-block',
    renderMap: getBorderBlockRenderMap()
}
```
这里注意编辑器内部的一些数据结构都是用的`immutable`，比如`Map`和`List`是常用的两个，我们需要遵守他们内部的一些结构类型。

`renderMap`就是一个键值对，其中`key`就是之前定义的扩展名称表示唯一性，值包含渲染的基本`element`和`wrapper`。如果是一个简单的拓展直接用一些简单标签做处理即可，考虑到这次实现的边框内容需要支持包含文本和图片两种类型，所以用了一个单独的类去做处理。

当然在实现这个`renderMap`之前，先要预设好这种边框的组件内部大致会是什么样的dom结构，然后根绝这个dom结构做后续的处理。

dom结构：

```html
    <section class="border-block-section">
        <div class="border-block-item">
            <img class="border-inner-image" .../> // 有图片的情况
        </div>
    </section>
```

编辑器会自动给图片类型的标签做转换，所以这边需要针对边框中的图片做自定义处理，后续在生成html的时候也方便得出自己想要的结果。但是想自定义图片dom结构，肯定需要拿到图片的基本信息比如：url之类的，这样的数据又从哪边能拿到呢？这块后面写到importer的时候一起说明，先上代码：

```js
class BorderBlock extends React.Component {
  constructor(props) {
    super(props);
  }
  render() {
    const { children, editorState } = this.props;
    return (
      <section className='border-block-section'>
        {
          children.map(child => {
            try {
              const childBlock = editorState.getCurrentContent().getBlockForKey(child.key);
              const childBlockData = childBlock?.getData();
              let newEle = React.cloneElement(child, {
                className: 'border-block-item'
              })
              if (childBlockData?.get('type') === 'border-block-image') {
                newEle = React.cloneElement(child, {
                  className: 'border-block-item',
                }, <img width="100%" className="border-inner-image" src={childBlockData.get('url')} />)
              }
              return newEle;
            } catch (error) {
              console.log(error);
              return <></>;
            }
          })
        }
      </section>
    );
  }
}
```

## 3.定义importer和exporter

### importer

简单说它的作用是将HTML转成editorState，详细的说就是，内部会遍历HTML所有的标签，我们可以根据特定的标签名或者标签属性决定转换成的类型。

这里需要注意考虑工具本身自定义某些标签的转换逻辑，比如`img`标签，BraftEditor会默认变成`media`类型，我们在需要的时候做特殊处理。特殊处理的过程中需要进行数据的传递：

**返回类型**：

**{ type: ‘’, data:{}} or null（null会走到默认处理）**

其中传递的data会通过`editorState.getCurrentContent().getBlockForKey(key).getData()`拿到，在renderMap中可以通过这个方式拿到特殊的参数。这里就解答了上面的传值疑问。

```js
importer: (nodeName, node) => {
      if (nodeName.toLowerCase() === 'div' && node.classList && node.classList.contains('border-block-item')) {
        let imgData = {};
        if (node.firstChild && node.firstChild.nodeName === 'IMG') {
          imgData = {
            type: 'border-block-image',
            url: node.firstChild.getAttribute?.('src') || ''
          }
        }
        return {
          type: 'border-block',
          data: imgData
        }
      }
      // 防止img标签转成media
      if (nodeName.toLowerCase() === 'img' && node.classList.contains('border-inner-image')) {
        return {
          type: 'border-block'
        }
      }
      return null
    }
```

### exporter

将editorState输出成HTML，遍历所有editorState中的block，判断类型包装不同标签以及标签属性。

**返回类型**：

**{start,end}表示当前内容以start标签开头，以end标签结尾**

通过使用`contentState.getBlockBefore`和`contentState.getBlockAfter` ，判断当前block的前一个和下一个block的类型来生成当前block需要的包装标签。

比如：定义了一个新的类型border，他的包装标签`<div class=”border”></div>` ，如果返回这个标签可以通过`class`特殊处理不同的展现形式。而且这个类型会包含很多子类型，所以需要很好的分清楚在哪里是包装标签的头，哪里是尾，即找到标签开始和结束的地方，把逻辑和关系理清楚就可以了。

```js
exporter: (contentState, block) => {
      if (block.type.toLowerCase() !== 'border-block') {
        return null;
      }
      const previousBlock = contentState.getBlockBefore(block.key);
      const nextBlock = contentState.getBlockAfter(block.key);
      const previousBlockType = previousBlock && previousBlock.getType();
      const previousBlockData = previousBlock ? previousBlock.getData().toJS() : {};
      const nextBlockType = nextBlock && nextBlock.getType();
      const nextBlockData = nextBlock ? nextBlock.getData().toJS() : {};

      let start = '';
      let end = '';
      if (previousBlockType !== 'border-block') {
        start = `<section class="d-f f-b-c border-block-section"><div class="border-block-content"><div class="border-block-item">`;
      } else if (previousBlockData.type !== block.data.type ||
        (previousBlockData.type === block.data.type && previousBlockData.type === 'border-block-image')
      ) {
        start = '<div class="border-block-content"><div class="border-block-item">';
        if (block.data.type === 'border-block-image') {
          start = `<div class="border-block-content"><div class="border-block-item border-block-image"><img class="border-inner-image" src="${block.data.url}" >`;
        }
      } else {
        start = '<div class="border-block-item">';
      }

      if (nextBlockType !== 'border-block') {
        end = '</div><!--rm--></div><!--rm--></section>';
      } else if (nextBlockData.type !== block.data.type ||
        (previousBlockData.type === block.data.type && previousBlockData.type === 'border-block-image')
      ) {
        end = '</div><!--rm--></div><!--rm-->';
      } else {
        end = '</div>';
      }
      return { start, end }
    }
```

## 4.添加control指令

这一步相对比较简单，就是在编辑器的工具栏添加一个子女的工具控件。

```js
{
    type: 'control',
    includeEditors,
    excludeEditors,
    control: () => {
      return {
        key: 'custom-border',
        type: 'block-type',
        title: '边框',
        text: '边框',
        command: 'border-block'
      }
    }
  }
```

有一个很重要的点，是我之前忽略的地方。我们在最后使用这个拓展时，需要调用编辑器的`use`方法，而在调用这个方法的时候，相当于当前编辑器BraftEditor全局对象就包含了这个拓展控件，导致其他使用这个编辑器的页面也包含了这个拓展控件。

这种影响是比较危险的，因为虽然只是一个通用的扩展控件，但是在其他页面其他开发人员的代码是无法保证不会出现一些问题。所以在写拓展时，需要添加一个很重的参数：

`includeEditors`，`excludeEditors`

表明哪些编辑器实例要有，哪些编辑器实例不要。这两个参数是一个数组，里面就是每个编辑器的id。

## 5.添加指令拦截器

在实现这个自定义拓展时，发现原本编辑器对于不同的按键指令处理的方式不一样，比如：在边框里面回车会直接退出这个边框块换行，即光标会到边框的外面，而ctr+回车才会在边框内部换行，光标依旧在边框内部。

但是这种不符合我们的常规操作，所以需要定义一个拦截器，将其转换成我们习惯的处理方式：回车正常在边框内部换行，ctr+回车才会退出边框重新生成一块。

实现方法就是重新定义指令处理器和结束处理 `pop-interception`中的`handleReturn`和`handleKeyCommand`，相当于是一个拦截器，可以在中间做相关的自定义处理。

**返回类型**：`not-handled`会走默认处理，返回`handled`不会继续走后面的默认处理.

**注意**！处理的时候，**尽量缩小自定义的处理判断范围**，避免影响原来的处理逻辑，毕竟我们只是在这个大的工具基础上拓展一块小的特性功能。

其中：`handleReturn`处理**回车键**，`handleKeyCommand`处理**删除键和Tab键**

```js
export default ({includeEditors}) => {
  return [{
    type: 'control',
    includeEditors,
    ...
  }, {
    type: 'prop-interception',
    includeEditors,
    interceptor: (editorProps) => {
      editorProps.handleReturn = handleReturn(editorProps.handleReturn);
      editorProps.handleKeyCommand = handleKeyCommand(editorProps.handleKeyCommand);
      return editorProps;
    }
  }, {
    type: 'block',
    name: 'border-block',
    includeEditors,
    renderMap: getBorderBlockRenderMap(),
    importer: (nodeName, node) => {
        ...
    },
    exporter: (contentState, block) => {
        ...
    }
  }]
}
```

## 拓展使用

```js
// BlockBorderExtension.js
export default(option: {includeEditors}){
	return [{
    type: 'control',
    includeEditors: includeEditors,
    ...
  }]
```

```js
// ContentEdit.js
BraftEditor.use(BlockBorderExtension({
  includeEditors: ['editor-with-border']
}));

<BraftEditor
    id="editor-with-border"
    ...
/>
```

最后的效果：

![WechatIMG627.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5add49d2d7b4447d88730cfc4345ddc9~tplv-k3u1fbpfcp-watermark.image?)

# 问题总结

1.  多个待过滤标签会匹配到第一个和最后一个，导致中间内容全部被清空，这种情况应该如何处理，即：怎么样让正则从头到尾匹配，匹配到第一个清空后还会继续匹配后面的内容？

    项目里，因为特殊性，直接匹配标签内容，如果是`</div>`这种，就在外面包一层特殊tag然后全局匹配即可。这样**通用性不强**，需要**手动添加tag**，目前没有很好的其他实现方式。

2.  `Braft Editor`创建初始化时如何将最新传入的`content html`字符串放入`createEditor`参数中，但怎么样又能防止组件`render`导致编辑器多次`createEditor`而引起的页面卡顿？

    父组件使用的form表单，正常逻辑进入编辑，会传入一个`currentItem` 代表之前最新编辑内容，充当每个表单项的`initialValue`，编辑器组件作为控制组件，需要保证`content`初始化成功。

    如果以`props`形式直接父传子，无法将最新`content`传递给它，因为在初始化阶段编辑器拿到是`null` ，进行`rerender`时才会拿到最新内容，但如果在`render`阶段拿到最新值并调用`createEditor`，会导致多次重复调用引起页面卡顿等问题。

    为了能拿到最新的`content`内容值，又避免渲染导致的多次`createEditor`，可以通过`ref`的方式，让父组件手动去调用编辑组件的`create`方法，这样就能保证只调用一次，且拿到的内容是最新值。

3.  因为编辑器插件相对比较灵活，如果是自己添加自定义的一个单独类型，需要考虑自己在做自定义处理时会不会有`null`或者数据类型这种异常，防止页面突然报错白屏，保险起见，我们在做一些特殊数据处理时最好添加`try...catch`。

4.  在处理操作拦截处理时，需要考虑到很多边界情况，比如是否是以新类型开始，结束等等，这里就需要从代码以及操作的角度同步去考虑各种边界情况。

5.  引入自定义拓展时，要注意过滤id。使用自定义扩展时，使用`BraftEditor.use`方法，提前在组件外引用，如果不加`includeEditors`，就会默认全局对象引用。

    **这样会导致在其他页面内基于BrafEditor的编辑器也会出现这个自定的扩展控件！！！记得加id屏蔽掉！！！！**

# 最后

这一次的实现虽然能够基本满足需要的功能，但是在特定标签处理那块还没有产出一个更好的实现方式。这样在和其他端对接的时候，需要花更多的精力去做一些兼容的处理，这是目前这边做的不太好的地方。但是整体上已经理清了整个自定义拓展的实现流程和相关的一些内部的实现细节。这次的功能相对来说还不是特别完善，后续有时间还需要在进一步研究。

参考链接：

[BraftEditor在线文档](https://www.yuque.com/braft-editor/be)

[BraftEditor在线Demo](https://braft.margox.cn/demos/basic)

[BraftEditor源码](https://github.com/margox/braft-extensions/tree/master)

[DraftJS在线文档](https://draftjs.org/docs/getting-started)
