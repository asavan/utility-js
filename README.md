# utility-js
An assortment of helpful Javascript functions

## makeWorker
### Create a WebWorker from an existing function. Used to offload CPU-expensive functions from the main thread.
```javascript
// Turns a given function's content into a WebWorker object. Only function without arguments can be converted
const makeWorker = fn => new Worker(URL.createObjectURL(new Blob([fn.toString().replace(/\t|(^[a-z ]*\(\) ?{\r?\n?)|(^(\w*? )?(\w*? )?(= )?\(?\w*?,?\w*?\)? ?=> ?{\r?\n?)|(\r?\n?};?$)/g, "")])));

// Example function to return "PONG" when a message of "PING" is received from the main thread
const work = () => {
  this.addEventListener("message", e => {
    if (e.data === "PING") postMessage("PONG");
  });
};

let worker = makeWorker(work);
worker.addEventListener("message", e => console.log(e.data))
worker.postMessage("PING");
```

## saveAs
### Save arbitrary data as a file of your choice
```javascript
// Creates a file save prompt for a given input
const saveAs = (data, filename = "untitled") => {
  switch (typeof data) {
    case "undefined":
      throw Error("No data or variable is not yet initialized");
    // Symbols are for in-program use and not meant to be viewed/saved to disk
    case "symbol":
    	throw Error("Symbols cannot be resolved to a serializable form");
    case "string":
    case "number":
    case "bigint":
    case "boolean":
    case "function":
      // Text and numbers are stored as UTF-8 formatted plaintext
      data = new Blob([data], {type: "text/plain;charset=utf-8"});
      break;
    case "object":
      // Weak implementations are meant for in-program use, just like Symbols
      if (data instanceof WeakMap || data instanceof WeakSet)
      	throw Error("WeakSet and WeakMap cannot be enumerated, thus cannot be traversed and saved");
      // Arrays and Sets are stored simply as a comma-delimited list of values
      if (Array.isArray(data)) data = new Blob([data], {type: "text/plain;charset=utf-8"});
      if (data instanceof Set) data = new Blob([[...data]], {type: "text/plain;charset=utf-8"});
      // Maps are converted into key-value pairs, object values are turned into JSON strings, then the
      // keypairs are bundled into a multi-line string separated by Windows style newlines
      if (data instanceof Map) {
        data = [...data].map(x => {
          let [key, value] = [...x];
          switch (typeof value) {
            case "object":
              return `${key} = ${JSON.stringify(value)}`;
            default:
              return `${key} = ${value}`;
          }
        }).join("\r\n");
        data = new Blob([data], {type: "text/plain;charset=utf-8"});
      }
      // Objects without an arrayBuffer property (which would be a Blob) can be turned into a JSON string and saved as such
      if (!data.arrayBuffer) data = new Blob([JSON.stringify(data)], {type: "application/json"});
      break;
    default:
      throw Error("Data type not supported");
  }
  // Turn our Blob into a data uri
  const url = window.URL.createObjectURL(data);
  const a = document.createElement('a');
  // Set the <a> tag attributes to allow a file download
  a.download = filename;
  // Add the data uri
  a.href = url;
  a.style.display = "none";
  // Then append the hidden <a> tag to the body and click it to open a save dialog box
  document.body.appendChild(a);
  a.click();
  // Finally, clean up!
  document.body.removeChild(a);
  window.URL.revokeObjectURL(url);
};
```

## Encrypt / Decrypt
### Simple implementation of (en/de)cryption functions for the browser. PBKDF2 was chosen as it is purposefully slow in order to reduce attack time when trying to bruteforce the encryption password.
```javascript
// Provides two functions to encrypt or decrypt a payload with a password
const getPasswordKey = (password) =>
  window.crypto.subtle.importKey("raw", (new TextEncoder()).encode(password), "PBKDF2", false, ["deriveKey"]);
const deriveKey = (passwordKey, salt, keyUsage) =>
  window.crypto.subtle.deriveKey(
    {
      name: "PBKDF2",
      salt: salt,
      iterations: 696969, // We set a really high number of iterations to protect from brute-force attacks, may be considered excessive in amount (100-250k is okay)
      hash: "SHA-256",
    },
    passwordKey,
    { name: "AES-GCM", length: 256 },
    false,
    keyUsage
  );

 const encryptData = async (secretData, password) => {
  try {
    const salt = window.crypto.getRandomValues(new Uint8Array(16));
    const iv = window.crypto.getRandomValues(new Uint8Array(12));
    const passwordKey = await getPasswordKey(password);
    const aesKey = await deriveKey(passwordKey, salt, ["encrypt"]);
    const encryptedContent = await window.crypto.subtle.encrypt(
      {
        name: "AES-GCM",
        iv: iv,
      },
      aesKey,
      (new TextEncoder()).encode(secretData)
    );

    const encryptedContentArr = new Uint8Array(encryptedContent);
    let buff = new Uint8Array(
      salt.byteLength + iv.byteLength + encryptedContentArr.byteLength
    );
    buff.set(salt, 0);
    buff.set(iv, salt.byteLength);
    buff.set(encryptedContentArr, salt.byteLength + iv.byteLength);
    return btoa(String.fromCharCode.apply(null, buff));
  } catch (e) {
    console.log(`Error - ${e}`);
    return "";
  }
}
const decryptData = async (encryptedData, password) => {
  try {
    const encryptedDataBuff = Uint8Array.from(atob(encryptedData), (c) => c.charCodeAt(null));
    const salt = encryptedDataBuff.slice(0, 16);
    const iv = encryptedDataBuff.slice(16, 28); // 16 + 12
    const data = encryptedDataBuff.slice(28); // 16 + 12
    const passwordKey = await getPasswordKey(password);
    const aesKey = await deriveKey(passwordKey, salt, ["decrypt"]);
    const decryptedContent = await window.crypto.subtle.decrypt(
      {
        name: "AES-GCM",
        iv: iv,
      },
      aesKey,
      data
    );
    return (new TextDecoder()).decode(decryptedContent);
  } catch (e) {
    console.log(`Error - ${e}`);
    return "";
  }
}

const encrypt = async data => await encryptData(data, window.prompt("Password / Passphrase"));
const decrypt = async data =>  await decryptData(data, window.prompt("Password / Passphrase")) || "Incorrect password/passphrase";
```

## sleep
### Javascript-based implementation of the sleep() function
```javascript
const sleep = ms => new Promise(resolve => setTimeout(resolve, ms));
```

## $make
### Helper function to create a DOM element with optional properties, attributes, child elements, and event listeners.
```javascript
const $make = (tag, properties, attributes, children, eventListeners) => {
	const $element = document.createElement(tag);
	if (children) {
		if (children instanceof Array && children.length > 0) {
			$element.append(...children);
		} else if (children instanceof Element) {
			$element.append(children);
		} else if (typeof children === "string") {
			$element.innerHTML = children;
		}
	}
	for (const property in properties) {
		if (typeof properties[property] === "object") {
			Object.assign($element[property], properties[property]);
		} else {
			$element[property] = properties[property];
		}
	}
	for (const attribute in attributes) $element.setAttribute(attribute, attributes[attribute]);
	if (eventListeners) {
		if (eventListeners instanceof Array && eventListeners.length > 0) {
			eventListeners.forEach(({event, function: fn, options}) => {
				$element.addEventListener(event, fn, options);
			});
		} else if (eventListeners instanceof Object && Object.keys(eventListeners).length > 0) {
			const { event, function: fn, options } = eventListeners;
			$element.addEventListener(event, fn, options);
		}
	}
	return $element;
};
```
