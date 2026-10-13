# AsyncStorage

Async Storage is a data storage system in React Native that is "not-encrypted", "asyncronous" and allow users to save data offline.

Because async storage is unencrypted, adding or retrieving data is easier and more efficient. Also, because it is asynchronous, methods run concorrently with the rest of the code. Being presistent means that data will always be available globally, even when logging out, restarting the application or the device.

It thus allows accessing data from any react native component in a performance efficient way. Storing and retrieving data from the AsyncStorage does not affect the other codes from running.

Async Storage uses key-value pairs. 

How it works:
async storage accepts and stores **ONLY STRING** data, so we must always serialize the data before storing it if it isn't a string. Both key and value must be strings.

* for converting the object to a string, use JSON.stringify().
* when we get data back from the storage, use JSON.parse().

Cases where we might need async storage such as theme data, user preferences in general. However, we can't use it with sensitive information since it is not encrypted, and anyone with access to the phone's file system can access the data.
