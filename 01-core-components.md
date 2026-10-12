## <span style="color:gold;font-weight:bold"> React Native Core Components </span>

### <span style="color:silver;font-weight:bold">1. View</span>

View is the most generic react native componen it maps to UIView, div android.View in other platforms.
it is designed to be nested, so a view can have 0 to many children of any type.

> Also btw, Views are meant to be used with stylesheets - for clarity, performance, and good practice. eventhough inline styling is also supported

View component can be imported from <span style="color:lightgreen">'react-native'</span> library.

<hr />

View components has a plethora of props, but imo they can be categorized into four categories:
##### 1. Layout and styling: especially the <span style="color:chocolate">style</span> prop, which is essential for applying Flexbox layouts. Also the <span style="color:chocolate">hitSlop</span> prop defines the touchable area extention. Also, <span style="color:chocolate">removeClippedSubviews</span> optimizes performance for scrolling content with manny off-screen subviews.

##### 2. Accessibility: for examble all props with accessibility- prefex. Notable ones are <span style="color:chocolate">accessible</span>, <span style="color:chocolate">accessibilityLabel</span>, and <span style="color:chocolate">accessibilityLabel</span>, which are critical to screen renders.
##### 3. Touch and Interaction: E.g. <span style="color:chocolate">onStartShouldSetResponder</span>, which contorls whether the view can be the target of touch events.
##### 4. Identification and Testing: E.g. <span style="color:chocolate">nativeID</span>, which locates the view from native classes, while <span style="color:chocolate">testID</span> is specifically designed for e2e testing frameworks to identify the component in the DOM tree.

<hr />

### <span style="color:silver;font-weight:bold">2. Text</span>

Text is the component for displaying text. it maps to **UITextView** on iOS, **TextView** on android, and basically a **span/p** on web.
the thing is, in react native you **CANNOT** just throw a string inside a View like in html, ***every piece of text must be wrapped inside a Text component***, otherwise the app crashes.

> Also btw, Text supports nesting, so a Text inside a Text inherits the styles of the parent one. this is the only place in RN where style inheritance actually happens kinda like css.

Text component can be imported from <span style="color:lightgreen">'react-native'</span> library.

<hr />

Text props imo can be categorized into three categories:
##### 1. Layout and styling: the <span style="color:chocolate">style</span> prop of course (fontSize, fontWeight, color, etc). Also <span style="color:chocolate">numberOfLines</span>, which truncates the text after a certain number of lines, and <span style="color:chocolate">ellipsizeMode</span> which decides where the "..." goes (head, middle, tail, clip).
##### 2. Touch and Interaction: E.g. <span style="color:chocolate">onPress</span> and <span style="color:chocolate">onLongPress</span>, yes Text is pressable on its own. Also <span style="color:chocolate">selectable</span>, which lets the user select and copy the text.
##### 3. Accessibility and Testing: same story as View, <span style="color:chocolate">accessibilityLabel</span>, <span style="color:chocolate">accessibilityRole</span>, and <span style="color:chocolate">testID</span>.

<hr />

### <span style="color:silver;font-weight:bold">3. Image</span>

Image is the component for displaying images, it maps to UIImageView on iOS and ImageView on android, and img on web.
it can display images from different sources: local static images (using require), network images (using a uri), and even base64 data.

> Also btw, network images MUST have width and height specified in the style, otherwise they wont show at all. local images using require get their dimensions automatically tho.

Image component can be imported from <span style="color:lightgreen">'react-native'</span> library.

<hr />

Image props imo can be categorized into three categories:
##### 1. Source: the <span style="color:chocolate">source</span> prop, which is either require('./img.png') or an object like {uri: 'https://...'}. Also <span style="color:chocolate">defaultSource</span> which shows a placeholder while the actual image is loading.
##### 2. Layout and styling: <span style="color:chocolate">style</span> obviously, and <span style="color:chocolate">resizeMode</span>, which decides how the image fits its frame (cover, contain, stretch, repeat, center). Also <span style="color:chocolate">blurRadius</span> for blurring the image.
##### 3. Loading lifecycle: E.g. <span style="color:chocolate">onLoadStart</span>, <span style="color:chocolate">onLoad</span>, <span style="color:chocolate">onLoadEnd</span>, and <span style="color:chocolate">onError</span>, which are useful for showing loaders or fallback images when something goes wrong.

> Note: if you want to put content on top of an image, use <span style="color:lightgreen">ImageBackground</span> instead, since Image can't have children.

<hr />

### <span style="color:silver;font-weight:bold">4. ScrollView</span>

ScrollView is a scrollable container, it maps to UIScrollView on iOS and ScrollView on android.
in RN views are NOT scrollable by default (unlike web), so if your content overflows the screen you need to wrap it in a ScrollView.

> Also btw, ScrollView renders ALL of its children at once, even the ones off-screen. so it's fine for small amounts of content, but for long lists use FlatList instead, otherwise performance goes to hell.

ScrollView component can be imported from <span style="color:lightgreen">'react-native'</span> library.

<hr />

ScrollView props imo can be categorized into three categories:
##### 1. Layout and styling: <span style="color:chocolate">style</span> for the outer container and <span style="color:chocolate">contentContainerStyle</span> for the inner content (this is where you put padding, alignItems, etc). Also <span style="color:chocolate">horizontal</span> which makes it scroll sideways.
##### 2. Scrolling behavior: E.g. <span style="color:chocolate">pagingEnabled</span>, which snaps the scroll to pages (useful for carousels), <span style="color:chocolate">showsVerticalScrollIndicator</span> and <span style="color:chocolate">showsHorizontalScrollIndicator</span> to hide the scroll bars, and <span style="color:chocolate">keyboardShouldPersistTaps</span> which decides what happens to the keyboard when you tap inside the scroll view.
##### 3. Events and refreshing: <span style="color:chocolate">onScroll</span> which fires while scrolling (use it with <span style="color:chocolate">scrollEventThrottle</span> to control how often it fires), and <span style="color:chocolate">refreshControl</span> for pull-to-refresh.

<hr />

### <span style="color:silver;font-weight:bold">5. TextInput</span>

TextInput is the component for taking text input from the user via the keyboard, it maps to UITextField on iOS, EditText on android, and input on web.
it is usually used as a controlled component, meaning you store its value in a state and update it on every change.

> Also btw, TextInput has almost no default styling on iOS, so if you don't give it a border or something you won't even see it lol.

TextInput component can be imported from <span style="color:lightgreen">'react-native'</span> library.

<hr />

TextInput props imo can be categorized into three categories:
##### 1. Value and changes: <span style="color:chocolate">value</span> and <span style="color:chocolate">onChangeText</span> which are the core of controlled inputs. Also <span style="color:chocolate">placeholder</span>, <span style="color:chocolate">defaultValue</span>, and <span style="color:chocolate">maxLength</span>.
##### 2. Keyboard configuration: E.g. <span style="color:chocolate">keyboardType</span> (default, numeric, email-address, phone-pad, etc), <span style="color:chocolate">secureTextEntry</span> for passwords, <span style="color:chocolate">autoCapitalize</span>, <span style="color:chocolate">autoCorrect</span>, and <span style="color:chocolate">returnKeyType</span> which changes the label of the enter key.
##### 3. Events and focus: <span style="color:chocolate">onFocus</span>, <span style="color:chocolate">onBlur</span>, and <span style="color:chocolate">onSubmitEditing</span> which fires when the user hits the enter key. Also <span style="color:chocolate">multiline</span> which turns it into a textarea basically.

<hr />

### <span style="color:silver;font-weight:bold">6. Pressable</span>

Pressable is a wrapper component that detects press interactions on its children. it is the modern and recommended way to handle touches in RN, replacing the older TouchableOpacity, TouchableHighlight, etc.
the cool thing about it is that it gives you the <span style="color:chocolate">pressed</span> state, so you can change the style or content based on whether the user is pressing or not.

> Also btw, both style and children can be functions that receive {pressed}, e.g. style={({pressed}) => [styles.btn, pressed && styles.btnPressed]}.

Pressable component can be imported from <span style="color:lightgreen">'react-native'</span> library.

<hr />

Pressable props imo can be categorized into three categories:
##### 1. Press events: <span style="color:chocolate">onPress</span>, <span style="color:chocolate">onPressIn</span>, <span style="color:chocolate">onPressOut</span>, and <span style="color:chocolate">onLongPress</span>. Also <span style="color:chocolate">delayLongPress</span> to control how long a long press is.
##### 2. Touch area: <span style="color:chocolate">hitSlop</span> which extends the touchable area without changing the layout (same as View), and <span style="color:chocolate">pressRetentionOffset</span> which defines how far the finger can move away before the press is cancelled.
##### 3. Styling and feedback: <span style="color:chocolate">style</span> (as a function or object), <span style="color:chocolate">android_ripple</span> for the ripple effect on android, and <span style="color:chocolate">disabled</span> which disables the press behavior.

<hr />

### <span style="color:silver;font-weight:bold">7. Modal</span>

Modal is a component for presenting content on top of everything else, like a popup or a dialog. it renders natively above the whole app, so you don't have to worry about z-index stuff.
it is controlled by the <span style="color:chocolate">visible</span> prop, which you usually tie to a state.

> Also btw, on android the hardware back button doesn't close the modal by itself, you have to handle it with <span style="color:chocolate">onRequestClose</span>, otherwise the user gets stuck.

Modal component can be imported from <span style="color:lightgreen">'react-native'</span> library.

<hr />

Modal props imo can be categorized into three categories:
##### 1. Visibility: <span style="color:chocolate">visible</span> which shows/hides the modal, and <span style="color:chocolate">onRequestClose</span> which fires when the user presses back on android (required on android btw).
##### 2. Appearance and animation: <span style="color:chocolate">animationType</span> (none, slide, fade), <span style="color:chocolate">transparent</span> which makes the background see-through so you can make a dimmed overlay, and <span style="color:chocolate">presentationStyle</span> on iOS (fullScreen, pageSheet, formSheet).
##### 3. Lifecycle events: <span style="color:chocolate">onShow</span> which fires once the modal is shown, and <span style="color:chocolate">onDismiss</span> (iOS) which fires after it's dismissed.
