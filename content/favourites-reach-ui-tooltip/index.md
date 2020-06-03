---
title: "Favourites: Reach UI Tooltip"
description: Why I like — and what I learned from — the Reach UI Tooltip
date: "2020-05-21T13:04:00+1000"
categories: []
keywords: []
ckTags: ["1464981"]
cardImage: "./title-card.jpeg"
---

![Neon sign](title.jpeg "Photo by Yana Nikulina on Unsplash")

A while ago, I decided I ought to start writing blog articles about some of my favourite components, packages and tools. I'd like to share not just what these things do and what prompted me to choose them, but also what I've learned from them, as — with components, in particular — there's plenty that can be learned by delving into the source code.

This is the first of those articles. It's a look at the Reach UI Tooltip: why I like it and what I learned from it.

## Problems with tooltips

Whenever I've had to deal with them, tooltip and popup components have been a source of frustration. Too many of the components that I've used have suffered from at least one of the following problems:

- styling that's at odds with the application's styling;
- unwanted host elements in the DOM; and/or
- buggy state management that sometimes results in multiple tooltips being visible.

So when [Brian Vaughn](https://twitter.com/brian_d_vaughn) added the Reach UI Tooltip [to the React DevTools](https://github.com/bvaughn/react-devtools-experimental/commit/8b33dd673d965d23d2516bd30c9c3d05386ce881), I took a peek at it. I liked what I saw, I learned a few things from it and I started using it in the [RxJS DevTools](https://rxjs.tools) that I've been working on.

Let's take look at how the tooltip avoids or solves the above problems and maybe learn a few things along the way.

## Styling

Out of the box, the Reach UI components have minimal styling — just enough for the components to make visual sense. The default tooltip looks like this:

![Default tooltip](default-widened.png)

The tooltip's default styles are declared in a [`styles.css`](https://github.com/reach/reach-ui/blob/52d721cd643d1a6d24c7253ab24e114be262d092/packages/tooltip/styles.css) file within the package and there are number of ways you can override them — although, you might choose to ignore them completely. If you do ignore them, targeting the tooltip popup's element with a CSS selector is easy, as the element has a `data-reach-tooltip` attribute applied to it.

The most straightforward way to override the default styles is by specifying a `styles` prop for the `Tooltip` element:

<!-- prettier-ignore -->
```tsx
<Tooltip
  label="Custom tooltip style"
  style={{
    backgroundColor: "#333",
    border: "none",
    borderRadius: "3px",
    color: "#fff",
  }}
>
  <button>❤️</button>
</Tooltip>
```

The `style` prop is passed through to the tooltip popup and the resulting tooltip looks like this:

![Custom tooltip](custom-widened.png)

That's useful, but the really nice bit is that the tooltip has multiple levels of abstraction. In addition to the `Tooltip` component — that we've seen above — there are two lower-level abstractions:

- a `useTooltip` hook; and
- a `TooltipPopup` component.

You can use these lower-level abstractions to compose tooltips that include additional elements — like [a triangle](https://reacttraining.com/reach-ui/tooltip/#triangle-pointers-and-custom-styles):

<!-- prettier-ignore -->
```tsx
function Tooltip({ children, label, "aria-label": ariaLabel }) {
  const [trigger, tooltip] = useTooltip();
  const { isVisible, triggerRect } = tooltip;
  return (
    <Fragment>
      {cloneElement(children, trigger)}
      {isVisible && (
        <Portal>
          <div
            style={{
              borderBottom: "10px solid #333",
              borderLeft: "10px solid transparent",
              borderRight: "10px solid transparent",
              height: 0,
              left: triggerRect
                ? triggerRect.left - 10 + triggerRect.width / 2
                : undefined,
              position: "absolute",
              top: triggerRect
                ? triggerRect.bottom + window.scrollY
                : undefined,
              width: 0,
            }}
          />
        </Portal>
      )}
      <TooltipPopup
        {...tooltip}
        aria-label={ariaLabel}
        label={label}
        position={centered}
        style={{
          backgroundColor: "#333",
          border: "none",
          borderRadius: "3px",
          color: "#fff",
          padding: "0.5em 1em",
        }}
      />
    </Fragment>
  );
}
```

Which looks like this:

![Tooltip with triangle](triangle-widened.png)

At first glance, it seems like there's a lot of code involved with this, but almost all of it is styling. The `useTooltip` hook and the `TooltipPopup` component abstract the tedious and tricky stuff. Nice!

## Host elements

It's annoying if a tooltip component wraps its children with additional host elements, as that messes with inline elements. For example, it ought to be possible to add a tooltip to a `span`, like this:

<!-- prettier-ignore -->
```tsx
<span>this next phrase</span>
<Tooltip label="This is the tooltip">
  <span>has a tooltip</span>
</Tooltip>
```

But if the `Tooltip` component puts its children into a `div` host element, it will break the layout.

The Reach UI Tooltip does not do this. It doesn't wrap its children with anything. When the Reach UI Tooltip is used in the above JSX, the DOM will look like this:

```html
<span>this next phrase</span>
<span data-reach-tooltip-trigger>has a tooltip</span>
```

It manages this using React's [`cloneElement`](https://reactjs.org/docs/react-api.html#cloneelement) function. You might have noticed that it's called in the triangle tooltip that we looked at earlier:

<!-- prettier-ignore -->
```tsx
<Fragment>
  {cloneElement(children, trigger)}
```

Here, `cloneElement` creates a new element — that's a clone of `children` — with props that are a shallow merge of the props in the original element with the props specified in `trigger`.

So the tooltip doesn't mess with the DOM elements around the child that it wraps. Instead, it merges [additional props](https://github.com/reach/reach-ui/blob/52d721cd643d1a6d24c7253ab24e114be262d092/packages/tooltip/src/index.tsx#L333-L347) — in particular, the mouse, keyboard and focus callbacks — into a clone of the child.

That means you can use a tooltip anywhere. You can put one around a `span` or an inline `button` and you can even put one inside an SVG.

## State management

Out of the three problems that I listed earlier, buggy state management is the most common. The reason for this is that the logic involved in managing a tooltip's state transitions is complicated: whether or not the tooltip is shown or dismissed depends upon the focus, the keyboard and the mouse.

The Reach UI Tooltip's state management is solid. It uses state machine: a [state chart](https://github.com/reach/reach-ui/blob/52d721cd643d1a6d24c7253ab24e114be262d092/packages/tooltip/src/index.tsx#L105-L160) and a [`transition`](https://github.com/reach/reach-ui/blob/52d721cd643d1a6d24c7253ab24e114be262d092/packages/tooltip/src/index.tsx#L609-L636) function that's called from the keyboard and mouse event handlers. A single state machine is used to manage the tooltip, so there's no chance of multiple tooltips being shown simultaneously. It makes the complexity manageable and understandable. It just works.

---

Reach UI has a whole bunch of other components and, like the tooltip, they're all written to be accessible and composable and have unopinionated styling. If the tooltip appeals to you, you should definitely [check them out](https://reacttraining.com/reach-ui).

## Additional resources

[Michael Chan](https://twitter.com/chantastic) interviews [Chance Strickland](https://twitter.com/chancethedev) — one of Reach UI's contributors — in this podcast: [Reach UI and Building Composable Open Source](https://reactpodcast.com/92).

[David Khourshid](https://twitter.com/DavidKPiano) looks at the advantages of styling with data attributes — they're used extensively in the Reach UI compoments — in his talk at CSSConf BP 2019: [Crafting Stateful Styles with State Machines](https://www.youtube.com/watch?v=0cqeGeC98MA).
