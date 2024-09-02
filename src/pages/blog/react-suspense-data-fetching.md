---
layout: "../../layouts/Post.astro"
title: Suspense for data fetching
image: /images/posts/react-suspense-data-fetching/react-suspense-thumbnail
publishedAt: "2024-09-02"
category: "React"
---

## Suspense For Data Fetching

Since React 18, Suspense has improved not only by supporting asynchronous component loading but also by allowing asynchronous data loading and handling within a component. This way, we can avoid using state and useEffect to control asynchronous data within a component.

In this post, we will explore how we can improve the readability of our code by using Suspense for asynchronous data control within the component, along with ErrorBoundaries to handle errors in these asynchronous calls.

Let’s start by understanding the ErrorBoundary component (in this case, it’s a component I defined, but you can use libraries that already implement it), which will encapsulate our components that depend on asynchronous data, waiting to catch any rendering errors in the component. If an error is detected, it will handle it by displaying a fallback component.

```js
// ErrorBoundary.tsx
import React, { ErrorInfo, ReactNode } from "react";

interface ErrorBoundaryProps {
  fallback: ReactNode;
  children: ReactNode;
}

class ErrorBoundary extends React.Component<ErrorBoundaryProps> {
  state = { hasError: false };

  // eslint-disable-next-line @typescript-eslint/no-unused-vars
  static getDerivedStateFromError(error: Error) {
    return { hasError: true };
  }

  componentDidCatch(error: Error, info: ErrorInfo) {
    console.log(error, info);
  }

  render() {
    if (this.state.hasError) {
      return this.props.fallback;
    }
    return this.props.children;
  }
}

export default ErrorBoundary;
```

First, I will show how our App.tsx and Pokemon.tsx components would look without using Suspense.

```js
// App.tsx
import "./App.css";
import ErrorBoundary from "./ErrorBoundary";
import Pokemon from "./Pokemon";

function App() {
  return (
    <>
      <ErrorBoundary fallback="There was an error. Ups... :p">
        <Pokemon name="charmander" />
      </ErrorBoundary>
      <ErrorBoundary fallback="There was an error. Ups... :p">
        <Pokemon name="asda" />
      </ErrorBoundary>
    </>
  );
}

export default App;
```

```js
// Pokemon.tsx
import { useEffect, useState } from 'react';

interface PokemonProps {
    name: string;
}

function Pokemon({ name }: PokemonProps) {
  const [data, setData] = useState<Pokemon | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);

  interface Pokemon {
    name: string;
    height: string;
  }

  useEffect(() => {
    fetch(`https://pokeapi.co/api/v2/pokemon/${name}`)
      .then((response) => {
        if (!response.ok) {
          throw new Error('Network response was not ok');
        }
        return response.json();
      })
      .then((data: Pokemon) => {
        setData(data);
        setLoading(false);
      })
      .catch((error: Error) => {
        setError(error);
        setLoading(false);
      });
  }, []);

  if (loading) {
    return <div>Loading...</div>;
  }

  if (error) {
    // Throw here to force a render error so ErrorBoundary can catch it.
    throw error; // Throw error to ErrorBoundary
  }

  return (
    <div>
      <h1>name: {data!.name}</h1>
      <p>height: {data!.height} decimeters</p>
    </div>
  );
}

export default Pokemon;
```

As you can see, in the Pokemon.tsx component, we use several states to manage errors, loading status, and data, along with a useEffect. We can simplify all of this by using the "swr" package, which allows us to use Suspense to control the asynchronous request that the component’s data depends on.

With this package, the App.tsx component would look as follows:

```js
// App.tsx with Suspense
import { Suspense } from "react";
import "./App.css";
import ErrorBoundary from "./ErrorBoundary";
import Pokemon from "./Pokemon";

function App() {
  return (
    <>
      <ErrorBoundary fallback="There was an error. Ups... :p">
        <Suspense fallback="Is loading...">
          <Pokemon name="charmander" />
        </Suspense>
      </ErrorBoundary>
      <ErrorBoundary fallback="There was an error. Ups... :p">
        <Suspense fallback="Is loading...">
          <Pokemon name="sada" />
        </Suspense>
      </ErrorBoundary>
    </>
  );
}

export default App;
```

Here you can see, compared to what we had before, that the Pokemon component is now wrapped with the Suspense component, where we specify a fallback component (in this case, simple text saying “Is loading”) that will be displayed while the asynchronous data request for the Pokemon component is being made.

The Pokemon.tsx component would look as follows:

```js
// Pokemon.tsx with Suspense
import useSWR from 'swr';

interface PokemonProps {
    name: string;
}

const fetcher = (url: string) => fetch(url).then((res) => {
  if (!res.ok) {
      throw new Error('Network response was not ok');
  }
  return res.json();
});

function Pokemon({ name }: PokemonProps) {

  interface Pokemon {
    name: string;
    height: string;
  }

  const { data } = useSWR<Pokemon>(`https://pokeapi.co/api/v2/pokemon/${name}`, fetcher, {
    suspense: true
  });

  return (
    <div>
      <h1>name: {data!.name}</h1>
      <p>height: {data!.height} decimeters</p>
    </div>
  );
}

export default Pokemon;
```

Here we can see how, by using useSWR from the "swr" package, we can make the API request by specifying a fetch method as the second parameter, which will handle making that request to the specified URL. If the response is not satisfactory, it will return an error, which will be handled by the ErrorBoundary. Additionally, as a third parameter, we need to pass an object with the suspense property set to true, so that the Suspense component wrapping this component (as we saw in App.tsx) can be used.

In this simple way, we can make our components that depend on asynchronous data simpler and more readable.
