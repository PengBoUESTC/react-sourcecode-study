## 概要
本文将对`react`项目初始化流程进行简要的分析。我会在源码分析过程中将开发调试相关代码移除，以便于理清代码的主体脉络。

在初始化`react`项目时，我们首先会通过`createRoot`来创建一个`react`组件根容器
```javascript
const root = createRoot(domNode, options?)
```

其中`createRoot` 方法为 `react-dom` 中暴露的方法。其实现代码如下：
```javascript
export function createRoot(
  container: Element | Document | DocumentFragment,
  options?: CreateRootOptions,
): RootType {
  // 这里删除了 开发相关代码逻辑
  try {
    // 移除开发相关代码之后，
    // 能看到 createRoot 其实是对 createRootImpl 的一层包装，目的就是注入开发调试代码
    return createRootImpl(container, options);
  } finally {
  }
}
```

`createRootImpl` 真正的逻辑在 `react-dom/client/ReactDom`中，具体实现如下：
```javascript
// 同样为了 便于分析具体逻辑将其中的 开发调试环境相关代码移除了
export function createRoot(
  container: Element | Document | DocumentFragment,
  options?: CreateRootOptions,
): RootType {
  // 对用户传入根节点类型验证，
  if (!isValidContainer(container)) {
    throw new Error('createRoot(...): Target container is not a DOM element.');
  }
  
  // 严格模式
  let isStrictMode = false;
  let concurrentUpdatesByDefaultOverride = false;
  let identifierPrefix = '';
  let onRecoverableError = defaultOnRecoverableError;
  let transitionCallbacks = null;

  // 根据用户传入的 配置选项进行 处理，用于生成 FiberRoot
  // 由于 options 参数是可选的，因此 可以先不用关注这些 选项
  if (options !== null && options !== undefined) {
    if (options.unstable_strictMode === true) {
      isStrictMode = true;
    }
    if (
      allowConcurrentByDefault &&
      options.unstable_concurrentUpdatesByDefault === true
    ) {
      concurrentUpdatesByDefaultOverride = true;
    }
    if (options.identifierPrefix !== undefined) {
      identifierPrefix = options.identifierPrefix;
    }
    if (options.onRecoverableError !== undefined) {
      onRecoverableError = options.onRecoverableError;
    }
    if (options.unstable_transitionCallbacks !== undefined) {
      transitionCallbacks = options.unstable_transitionCallbacks;
    }
  }

  // 生成 FiberRoot
  const root = createContainer(
    container,
    ConcurrentRoot,
    null,
    isStrictMode,
    concurrentUpdatesByDefaultOverride,
    identifierPrefix,
    onRecoverableError,
    transitionCallbacks,
  );
 
  // 将 生成的 FiberRoot 节点 绑定到 dom 根节点上， container[unique_key] = root.current
  markContainerAsRoot(root.current, container);
  // ReactDOMClientDispatcher 中包含 prefetchDNS preconnect preload preinit ，方法，
  // 通过 全局变量 Dispatcher 暴露给用户
  // 使用全局变量 + 引用的方式，主要目的在于可以动态的 更改 变量中引用的值
  // 后续很重要的 hook 逻辑也是通过这种全局变量 + 引用的 方式 实现的 （ReactCurrentDispatcher.current）
  Dispatcher.current = ReactDOMClientDispatcher;

  const rootContainerElement: Document | Element | DocumentFragment =
    container.nodeType === COMMENT_NODE
      ? (container.parentNode: any)
      : container;
  // 通过事件代理的方式 将 react 下所有的时间都绑定到 react 根节点 dom 元素上
  listenToAllSupportedEvents(rootContainerElement);
  // 生成 根 节点， 暴露 render/unmount 方法， 分别用于 渲染/卸载 react 实例 
  return new ReactDOMRoot(root);
}
```

通过 `createRoot` 生成根节点之后，用户可以通过调用 `render` 方法来将 `react` 组件渲染到根节点 `dom` （`container`）上。

## `ReactDOMRoot`
其中 `createRoot` 过程中有两个比较重要的环节，分别是 `fiberRoot` （`createContainer`）生成与事件代理（`listenToAllSupportedEvents`）。由于这篇文章的目的是理清`react`组件的初始化挂载流程，因此这里先不深入讨论这两个流程。让我们把注意力集中到`ReactDOMRoot`实例中的 `render` 方法上。
```javascript
// 存储 根fiber fiberRoot
function ReactDOMRoot(internalRoot: FiberRoot) {
  this._internalRoot = internalRoot;
} 

function (children: ReactNodeList): void {
   const root = this._internalRoot;
    //  移除大量开发调试代码后，只剩下 以下两个流程
    // 1: 根节点判空处理
   if (root === null) {
      throw new Error('Cannot update an unmounted root.');
    }
    // 更新 根节点 dom
    updateContainer(children, root, null, null);
 };
 ```
因此react组件的初次挂载也被视为一次组件的更新操作。 `updateContainer` 流程将会在后续的文章中进行分析。整体流程经过上述流程的分析，我们已经对 react项目的初始化流程有了初步的了解，以下是上述代码的流程：
![`react` 初始化流程](../react-%E7%AC%AC%202%20%E9%A1%B5.png)

图1： `react` 初始化流程

