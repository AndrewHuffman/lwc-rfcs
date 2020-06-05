---
title: LWC server runtime
status: DRAFTED
created_at: 2020-01-06
updated_at: 2020-01-06
pr: https://github.com/salesforce/lwc-rfcs/pull/23
---

# LWC server runtime

## Summary

The LWC engine has been designed from the beginning to run in an environment with access to the DOM APIs. To accommodate server-side rendering (SSR) requirements, we need a way to decouple the engine from the DOM APIs. The existing `@lwc/engine` will be replaced by 3 new packages:

-   `@lwc/engine-core` exposes platform-agnostic APIs and will be used by the different runtime packages to share the common logic.
-   `@lwc/engine-dom` exposes LWC APIs available on the browser.
-   `@lwc/engine-server` exposes LWC APIs used for server-side rendering.

> > TODO: Add more details

## Goals

Server Side Rendering (SSR) is a popular technique for rendering pages and components that would usually run in a browser on the server. SSR takes as input a root component and a set of properties and renders an HTML fragment and optionally one or multiple stylesheets. This technique can be used in many different ways: improve initial page render time, generate static website, …

Due to the size and complexity of this feature, the rollout of SSR is broken up in 3 different phases and is spread over multiple releases:

-   **Phase 1:** Decouple the LWC engine from the DOM and allow it to run in a JavaScript environment without access to the DOM APIs (like Node.js). As part of this phase the LWC server engine will also generate a prototype version of the serialized HTML (Phase 2).
-   **Phase 2:** Settle on a HTML serialization format
-   **Phase 3:** Rehydrate in a browser the DOM tree resulting of the serialization into an LWC component tree

As part of phase 1, the goal is to provide a capability for LWC to render components transparently on the server. From the component stand point, the LWC SSR and DOM versions provide identical APIs and will run the component in the same fashion on both platforms.

## Proposal Scope

This proposal will be only focussing on the phase 1 of the SSR project. The phase 2 and 3 will be covered in subsequent proposals. This proposal cover 2 main aspects of the LWC SSR:

-   What are the differences between the DOM version and the SSR version of LWC?
-   How to create 2 different implementation of the LWC engine while keeping the core logic shared.

## Detailed design

### Approaching SSR in UI framework

Many popular UI framework already provide options to enable server side rendering. There is no single way to implement SSR in a UI framework and each framework approach this in a different way. That being said all the major frameworks converge around 2 main approaches.

The first approach is to **polyfill the server environment with DOM APIs and to reuse same core UI framework code on the server and on the client**. This is the approach [Stencil](https://github.com/ionic-team/stencil/master/tree/src/mock-doc), [Ember](https://github.com/ember-fastboot/simple-dom) and [lit-ssr](https://github.com/PolymerLabs/lit-ssr).

This approach is really convenient because it require almost no change to the core UI framework to enable SSR. Since most of the DOM APIs used by UI framework are low level dom API, most of them are easy to polyfill. Popular DOM APIs implementations like [domino](https://github.com/fgnass/domino) or [jsdom](https://github.com/jsdom/jsdom) can be used to avoid having to write to polyfills.

This approach also suffers from multiple drawback. The main issue is that the polyfill has to stay in sync with the core UI framework. As the core UI framework evolves and the DOM APIs used by the engine changes, developers has to always double check that the newly introduced APIs are also polyfilled properly. The other main drawback, is that by applying polyfills to evaluate the engine the same DOM interfaces and methods a also from the component code (eg. `window`, `document`, ...) which gives a false senses that the component is running in the browser.

Using the polyfilling approach, the existing `@lwc/engine` stays untouched and the entry point of the SSR version of LWC of would look something like this.

**`index.ts`:**

```ts
// Evaluate fist the DOM polyfills, then the DOM engine version of the engine. Reexport all the
// exported properties from the DOM engine.
import './dom-environment';
export * from '@lwc/engine';
```

**`dom-environment.ts`:**

```ts
class Node {
    children: Node[] = [];

    insertBefore(child: Node, anchor: Node): void {
        const anchorIndex = this.children.indexOf(anchor);

        if (anchorIndex === -1) {
            this.children.push(child);
        } else {
            this.children.splice(anchorIndex, 0, child);
        }
    }
}

class Element extends Node {
    tagName: string;

    constructor(tagName: string) {
        super();
        this.tagName = tagName;
    }
}

const document = {
    createElement(tagName: string): Element {
        return new Element(tagName);
    },
};

const window = {
    document,
};

// Assigning the synthetic DOM APIs to the current environment global object.
Object.assign(globalThis, {
    window,
    document,
    Node,
    Element,
});
```

The second approach is to **abstract the DOM APIs and the core UI framework**. This approach is used by [React](https://github.com/facebook/react/tree/master/packages/react-dom), [Vue](https://github.com/vuejs/vue-next/tree/master/packages/runtime-dom) and [Angular](https://github.com/angular/angular/tree/master/packages/platform-browser/src). This approach consists in creating an indirection between the rendering APIs and the core framework code. The indirection is injected at runtime depending on the environment. When loaded in the browser the rendering APIs create DOM `Element` and set DOM `Attribute` on the elements. While when loaded server the rendering APIs are manipulating string to create the serialized version of the components on the fly.

When used with a type-safe language like TypesScript, it is possible to ensure at compile time that all the rendering APIs are fulfilled in all the environments. By injecting at runtime this indirection, it also ensure that component code don't have access to any DOM APIs when running on the server.

The main drawback with this approach is the amount of refactor that need to happen in LWC engine code specifically. So far the LWC engine has been designed to run in JavaScript environment with direct access to the DOM APIs. This means that if we were to adopt this approach a large chunk of the LWC engine code will need to be rewritten. For example some of the LWC engine code store a reference for DOM interfaces (eg. `HTMLElement.prototype`, `Node.prototype`) at evaluation time. This will not be possible with this approach as the APIs will be injected into the core engine after its evaluation.

In order to inject the rendering APIs depending on the environment, we would need to create a different entry point per environment. You can find below some pseudo code for each entry point. The 2 entry points uses the same `createComponent` abstraction to create an LWC component but pass a renderer specific to the targeted environment.

**`entry-points/dom.js`:**

```ts
import { LightningElement, createComponent } from '@lwc/engine-core';

const domRenderer = {
    createElement(tagName: string): Element {
        return document.createElement(tagName);
    },
    insertBefore(node: Node, parent: Element, anchor: Node): void {
        return parent.insertBefore(node, anchor);
    },
};

// The method dedicated to create an LWC component in a dom environment. It returns a DOM element
// that can then inserted into the document to render.
export function createElement(
    tagName: string,
    opts: { Ctor: typeof LightningElement },
): Element {
    const elm = document.createElement(tagName);
    createComponent(elm, Ctor, domRenderer);
    return elm;
}
```

**`entry-points/server.js`:**

```ts
import { LightningElement, createComponent } from '@lwc/engine-core';

interface ServerNode {
    children: ServerNode[];
}

interface ServerElement extends ServerNode {
    tagName: string;
}

const serverRenderer = {
    createElement(tagName: string): ServerElement {
        return {
            tagName,
            children: [],
        };
    },
    insertBefore(
        node: ServerNode,
        parent: ServerElement,
        anchor: ServerNode,
    ): void {
        const anchorIndex = parent.children.indexOf(anchor);

        if (anchorIndex === -1) {
            parent.children.push(child);
        } else {
            parent.children.splice(anchorIndex, 0, child);
        }
    },
};

// The method dedicated to create an LWC component in a server environment. It returns a string
// resulting of the serialization component tree
export function renderComponent(
    tagName: string,
    opts: { Ctor: typeof LightningElement },
): string {
    const elm: ServerElement = {
        tagName,
        children: [],
    };

    createComponent(elm, Ctor, serverRenderer);

    return serializeElement(elm);
}
```

**Conclusion:** After weighting the pros and cons of the 2 approaches described above, **we decided to go with the second approach when the rendering APIs are injected lazily at runtime**.

### Splitting `@lwc/engine`

As discussed in the previous section, an abstraction of the the underlying rendering APIs into the core framework depending on the target environment. In order to share the core UI framework between the different environment the existing `@lwc/engine` will be split into 3 packages.

#### `@lwc/engine-core`

This package contains core logic shared by the different runtime environment including the rendering engine and the reactivity mechanism. It should never be consumed directly in an application. It only provides internal APIs for building custom runtimes. This packages exposes the following platform-agnostic public APIs:

-   `LightningElement`
-   `api`
-   `track`
-   `readonly`
-   `wire`
-   `setFeatureFlag`
-   `getComponentDef`
-   `isComponentConstructor`
-   `getComponentConstructor`
-   `unwrap`
-   `registerTemplate`
-   `registerComponent`
-   `registerDecorators`
-   `sanitizeAttribute`

On top of the public APIs, this package also exposes new internal APIs that are meant to be consumed 

-   `getComponentInternalDef(Ctor: typeof LightningElement): ComponentDef`: Get the internal component definition for a given LightningElement constructor.
-   `createVM(elm: HostElement, Ctor, options: { mode: 'open' | 'closed', owner: VM | null, renderer: Renderer }): VM`: Create a new View-Model (VM) associated with an LWC component.
-   `connectRootElement(elm: HostElement): void`: Mount a component and trigger a rendering.
-   `disconnectRootElement(elm: HostElement): void`: Unmount a component and trigger a disconnection flow.
-   `getAssociatedVMIfPresent(elm: HostElement): VM | undefined`: Retrieve the VM on a given element.
-   `setElementProto(elm: HostElement): void`: Patch an element prototype with the bridge element.

The DOM APIs used by the rendered engine are injected by the runtime depending on the environment. The list of all the current DOM APIs the engine depends upon can be found in the [DOM APIs usage](#dom-apis-usage). The [Renderer](#renderer-interface) interface will replace those direct invocation to the DOM API usages.

#### `@lwc/engine-dom`

Runtime that can be used to render LWC component trees in a DOM environment. This package exposes the following APIs:

-   `createElement` + reaction hooks
-   `isNodeFromTemplate`

This package injects the native DOM APIs into the `@lwc/engine-core` rendering engine.

#### `@lwc/engine-server`

Runtime that can be used to render LWC component trees as strings. This package exposes the following API:

-   `renderComponent(name: string, ctor: typeof LightningElement, props?: Record<string, any>): string`: This method creates an renders an LWC component synchronously to a string.

When a component is rendered using `renderComponent`, the following restriction applies:

-   each created component will execute the following life-cycle hooks once `constructor`, `connectedCallback` and `render`
-   properties and methods annotated with `@wire` will not be invoked
-   component reactivity is disabled

Another API might be created to accommodate wire adapter invocation and asynchronous data fetching on the server.

This package injects mock DOM APIs in the `@lwc/engine-core` rendering engine. Those DOM APIs produce a lightweight DOM structure. This structure can then be serialized into a string containing the HTML serialization of the element's descendants. As described in the Appendix, this package is also in charge of attaching on the global object a mock `CustomEvent` and only implements a restricted subset of the event dispatching algorithm on the server (no bubbling, no event retargeting).

The existing `lwc` package stays untouched and will be used to distribute the different versions of the engine. From the developer perspective, the experience writing a component remains identical.

**`c/app/app.js`:**

```js
import { LightningElement } from 'lwc';

export default class App extends LightningElement {}
```

**`client.js`:**

```js
import { createElement } from 'lwc'; // Resolves to `@lwc/engine-dom`
import App from 'c/app';

const app = createElement('c-app', { is: App });
document.body.appendChild(app);
```

**`server.js`:**

```js
import { renderComponent } from '@lwc/engine-server'; // Resolves to `@lwc/engine-server`
import App from 'c/app';

const str = renderComponent('c-app', { is: App });
console.log(str);
```

## Drawbacks

-   Requires a complete rewrite of the current `@lwc/engine`
-   Might require substantial changes to the existing tools (jest preset, rollup plugins, etc.) to load the right engine depending on the environment.

## Alternatives

### Using a DOM implementation in JavaScript before evaluating the engine

Prior the evaluation of the LWC engine, we would evaluate a DOM implementation write in JavaScript (like [jsdom](https://github.com/jsdom/jsdom) or [domino](https://github.com/fgnass/domino)). The compelling aspect of this approach is that it requires almost no change to the engine to work.

There are multiple drawbacks with this approach:

-   Performance: The engine only relies on a limited set of well-known APIs, leveraging a full DOM implementation for SSR would greatly reduce the SSR performance.
-   Predictability: By attaching the DOM interfaces on the global object, those APIs are not only exposed to the engine but also to the component author. Exposing such APIs to the component author might bring unexpected behavior when component code runs on the server.

## How we teach this

This proposal breaks up the existing `@lwc/engine` package into multiple packages. Because of this APIs that used to be imported from `lwc` might not be present anymore. With this proposal, the `createElement` API will be moving to `@lwc/engine-dom`. This change constitutes a breaking change by itself, so we need to be careful how this change will be released. The good news is that `createElement` is considered as an experimental API and should only used to create the application root and to test components.

Changing the application root creation consists in a one line change. Therefor it should be pretty straightforward updated as long as it is well documented. Updating all the usages of `createElement` in test will probably require more time. For testing purposes, the `lwc` module will be mapped to the `@lwc/engine-dom` instead of `@lwc/engine-core` for now. We will also add warning messages in the console to promulgate the new pattern. A codemod for test files can also be used rename the `lwc` import in the test files to `@lwc/engine-dom`.

Updating the documentation for the newly added server only APIs should be enough.

## Unresolved questions

-   **Remove `@lwc/engine-core` on TypeScript `dom` library?** The `@lwc/engine-core` package relies heavily on the ambient DOM TypeScript interfaces provided by the `dom` library. To ensure that the `@lwc/engine-core` is not leveraging any of the DOM APIs we will need to remove the `dom` lib from the `tsconfig.json`. It is currently unclear how all the ambient types can be removed on this package while ensuring the type safety.
-   **How to implement LWC Context for SSR?** Context uses eventing for registration between the provider and the consumer. Since `@lwc/engine-server` will only implement a subset of the DOM eventing, we will need to evaluate how we can replace the current registration mechanism.

---

## Appendix

### DOM APIs usage

We can break-down the current LWC DOM usages into 3 different categories:

-   [DOM constructors used by the engine](#dom-constructors-used-by-the-engine)
-   [DOM methods and accessors used by the engine](#dom-methods-and-accessors-used-by-the-engine)
-   [DOM constructors used by component authors](#dom-constructors-used-by-component-authors)

#### DOM constructors used by the engine

##### `HTMLElement`

-   **DOM usage:** Used by the engine to extract the descriptors and reassign them to the `LightningElement` prototype.
-   **SSR usage:** 🔵 NOT REQUIRED
-   **Observations:** We can create a hard-coded list of all the needed accessors: HTML global attributes, aria reflection properties and HTML the good part.

#### DOM methods and accessors used by the engine

On top of this the engine also rely on the following DOM methods and accessors:

##### `EventTarget.prototype.dispatchEvent()`

-   **DOM usage:** Exposed via `LightningElement.prototype.dispatchEvent()`
-   **SSR usage:** 🔴 REQUIRED
-   **Observations:** Components may dispatch event once connected.

##### `EventTarget.prototype.addEventListener()`

-   **DOM usage:** Exposed via `LightningElement.prototype.addEventListener()`. Used by the rendering engine to handle `on*` directive from the template.
-   **SSR usage:** 🔴 REQUIRED

##### `EventTarget.prototype.removeEventListener()`

-   **DOM usage:** Exposed via `LightningElement.prototype.removeElementListener()`.
-   **SSR usage:** 🔴 REQUIRED

##### `Node.prototype.appendChild()`

-   **DOM usage:** Used by the upgrade mechanism, synthetic shadow styling and to enforce restrictions.
-   **SSR usage:** 🔵 NOT REQUIRED
-   **Observations:** This API can be replaced by `Node.prototype.insertBefore(elm, null)`.

##### `Node.prototype.insertBefore()`

-   **DOM usage:** Used by the upgrade mechanism, by the rendering engine, and to enforce restrictions.
-   **SSR Usage:** 🔴 REQUIRED

##### `Node.prototype.removeChild()`

-   **DOM usage:** Used by the upgrade mechanism, by the rendering engine, and to enforce restrictions.
-   **SSR Usage:** 🔴 REQUIRED

##### `Node.prototype.replaceChild()`

-   **DOM usage:** Used by the upgrade mechanism, and to enforce restrictions
-   **SSR Usage:** 🔴 REQUIRED

##### `Node.prototype.parentNode` (getter)

-   **DOM usage:** Used to traverse the DOM tree
-   **SSR Usage:** 🔶 MIGHT BE REQUIRED
-   **Observations:** Depending on how the error boundary is implemented, we might be able to get rid of this API.

##### `Element.prototype.hasAttribute()`

-   **DOM usage:** Used by the aria reflection polyfill
-   **SSR Usage:** 🔵 NOT REQUIRED
-   **Observations:** For SSR, we will not need this polyfill

##### `Element.prototype.getAttribute()`

-   **DOM usage:** Exposed via `LightningElement.prototype.getAttribute()`. Used by the aria reflection polyfill
-   **SSR Usage:** 🔶 MIGHT BE REQUIRED
-   **Observations:** We might use `Element.prototype.getAttributeNS(null, name)` to reduce the API surface needed by the engine.

##### `Element.prototype.getAttributeNS()`

-   **DOM usage:** Exposed via `LightningElement.prototype.getAttributeNS()`
-   **SSR Usage:** 🔴 REQUIRED

##### `Element.prototype.setAttribute()`

-   **DOM usage:** Exposed via `LightningElement.prototype.setAttribute()`. Used by the rendering engine, for synthetic shadow styling and by the aria reflection polyfill.
-   **SSR Usage:** 🔶 MIGHT BE REQUIRED
-   **Observations:** We might use `Element.prototype.setAttributeNS(null, name)` to reduce the API surface needed by the engine.

##### `Element.prototype.setAttributeNS()`

-   **DOM usage:** Exposed via `LightningElement.prototype.setAttributeNS()`. Used by the rendering engine.
-   **SSR Usage:** 🔴 REQUIRED

##### `Element.prototype.removeAttribute()`

-   **DOM usage:** Exposed via `LightningElement.prototype.removeAttribute()`. Used by the rendering engine, for the synthetic shadow styling and by the aria reflection polyfill
-   **SSR Usage:** 🔶 MIGHT BE REQUIRED
-   **Observations:** We might use `Element.prototype.removeAttributeNS(null, name)` to reduce the API surface needed by the engine.

##### `Element.prototype.removeAttributeNS()`

-   **DOM usage:** Exposed via `LightningElement.prototype.removeAttributeNS()`.
-   **SSR Usage:** 🔴 REQUIRED

##### `Element.prototype.getElementsByTagName()`

-   **DOM usage:** Exposed via `LightningElement.prototype.getElementsByTagName()`.

-   **SSR Usage:** 🔵 NOT REQUIRED
-   **Observations:** This method is only exposed to select elements from the component light DOM, which is only available from the `renderedCallback`. Since SSR doesn't run `renderedCallback`, we can always returns an empty `HTMLCollection` when running on the server.

##### `Element.prototype.getElementsByClassName()`

-   **DOM usage:** Exposed via `LightningElement.prototype.getElementsByClassName()`.
-   **SSR Usage:** 🔵 NOT REQUIRED
-   **Observations:** This method is only exposed to select elements from the component light DOM, which is only available from the `renderedCallback`. Since SSR doesn't run `renderedCallback`, we can always returns an empty `HTMLCollection` when running on the server.

##### `Element.prototype.querySelector()`

-   **DOM usage:** Exposed via `LightningElement.prototype.querySelector()`. Used to enforce restrictions.
-   **SSR Usage:** 🔵 NOT REQUIRED
-   **Observations:** This method is only exposed to select elements from the component light DOM, which is only available from the `renderedCallback`. Since SSR doesn't run `renderedCallback`, we can always returns an empty `NodeList` when running on the server.

##### `Element.prototype.querySelectorAll()`

-   **DOM usage:** Exposed via `LightningElement.prototype.querySelectorAll()`. Used to enforce restrictions.
-   **SSR Usage:** 🔵 NOT REQUIRED
-   **Observations:** This method is only exposed to select elements from the component light DOM, which is only available from the `renderedCallback`. Since SSR doesn't run `renderedCallback`, we can always returns an empty `NodeList` when running on the server.

##### `Element.prototype.getBoundingClientRect()`

-   **DOM usage:** Exposed via `LightningElement.prototype.getBoundingClientRect()`.
-   **SSR Usage:** 🔵 NOT REQUIRED
-   **Observations:** Running a layout engine during SSR, is a complex task that will not bring much values to component authors. Returning an empty `DOMRect` (where all the properties are set to `0`), might be the best approach here.

##### `Element.prototype.classList` (getter)

-   **DOM usage:** Exposed via `LightningElement.prototype.classList`. Used by the rendering engine.
-   **SSR Usage:** 🔴 REQUIRED

##### HTML global attributes (setters and getters) and Aria reflection properties

-   **DOM usage:** Exposed on the `LightningElement.prototype` Used by the rendering to set the properties on custom elements.
-   **SSR Usage:** 🔴 REQUIRED

#### DOM constructors used by component authors

Finally, on top of all the APIs used by the engine to evaluate and run, component authors need to have access to the following DOM constructors in the context of SSR.

##### `CustomEvent`

-   **DOM usage:** Used to dispatch events
-   **SSR Usage:** 🔴 REQUIRED

##### `Event`

-   **DOM usage:** Used to dispatch events
-   **SSR Usage:** 🔶 MIGHT BE REQUIRED
-   **Observations:** This might not be needed because `CustomEvent` inherits from `Event` and because `CustomEvent` is the recommended way to dispatch non-standard events.

### Renderer interface

```ts
export interface Renderer<HostNode, HostElement> {
    syntheticShadow: boolean;
    insert(node: HostNode, parent: HostElement, anchor: HostNode | null): void;
    remove(node: HostNode, parent: HostElement): void;
    createElement(tagName: string, namespace?: string): HostElement;
    createText(content: string): HostNode;
    nextSibling(node: HostNode): HostNode | null;
    attachShadow(element: HostElement, options: ShadowRootInit): HostNode;
    setText(node: HostNode, content: string): void;
    getAttribute(
        element: HostElement,
        name: string,
        namespace?: string | null,
    ): string | null;
    setAttribute(
        element: HostElement,
        name: string,
        value: string,
        namespace?: string | null,
    ): void;
    removeAttribute(
        element: HostElement,
        name: string,
        namespace?: string | null,
    ): void;
    addEventListener(
        target: HostElement,
        type: string,
        callback: EventListenerOrEventListenerObject,
        options?: AddEventListenerOptions | boolean,
    ): void;
    removeEventListener(
        target: HostElement,
        type: string,
        callback: EventListenerOrEventListenerObject,
        options?: AddEventListenerOptions | boolean,
    ): void;
    dispatchEvent(target: HostNode, event: Event): boolean;
    getClassList(element: HostElement): DOMTokenList;
    getStyleDeclaration(element: HostElement): CSSStyleDeclaration;
    getBoundingClientRect(element: HostElement): ClientRect;
    querySelector(element: HostElement, selectors: string): HostElement | null;
    querySelectorAll(element: HostElement, selectors: string): NodeList;
    getElementsByTagName(
        element: HostElement,
        tagNameOrWildCard: string,
    ): HTMLCollection;
    getElementsByClassName(element: HostElement, names: string): HTMLCollection;
    isConnected(node: HostNode): boolean;
    tagName(element: HostElement): string;
}
```
