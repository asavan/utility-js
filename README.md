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
const saveAs = (data, filename) => {
  switch (typeof data) {
    case "string":
    case "number":
      // Text and numbers are stored as UTF-8 formatted plaintext
      data = new Blob([data], {type: "text/plain;charset=utf-8"});
      break;
    case "object":
      if (Array.isArray(data)) {
        // Arrays are joined with Windows-style newlines, then saved as UTF-8 formatted plaintext
        data = new Blob([data.join('\r\n')], {type: "text/plain;charset=utf-8"});
        break;
      }
      // However, Objects without a size property (which would be a Blob) can be turned into a JSON string and saved as such
      if (!data.size) data = new Blob([JSON.stringify(data)], {type: "application/json"});
      break;
    default:
      // At this point the object is most likely just a Blob, so we'll leave it alone
      break;
  }
  // Turn our Blob into a data uri
  const url = window.URL.createObjectURL(data);
  const a = document.createElement('a');
  // Set the <a> tag attributes to allow a file download
  a.download = filename || "untitled";
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
