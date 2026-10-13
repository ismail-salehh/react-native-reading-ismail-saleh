# Global vs local state

## Local state

State that belongs to **one component** (or a small subtree). Lives in `useState` / `useReducer` inside that component.

Examples: text in a `TextInput`, whether a dropdown is open, the current step of a form, a loading flag for one button.

```jsx
function LikeButton() {
  const [liked, setLiked] = useState(false);
  return <Button title={liked ? 'Liked' : 'Like'} onPress={() => setLiked(!liked)} />;
}
```

## Global state

State that **many unrelated components** need to read or change, often across screens.

Examples: the logged-in user, theme, language, shopping cart, unread notification count, feature flags.

Tools: Context API, Redux Toolkit, Zustand (see the next topics).

## Lifting state up (the middle ground)

When two siblings need the same value, move the state to their closest common parent and pass it down by props. This is still "local": no global tool is needed.

```jsx
function Parent() {
  const [query, setQuery] = useState('');
  return (
    <>
      <SearchBar value={query} onChange={setQuery} />
      <Results query={query} />
    </>
  );
}
```

## How to decide

Ask in order and stop at the first "yes":

1. Does only one component use it? Keep it local.
2. Do only a parent and a few children use it? Lift it up and pass props.
3. Is it passed through many layers that don't use it (prop drilling)? Consider Context.
4. Is it shared across screens, updated often, or needs selectors/devtools/persistence? Use a store (Redux Toolkit / Zustand).

## Why not make everything global?

- Every component that subscribes can re-render when it changes (see the re-rendering topic).
- Harder to reason about: any file can change it.
- Harder to test and reuse components, since they depend on the store.
- Screen-specific state should disappear when the screen unmounts. Global state sticks around and can show stale values when the user returns.

## React Native specifics

- Navigation params are a form of local state shared between two screens. Use them for small data (an id), not whole objects that can go stale.
- Form state (`TextInput` values) is almost always local. Updating a global store on every keystroke causes lots of re-renders.
- Screens in a stack stay mounted, so local state survives going forward and back. Global state is shared by every screen, including ones in the background.
- Data from an API is neither: it is **server state** (see topic 10).

## Rule of thumb

Start local. Lift up when needed. Go global only when sharing becomes painful.
