## <span style="color:gold;font-weight:bold"> Auth Navigation with Mock State </span>

most apps have two "worlds": the **logged-out** screens (Login, Register) and the **logged-in** screens (Home, Profile...). the user should never be able to go back to Login after logging in, or to Home after logging out.
the react-navigation way to do this is NOT navigating manually, it's **rendering different screens based on the auth state**.

<hr />

### <span style="color:silver;font-weight:bold">1. Conditional screens</span>

```jsx
function RootNavigator() {
  const { isLoggedIn } = useAuth();

  return (
    <Stack.Navigator>
      {isLoggedIn ? (
        <Stack.Group>
          <Stack.Screen name="Home" component={HomeScreen} />
          <Stack.Screen name="Profile" component={ProfileScreen} />
        </Stack.Group>
      ) : (
        <Stack.Group screenOptions={{ headerShown: false }}>
          <Stack.Screen name="Login" component={LoginScreen} />
          <Stack.Screen name="Register" component={RegisterScreen} />
        </Stack.Group>
      )}
    </Stack.Navigator>
  );
}
```

when isLoggedIn flips, react-navigation sees the screens changed and **automatically** jumps to the first screen of the new group. the old screens are removed, so there is no "back" to them.

> Also btw, don't call navigation.navigate('Home') after logging in. just update the state and let the navigator do its thing.

<hr />

### <span style="color:silver;font-weight:bold">2. Mock auth state</span>

since there is no real backend, the auth state is just a **useState inside a context**, with fake signIn/signOut functions.

```jsx
const AuthContext = createContext(null);

export function AuthProvider({ children }) {
  const [isLoggedIn, setIsLoggedIn] = useState(false);

  const signIn = () => setIsLoggedIn(true);
  const signOut = () => setIsLoggedIn(false);

  return (
    <AuthContext.Provider value={{ isLoggedIn, signIn, signOut }}>
      {children}
    </AuthContext.Provider>
  );
}

export const useAuth = () => useContext(AuthContext);
```

wrap the app in AuthProvider (outside the NavigationContainer), then the Login screen calls signIn() and the Profile screen calls signOut().
to make the mock more realistic, signIn can wait with a setTimeout to fake a network request, and you can add an *isLoading* state to show a splash screen while checking a saved token.
