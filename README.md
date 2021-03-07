# utility-js
An assortment of helpful Javascript functions

## makeWorker
```javascript
// Turns a given function into a WebWorker object
const makeWorker = fn => new Worker(URL.createObjectURL(new Blob([fn.toString().replace(/^.*{\r?\n? */, "").replace(/\r?\n? *}$/, "")])));

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
