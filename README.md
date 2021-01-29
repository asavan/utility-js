# utility-js
An assortment of helpful Javascript functions

## makeWorker
```javascript
// Turns a given function into a WebWorker object
const makeWorker = fn => new Worker(URL.createObjectURL(new Blob([fn.toString().replace(/^.*{\r?\n? */, "").replace(/\r?\n? *}$/, "")])));

const work = () => {
  this.addEventListener("message", e => {
    if (e.data === "PING") postMessage("PONG");
  }
}
let worker = makeWorker(work);
worker.postMessage("PING");
```

## saveAs
```javascript
// Creates a file save prompt for a given input
const saveAs = (data, filename) => {
  switch (typeof data) {
    case "string":
    case "number":
      data = new Blob([data], {type: "text/plain;charset=utf-8"});
      break;
    case "object":
      if (Array.isArray(data)) {
        data = new Blob([data.join('\r\n')], {type: "text/plain;charset=utf-8"});
        break;
      }
      if (!data.size) data = new Blob([JSON.stringify(data)], {type: "application/json"});
      break;
    default:
      break;
  }
  const url = window.URL.createObjectURL(data);
  const a = document.createElement('a');
  a.download = filename || "NEMO";
  a.href = url;
  a.style["display"] = "none";
  document.body.appendChild(a);
  a.click();
  document.body.removeChild(a);
  window.URL.revokeObjectURL(url);
};
```
