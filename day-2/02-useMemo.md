# UseMemo

useMemo is used for memoizing a calculation between renders.

Perché? Perché imagine you have for exampple a list of a thousand plants, and you want to display only the plants that are available. Ogni volta lat tua component re-renders, react normalmente runs tua filtering calculation un'altra volta, nonostante i pianti non hanno cambiare mai.

useMemo lets you reuse the previous result until the relevant data changes.

How do you use it?????

for exapmle 
```js
const filteredPlants = useMemo(() => {
    return plants.filter(plant => 
        plant.name.toLowerCase().includes(search.toLowerCase())
    );
}, [plants, search]);
```

ci sono tré cose importante che devi a sapere:
1. calculations: filters i pianti based on the search text.
2. dependencies: [plants, search] tells react which values the calculation depends on.
3. cached results: react can reuse the previous results when neither dependency has changed.

when should we use it in React Native?

1. Large Lists
2. expensive calculations
3. stable objects or array references. (when passing calculated values to a memoized child component and you want to avoid unnecessary child re-renders.)

