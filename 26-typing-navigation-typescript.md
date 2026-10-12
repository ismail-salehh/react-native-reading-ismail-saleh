## <span style="color:gold;font-weight:bold"> Typing Navigation with TypeScript </span>

without types, nothing stops you from navigating to a screen that doesn't exist or forgetting a param, and you only find out when the app crashes.
with TypeScript, you describe **every screen and its params** once, and then navigate, route.params, etc are all checked.

<hr />

### <span style="color:silver;font-weight:bold">1. The param list</span>

a type where each key is a screen name and the value is its params. use *undefined* for screens with no params.

```tsx
type RootStackParamList = {
  Home: undefined;
  Details: { id: number };
  Profile: { userId: string } | undefined; // optional params
};

const Stack = createNativeStackNavigator<RootStackParamList>();
```

<hr />

### <span style="color:silver;font-weight:bold">2. Typing screens</span>

use <span style="color:chocolate">NativeStackScreenProps</span> to type the props of a screen, it gives you both navigation and route typed.

```tsx
import type { NativeStackScreenProps } from '@react-navigation/native-stack';

type Props = NativeStackScreenProps<RootStackParamList, 'Details'>;

function DetailsScreen({ route, navigation }: Props) {
  route.params.id; // number ✅
}
```

for tabs and drawer it's the same idea with BottomTabScreenProps and DrawerScreenProps.

<hr />

### <span style="color:silver;font-weight:bold">3. Typing the hooks</span>

the hooks take the type as a generic:

```tsx
const navigation = useNavigation<NativeStackNavigationProp<RootStackParamList>>();
const route = useRoute<RouteProp<RootStackParamList, 'Details'>>();
```

> Also btw, to avoid passing the generic to useNavigation every time, you can declare the root param list **globally** once:
> `declare global { namespace ReactNavigation { interface RootParamList extends RootStackParamList {} } }`
