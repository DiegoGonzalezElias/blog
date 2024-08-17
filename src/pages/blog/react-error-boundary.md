---
layout: "../../layouts/Post.astro"
title: Handle errors with ErrorBoundaries
image: /images/posts/react-error-boundary/error-boundary-thumbnail
publishedAt: "2024-08-17"
category: "React"
---

## ErrorBoundaries

In case you wanna handle errors and avoid white screens when some of your components on react fails to render. You can use ErrorBoundary, which is like a component that wrap your piece of code/components that may brake becacose of whatever reason.

In this post we are gonna talk about this method called ErrorBoundary to handle render errors on react.

But first of all, it is important that the error must be a render error not a promise error or anything like that, so ErrorBoundary component can handle it.

Now, let’s take a look how the ErrorBoundary component should look like:

```js
//ErrorBoundary.tsx
import React, { ErrorInfo, ReactNode } from "react";

interface ErrorBoundaryProps {
  fallback: ReactNode;
  children: ReactNode;
}

class ErrorBoundary extends React.Component<ErrorBoundaryProps> {
  state = { hasError: false };

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

Inside this component we can see this methods:

- **getDerivedStateFromError:** which handles if there was an error or not.
- **componentDidCatch:** inside this methos we can define what are we going to do with the error. In this case is displayed on the console.
- **render:** this method returns the fallback component in case of error or the original component.

To test if this is working as expected. I have created this component call “Pokemon” that make a call to a Pokemon API and in case the pokemon you are searching for is not found, will throw the error so the ErrorBoundary component can display the fallback component:

```js
//Pokemon.tsx
import { useEffect, useState } from 'react';

interface PokemonProps {
    name: string;
}

function Pokemon({name}: PokemonProps) {
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
    //thow here to force a render error so ErrorBoundary can catch it.
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

```js
//App.tsx
import "./App.css";
import ErrorBoundary from "./ErrorBoundary";
import Pokemon from "./Pokemon";

function App() {
  return (
    <>
           {" "}
      <ErrorBoundary fallback="There was an error. Ups... :p">
                  <Pokemon name="charmander" />     {" "}
      </ErrorBoundary>
            <ErrorBoundary fallback="There was an error. Ups... :p">
                  <Pokemon name="asda" />     {" "}
      </ErrorBoundary>   {" "}
    </>
  );
}

export default App;
```

You can run this example and see how the first Pokemon component is rendered, but the second show the fallback component, that in this example is just a text saying: “There was an error. Ups... :p”.

![App](/images/posts/react-error-boundary/errorBoundary-app.webp)
