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

### 2. Client-side navigation

- SPAs don’t reload the page when navigating
- Client-side navigation will override browser navigation

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

<video
  controls
  playsinline
  loop
  src="client-side-navigation.mp4"
  style="max-width: 100%;"
/>

### 3. Rehydration

### 4. Async Components

### 5. Client Components

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
