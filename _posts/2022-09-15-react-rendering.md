---
layout: post
title:  "Improving React Rendering Performance with useCallback, useMemo and React.memo"
---

A component is re-evaluated when
  - state changes
  - props changes

Rendering is done in two phases
  1. **Render phase**: A component function is evaluated, a react element is created, and virtual DOM is created. If virtual DOM was already created from the previous rendering, it will adjust the virtual DOM with the new changes from the component if there is any and mark those changes.
  2. **Commit phase**: Apply the changes marked in render phase to real DOM. If no change was marked, then this phase is skipped. 


**Example**

Suppose we have the following simple component hierarchy where the child component does some heavy-performance task

```
// in Child.js
...
export default function Child({ f }) {
  return (
    <div>
      {Array.from({length:10000},(_,i)=> (
        <p>{i}</>p
      ))}
    </div>
  );
}

// in Parent.js
...
export default function Parent() {
  const [title, setTitle] = useState(null);
  const f = () => {};
  useEffect(() => setTitle('hello'), []);
  
  return (
    <div>
      <p>{title}</p>
      <Child f={f} />
    </div>
  );
}
```

In this case, when `Parent` is re-evaulated from `setTitle`, `Child` component function will be called again. This is because under the hood, react is actually calling `React.createElement('Child', {f: f})` when JSX in `Parent` component is complied into the javascript code and exceuted. A react element for `Child` component will be created. Since props value (i.e., `f`) has changed from the last evaluation (on every evaluation `Parent` creates a new function and set it to `f`), so react will mark this change and commit this to real DOM in the commit phase.

**How can we improve this?**

We can wrap the function from the `Parent` component with `useCallback`. This will memoize the function, and react will point to the same function on each evalution. `Child` component will go through render phase, but will not find any changes from the last evalution becuase `props` did not change. Since no change is marked, commit phase will be skipped.

**Can we do better?**

We still need to create a `Child` element and does a heavy workload even though the content of the Child element doesn't change. This is where `React.memo` comes in. `React.memo` allows us to skip re-rending of a componet if `props` passed to the component did not change from the last evaluation. 

```
...
export default React.memo(function Child({...}));
```

Then when `Parent` componet is re-rendered because of `setTitle`, react will simply use the `Child` element created from the first evaluation of `Parent`.

**What if we pass a reference type other than function to Child component?**
- Since `React.memo` does shallow comparison on props, we cannot just pass a reference type such as array or object to `Child`. To avoid re-rendering, we can pass the reference type variable to `useMemo`. This will memoize the variable and use it on each evaulation just like we passed a function to `useCallback`.

`useEffect({ title: 'Welcome', subtitle: 'blah' });`
