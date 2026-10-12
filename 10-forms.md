## <span style="color:gold;font-weight:bold"> Forms </span>

there is **no form element** in react native, no submit event, no name attributes, no FormData. a form is just a bunch of TextInputs + a button that does something with the state.

<hr />

### <span style="color:silver;font-weight:bold">1. Controlled inputs</span>

each TextInput gets a <span style="color:chocolate">value</span> and an <span style="color:chocolate">onChangeText</span>. for multiple fields, one state object is cleaner than a useState per field.

```jsx
const [form, setForm] = useState({ email: '', password: '' });

<TextInput
  value={form.email}
  onChangeText={email => setForm(prev => ({ ...prev, email }))}
  keyboardType="email-address"
  autoCapitalize="none"
/>
<TextInput
  value={form.password}
  onChangeText={password => setForm(prev => ({ ...prev, password }))}
  secureTextEntry
/>
<Button title="Login" onPress={handleSubmit} />
```

validation is also manual: check the values in handleSubmit, store the errors in a state, and show them under the inputs.

<hr />

### <span style="color:silver;font-weight:bold">2. The keyboard</span>

the keyboard is the annoying part of forms on mobile:
- **next field**: set *returnKeyType="next"*, and in <span style="color:chocolate">onSubmitEditing</span> call passwordRef.current.focus() to jump to the next input.
- **covering inputs**: wrap the form in <span style="color:chocolate">KeyboardAvoidingView</span> so the content moves up when the keyboard opens.
- **tapping buttons with the keyboard open**: inside a ScrollView, set *keyboardShouldPersistTaps="handled"*, otherwise the first tap only closes the keyboard.
- **closing it**: <span style="color:chocolate">Keyboard.dismiss()</span>.

> Also btw, for big forms, libraries like **react-hook-form** (with its Controller component for TextInput) and **zod** or **yup** for validation save you a lot of boilerplate.
