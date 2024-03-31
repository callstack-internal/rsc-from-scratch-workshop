# React Server Components from Scratch

## Agenda

**Total time**: 3h

**Introduction**: 5m

**1st part**: Sections 1-4 (1h 40m)

- Presentation: 5m
- Q&A: 5m
- Implementation: 15m

**Break**: 15m

**2nd part:** Section 5 (1h)

- Presentation: 10m
- Q&A: 10m
- Implementation: 40m

## Introduction

### What are React Server Components?

- Render only on the server
- Not interactive (no hooks, event handlers, etc.)
- JavaScript is not sent to the client

### Differences from Client Components

| Client Components                          | Server Components                 |
| ------------------------------------------ | --------------------------------- |
| Traditional React components               | New type of component             |
| Render on the client (and server with SSR) | Render on the server only         |
| Cannot be async                            | Can be async                      |
| Interactive and stateful                   | Not interactive and stateless     |
| Cannot render server components            | Can render client components      |
| Marked with `'use client'` directive       | All components are RSC by default |

### Benefits

- Data fetching on the server
- Can contain sensitive logic
- Less JavaScript sent to the client

### Example

```jsx
export default async function Profile({ id }) {
  const data = await fetchProfileData(id);

  return (
    <div>
      <h1>{data.name}</h1>
      <p>{data.bio}</p>
    </div>
  );
}
```

## Workshop

### 1. First steps

- Basic server-side rendered multi-page app
- Pages don't include any JavaScript
- Clicking a link reloads the page

<https://github.com/callstack-internal/rsc-from-scratch-workshop/assets/1174278/1edf8367-d991-4713-a648-e880f5b57058>

#### Environment setup

- Node.js 20
- Yarn 1

#### Project setup

1. Clone and set up the starter repo.

   Clone with SSH:

   ```sh
   git clone --single-branch --branch starter git@github.com:callstack-internal/rsc-from-scratch-workshop.git
   ```

   Or with HTTPS:

   ```sh
   git clone --single-branch --branch starter https://github.com/callstack-internal/rsc-from-scratch-workshop.git
   ```

   Change the branch to `main` and remove the remote origin:

   ```sh
   git branch -M main
   git remote remove origin
   ```

   Create your own repository on GitHub and push the code to it to start working on it.

2. Install the dependencies by running:

   ```sh
   yarn
   ```

3. Start the server to see the app running:

   ```sh
   yarn start
   ```

   Then open <http://localhost:5172> in your browser.

#### Project Overview

##### File structure

```sh
.
├── src                   # Source code
│   ├── app               # React components
│   │   ├── _document.jsx # Wrapper for all pages
│   │   ├── index.jsx     # Home page (`/`)
│   │   └── pokemon.jsx   # Pokemon page (`/pokemon?name=bulbasaur`)
│   └── server.jsx        # Server entry point
├── README.md
├── package.json
├── tsconfig.json
└── yarn.lock
```

##### Dependencies

- [`tsx`](https://github.com/privatenumber/tsx) for watching, compiling and running the server
- [`koa`](https://koajs.com) for the server with [`@koa/router`](https://github.com/koajs/router) for server-side routing
- [`react`](https://react.dev) and [`react-dom`](https://react.dev/reference/react-dom) for rendering components.

##### Server-side routing

When a page is requested, the router loads the corresponding component and renders it to HTML:

```js
router.get('/:path*', async (ctx) => {
  const filepath = `./app/${ctx.params.path ?? 'index'}.jsx`;

  if (!(await stat(join(import.meta.dirname, filepath)))) {
    ctx.status = 404;
    return;
  }

  const { default: Document } = await import('./app/_document.jsx');
  const { default: Page } = await import(filepath);

  const html = ReactDOMServer.renderToString(
    <Document>
      <Page query={ctx.request.query} />
    </Document>
  );

  ctx.type = 'text/html';
  ctx.body = html;
});
```

### 2. Client-side navigation

- SPAs don’t reload the page when navigating
- Client-side navigation will override browser navigation

#### What we’ll build

<https://github.com/callstack-internal/rsc-from-scratch-workshop/assets/1174278/c3b06001-0363-4f3e-8d9b-71c27c5aec6c>

#### How it’ll work

1. Load some JavaScript on the client

   - Create a file `src/client.js`
   - Update our router in `src/server.js` to serve this file

     ```jsx
     router.get('/client.js', (ctx) => {
       const stream = createReadStream(join(import.meta.dirname, ctx.path));

       ctx.type = 'text/javascript';
       ctx.body = stream;
     });
     ```

   - Add a script tag to the HTML we return

     ```js
     const clientHtml = `
       ${renderToString(jsx)}

       <script type="module" src="/client.js"></script>
     `;

     ctx.type = 'text/html';
     ctx.body = clientHtml;
     ```

2. In `src/client.js`, intercept navigation and dynamically replace the content

   - Intercept link clicks

     ```js
     document.addEventListener('click', (event) => {
       if (e.target.tagName !== 'A') {
         return;
       }

       if (e.metaKey || e.ctrlKey || e.shiftKey || e.altKey) {
         return;
       }

       const href = e.target.getAttribute('href');

       if (!href.startsWith('/')) {
         return;
       }

       // Prevent the navigation to the new page
       e.preventDefault();

       // Update the browser URL bar without a page refresh
       window.history.pushState(null, null, href);

       // We'll implement this function later
       navigate(href);
     });
     ```

   - Intercept browser back/forward buttons

     ```js
     window.addEventListener('popstate', () => {
       navigate(window.location.pathname);
     });
     ```

   - Fetch the new page and replace the content

     ```js
     async function fetchClientHTML(pathname) {
       const response = await fetch(pathname);
       return response.text();
     }

     let currentPathname = window.location.pathname;

     async function navigate(href) {
       currentPathname = pathname;

       const clientHtml = await fetchClientHTML(pathname);

       if (pathname === currentPathname) {
         document.body.innerHTML = clientHtml;
       }
     }
     ```

### 3. Rehydration

- Process of "attaching" React to existing HTML that was rendered on the server
- Necessary to make React features work, e.g. diffing the DOM, state, event handlers etc.
- Diffing the DOM to update only the parts that changed (reconciliation) preserves the existing state

#### What we’ll build

<https://github.com/callstack-internal/rsc-from-scratch-workshop/assets/1174278/a0d1642b-ae67-4abb-ab21-dc0e43240984>

#### How it’ll work

1. Include the JSX in the initial payload

   - Traverse the component tree to get a JSX representation:

     ```js
     const A = () => <div>Hello</div>;

     const B = () => (
       <main>
         <A />
       </main>
     );

     renderJSXToClientJSX(<B />); // <main><div>Hello</div></main>
     ```

     The JSX looks like this:

     ```js
     {
       $$typeof: Symbol.for('react.element'),
       type: 'main',
       props: {
         children: {
           $$typeof: Symbol.for('react.element'),
           type: 'div',
           props: {
             children: 'Hello',
           },
         },
       },
     }
     ```

     Implementation of `renderJSXToClientJSX`:

     ```js
     function renderJSXToClientJSX(jsx) {
       if (
         typeof jsx === 'string' ||
         typeof jsx === 'number' ||
         typeof jsx === 'boolean' ||
         jsx == null
       ) {
         return jsx;
       } else if (Array.isArray(jsx)) {
         return jsx.map((child) => renderJSXToClientJSX(child));
       } else if (jsx != null && typeof jsx === 'object') {
         if (jsx.$$typeof === Symbol.for('react.element')) {
           if (typeof jsx.type === 'string') {
             return {
               ...jsx,
               props: renderJSXToClientJSX(jsx.props),
             };
           } else if (typeof jsx.type === 'function') {
             const Component = jsx.type;
             const props = jsx.props;
             const returnedJsx = Component(props);

             return renderJSXToClientJSX(returnedJsx);
           } else {
             throw new Error('Not implemented.');
           }
         } else {
           return Object.fromEntries(
             Object.entries(jsx).map(([propName, value]) => [
               propName,
               renderJSXToClientJSX(value),
             ])
           );
         }
       } else {
         throw new Error('Not implemented');
       }
     }
     ```

   - Include the JSON representation of the JSX in the initial payload:

     ```js
     function respond(ctx, jsx) {
       const clientJSX = renderJSXToClientJSX(jsx);
       const clientJSXString = JSON.stringify(clientJSX);
       const clientHtml = html`
         ${renderToString(clientJSX)}

         <script>
           window.__INITIAL_CLIENT_JSX_STRING__ = ${JSON.stringify(
             clientJSXString
           ).replace(/</g, '\\u003c')};
         </script>
         <script type="module" src="/client.js"></script>
       `;

       ctx.type = 'text/html';
       ctx.body = clientHtml;
     }
     ```

   - JSX includes non-serializable values like symbols (e.g. `Symbol.for('react.element')`), so we need to replace them with a string representation (e.g. `'$RE'`):

     ```js
     function stringifyJSX(key, value) {
       if (value === Symbol.for('react.element')) {
         return '$RE';
       } else if (typeof value === 'string' && value.startsWith('$')) {
         return '$' + value;
       } else {
         return value;
       }
     }

     // ...

     const clientJSXString = JSON.stringify(clientJSX, stringifyJSX);
     ```

2. On the client, use the JSX to rehydrate the existing HTML

   - Load `react` and `react-dom` on the client by including them in the HTML in the response from the server:

     ```html
     <script>
       window.__INITIAL_CLIENT_JSX_STRING__ = ${JSON.stringify(
         clientJSXString
       ).replace(/</g, '\\u003c')};
     </script>
     <script type="importmap">
       {
         "imports": {
           "react": "https://esm.sh/react@canary",
           "react-dom/client": "https://esm.sh/react-dom@canary/client"
         }
       }
     </script>
     <script type="module" src="/client.js"></script>
     ```

   - Use `hydrateRoot` from `react-dom/client` to hydrate the existing HTML:

     ```js
     import { hydrateRoot } from 'react-dom/client';

     const root = hydrateRoot(document, getInitialClientJSX());

     function getInitialClientJSX() {
       const clientJSX = JSON.parse(
         window.__INITIAL_CLIENT_JSX_STRING__,
         parseJSX
       );

       return clientJSX;
     }

     function parseJSX(key, value) {
       if (value === '$RE') {
         return Symbol.for('react.element');
       } else if (typeof value === 'string' && value.startsWith('$$')) {
         return value.slice(1);
       } else {
         return value;
       }
     }
     ```

3. Instead of fetching HTML and overwriting DOM, fetch JSX and rerender content

   - Add the ability to fetch the JSX from the server with a query parameter in `respond`:

     ```js
     function respond(ctx, jsx) {
       const clientJSX = renderJSXToClientJSX(jsx);

       if (ctx.request.query.jsx != null) {
         const clientJSXString = JSON.stringify(clientJSX, stringifyJSX);

         ctx.type = 'application/json';
         ctx.body = clientJSXString;

         return;
       }

       // ...
     }
     ```

   - Replace the `fetchClientHTML` function with `fetchClientJSX`:

     ```js
     async function fetchClientJSX(pathname) {
       const url = new URL(pathname, location.href);

       url.searchParams.set('jsx', true);

       const response = await fetch(url.href);
       const clientJSXString = await response.text();
       const clientJSX = JSON.parse(clientJSXString, parseJSX);

       return clientJSX;
     }
     ```

   - Update the `navigate` function to fetch the JSX instead of the HTML:

     ```js
     async function navigate(pathname) {
       currentPathname = pathname;

       const clientJSX = await fetchClientJSX(pathname);

       if (pathname === currentPathname) {
         root.render(clientJSX);
       }
     }
     ```

### 4. Async Components

- Execute async components on the server

#### What we’ll build

<https://github.com/callstack-internal/rsc-from-scratch-workshop/assets/1174278/9e40b1f7-c818-4a43-956f-af4acf93a056>

#### How it will work

1. Make components async

- Change `Index` component to an `async` function

  ```js
  export default async function Index() {
    ...
  }
  ```

- Change hardcoded pokemon array to `fetch` call to API

  ```js
  const data = await fetch(
    'https://pokeapi.co/api/v2/pokemon?limit=10&offset=0'
  ).then((res) => res.json());
  ```

Repeat this step for `Pokemon` component. Use `https://pokeapi.co/api/v2/pokemon/${query.name}` endpoint.

2. Adjust `src/server.jsx` to handle `async` components - Make `respond`, `renderJSXToClientJSX`, and `Component` an `async` function, make sure to `await` them where necessary

```js
await respond(...);
```

```js
async function respond(ctx, jsx) {
  const clientJSX = await renderJSXToClientJSX(jsx);

  ...
}
```

```js
async function renderJSXToClientJSX(jsx) {
  if (
    typeof jsx === 'string' ||
    typeof jsx === 'number' ||
    typeof jsx === 'boolean' ||
    jsx == null
  ) {
    return jsx;
  } else if (Array.isArray(jsx)) {
    return Promise.all(jsx.map((child) => renderJSXToClientJSX(child)));
  } else if (jsx != null && typeof jsx === 'object') {
    if (jsx.$$typeof === Symbol.for('react.element')) {
      if (typeof jsx.type === 'string') {
        return {
          ...jsx,
          props: await renderJSXToClientJSX(jsx.props),
        };
      } else if (typeof jsx.type === 'function') {
        const Component = jsx.type;
        const props = jsx.props;
        const returnedJsx = await Component(props);

        return renderJSXToClientJSX(returnedJsx);
      } else {
        throw new Error('Not implemented.');
      }
    } else {
      return Object.fromEntries(
        await Promise.all(
          Object.entries(jsx).map(async ([propName, value]) => [
            propName,
            await renderJSXToClientJSX(value),
          ])
        )
      );
    }
  } else {
    throw new Error('Not implemented');
  }
}
```

### 5. Client Components

- Add support for client components

#### What we’ll build

<https://github.com/callstack-internal/rsc-from-scratch-workshop/assets/1174278/9b0d97fe-e314-42e3-a441-119408245702>

#### How it will work

1. Add client component in `src/app/favorite.jsx`

   ```js
   'use client';

   import React from 'react';

   export default function Favorite({ name }) {
     const [isFavorite, setIsFavorite] = React.useState(null);

     React.useEffect(() => {
       const isFavorite = localStorage.getItem(`${name}:favorite`) === 'true';

       setIsFavorite(isFavorite);
     }, [name]);

     const onClick = () => {
       setIsFavorite(!isFavorite);
       localStorage.setItem(`${name}:favorite`, String(!isFavorite));
     };

     return (
       <button disabled={isFavorite == null} onClick={onClick}>
         {isFavorite == null
           ? '…'
           : isFavorite
           ? 'Remove Favorite'
           : 'Add Favorite'}
       </button>
     );
   }
   ```

2. Import `Favorite` component and use it in `Pokemon`

   ```js
   import React from 'react';
   import Favorite from './favorite.jsx';

   export default async function Pokemon({ query }) {
     const data = await fetch(
       `https://pokeapi.co/api/v2/pokemon/${query.name}`
     ).then((res) => res.json());
     return (
       <div>
         <a href="/">Home</a>
         <h1>
           {query.name} ({data.types.map((type) => type.type.name).join(', ')})
         </h1>
         <div>
           <Favorite name={query.name} />
         </div>
         <img src={data.sprites.front_default} />
       </div>
     );
   }
   ```

3. Install `esbuild` as a dependency. It will be used to transpile jsx.

   ```
   yarn add esbuild
   ```

4. Create new file in `src/loader.js`

   ```js
   import { relative } from 'path';
   import { fileURLToPath } from 'url';
   import { renderToString } from 'react-dom/server';

   export async function load(url, context, defaultLoad) {
     const result = await defaultLoad(url, context, defaultLoad);

     if (result.format === 'module' && !url.includes('?original')) {
       const code = result.source.toString();

       if (/['"]use client['"]/.test(code)) {
         const source = `
           export default {
             $$typeof: Symbol.for('react.client.reference'),
             name: 'default',
             filename: ${JSON.stringify(
               relative(import.meta.dirname, fileURLToPath(url))
             )},
           }
         `;

         return { source, format: 'module' };
       }
     }

     return result;
   }
   ```

5. Update `src/client.js` to load client modules when parsing JSX.

   ```js
   function parseJSX(key, value) {
     if (value === '$RE') {
       return Symbol.for('react.element');
     } else if (typeof value === 'string' && value.startsWith('$$')) {
       return value.slice(1);
     } else if (value?.$$typeof === '$RE_M') {
       return window.__CLIENT_MODULES__[value.filename];
     } else {
       return value;
     }
   }
   ```

6. Update `src/server.jsx` to transform client components using `esbuild`

   - Add all necessary imports

     ```js
     import Router from '@koa/router';
     import { transform } from 'esbuild';
     import Koa from 'koa';
     import { AsyncLocalStorage } from 'node:async_hooks';
     import { readFile, readdir, stat } from 'node:fs/promises';
     import { register } from 'node:module';
     import { join } from 'node:path';
     import { renderToString } from 'react-dom/server';
     ```

   - Use `register` method from `node:module` to hook into module resolution and run `loader` before before running application code. Add it after imports

     ```js
     register('./loader.js', import.meta.url);
     ```

   - Change `router` from handling only `/client.js` route to return any `js/jsx` file to the client

     ```js
     router.get('/(.*).(js|jsx)', async (ctx) => {
       const content = await readFile(
         join(import.meta.dirname, ctx.path),
         'utf8'
       );
       const transformed = await transform(content, {
         loader: 'jsx',
         format: 'esm',
         target: 'es2020',
       });

       ctx.type = 'text/javascript';
       ctx.body = transformed.code;
     });
     ```

## Exercises

- Try more types of async operations like reading from the file system
- Add support for `React.Fragment` and `React.Suspense`
- Request only the client components used in a page instead of including all of them
- Cache JSX to reuse them on browser back/forward
- Try third-party libraries and see what's missing

## Additional resources

- [RFC: React Server Components](https://github.com/reactjs/rfcs/blob/main/text/0188-server-components.md) by [@en_js](https://twitter.com/en_js)
- [React Server Components (Video) - Introduction](https://reactjs.org/server-components)
- [Official React Server Components Demo](https://github.com/reactjs/server-components-demo)
- [RSC from scratch - Reference for this repo](https://github.com/reactwg/server-components/discussions/5) by [@dan_abramov2](https://twitter.com/dan_abramov2)
- [RSC payload (Diagram)](https://www.tldraw.com/v/ewUjqL4R984F5b-qghZUC?v=-5446,459,5280,3031&p=page) by [@lubieowoce](https://twitter.com/lubieowoce)
- [RSC from scratch (Video) - Using `react-server-dom-webpack`](https://www.youtube.com/watch?v=MaebEqhZR84) by [@BHolmesDev](https://twitter.com/BHolmesDev)
- [How React server components work: an in-depth guide](https://www.plasmic.app/blog/how-react-server-components-work) by [@chungwu](https://twitter.com/chungwu)
- [React Server Components, without a framework?](https://timtech.blog/posts/react-server-components-rsc-no-framework/) by [@tpillard](https://twitter.com/tpillard)
- [The Two Reacts](https://overreacted.io/the-two-reacts/) by [@gaearon](https://twitter.com/dan_abramov2)
- [Waku - A framework with server components](https://github.com/dai-shi/waku) by [@dai_shi](https://twitter.com/dai_shi)
