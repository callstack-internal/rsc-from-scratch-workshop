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
