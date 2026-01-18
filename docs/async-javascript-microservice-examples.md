# Async JavaScript Microservice (Callbacks, Promises, Async/Await)

Kid-friendly goal (Hinglish): "Hum ek chhota sa microservice banayenge jo toy ka price
nikaalta hai. Isme 2 kaam async honge: (1) fake database se item lena, (2) fake
discount service se discount lena. Phir total price bhej dena."

Below, the SAME microservice is shown in 3 styles:
1) callbacks
2) promises
3) async/await

All 3 behave the same. Only the way we write async code is different.

---

## 1) Callback Style

### Code (single file)
```js
const http = require("http");
const { URL } = require("url");

function fetchItemFromDb(itemId, cb) {
  // Simulated DB I/O: completes later
  setTimeout(() => {
    if (!itemId) return cb(new Error("missing itemId"));
    cb(null, { id: itemId, basePrice: 100 });
  }, 60);
}

function fetchDiscountFromService(itemId, cb) {
  // Simulated HTTP call: completes later
  setTimeout(() => {
    cb(null, { percent: 10 });
  }, 40);
}

function handlePriceRequest(req, res) {
  const url = new URL(req.url, "http://localhost");
  const itemId = url.searchParams.get("itemId");

  // Step A: start async DB work
  fetchItemFromDb(itemId, (dbErr, item) => {
    if (dbErr) {
      res.statusCode = 400;
      return res.end(JSON.stringify({ error: dbErr.message }));
    }

    // Step B: start async discount work
    fetchDiscountFromService(item.id, (discErr, discount) => {
      if (discErr) {
        res.statusCode = 500;
        return res.end(JSON.stringify({ error: "discount failed" }));
      }

      // Step C: compute and respond
      const finalPrice = Math.round(
        item.basePrice * (1 - discount.percent / 100)
      );
      res.setHeader("Content-Type", "application/json");
      res.end(JSON.stringify({ itemId: item.id, price: finalPrice }));
    });
  });
}

const server = http.createServer((req, res) => {
  if (req.url.startsWith("/price")) return handlePriceRequest(req, res);
  res.statusCode = 404;
  res.end("not found");
});

server.listen(8080, () => {
  console.log("callback service on :8080");
});
```

### Execution sequence (simple steps)
1. Request aata hai: `GET /price?itemId=toy1`.
2. `fetchItemFromDb` call hota hai. Yeh function **setTimeout** lagata hai.
3. setTimeout ke baad callback chalti hai -> DB result milta hai.
4. Ab `fetchDiscountFromService` call hota hai, phir setTimeout.
5. Uska callback chalta hai -> discount milta hai.
6. Final price calculate hota hai, response send hota hai.

### Asynchronousity kaise milti hai?
- `setTimeout` Node event loop ko bolta hai: "Is kaam ko baad me karna."
- Event loop free hota hai to callback run hota hai.
- Callback style me async kaam ka result **callback function** se aata hai.

### Diagram (simple)
```
Client -> Service
Service -> fetchItemFromDb (setTimeout)
... time passes ...
Timer Queue -> callback (item ready)
callback -> fetchDiscountFromService (setTimeout)
... time passes ...
Timer Queue -> callback (discount ready)
callback -> respond to Client
```

### Kaun wait karta hai, kab tak, kyun?
- **Wait kaun karta hai?** Callback khud wait nahi karta. **Node event loop** wait karta hai
  ki timer complete ho jaye, phir callback run hota hai.
- **Kab tak wait?** `setTimeout` me diye gaye delay tak (yahan 60ms, 40ms).
- **Kyun wait?** Kyunki async kaam (DB/HTTP) time leta hai. Timer delay us waiting ko
  simulate karta hai.

### Wait ke dauran execution ka kya hota hai?
- `fetchItemFromDb` turant return ho jata hai.
- Main thread block nahi hota. Event loop dusre requests handle kar sakta hai.
- Jab timer complete hota hai, callback queue se uthkar execute hota hai.

### Kis method ka kya kaam?
- `fetchItemFromDb(itemId, cb)`: fake DB call start karna, result callback me dena.
- `fetchDiscountFromService(itemId, cb)`: fake discount call start karna.
- `handlePriceRequest`: request parse, async chain start, final response send.
- `setTimeout`: async delay simulate karna (timer queue).

---

## 2) Promise Style

### Code (single file)
```js
const http = require("http");
const { URL } = require("url");

function fetchItemFromDb(itemId) {
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      if (!itemId) return reject(new Error("missing itemId"));
      resolve({ id: itemId, basePrice: 100 });
    }, 60);
  });
}

function fetchDiscountFromService(itemId) {
  return new Promise((resolve) => {
    setTimeout(() => resolve({ percent: 10 }), 40);
  });
}

function handlePriceRequest(req, res) {
  const url = new URL(req.url, "http://localhost");
  const itemId = url.searchParams.get("itemId");

  fetchItemFromDb(itemId)
    .then((item) => {
      return fetchDiscountFromService(item.id).then((discount) => ({
        item,
        discount,
      }));
    })
    .then(({ item, discount }) => {
      const finalPrice = Math.round(
        item.basePrice * (1 - discount.percent / 100)
      );
      res.setHeader("Content-Type", "application/json");
      res.end(JSON.stringify({ itemId: item.id, price: finalPrice }));
    })
    .catch((err) => {
      res.statusCode = err.message === "missing itemId" ? 400 : 500;
      res.end(JSON.stringify({ error: err.message }));
    });
}

const server = http.createServer((req, res) => {
  if (req.url.startsWith("/price")) return handlePriceRequest(req, res);
  res.statusCode = 404;
  res.end("not found");
});

server.listen(8081, () => {
  console.log("promise service on :8081");
});
```

### Execution sequence (simple steps)
1. Request aata hai.
2. `fetchItemFromDb` call hota hai -> Promise pending.
3. Jab setTimeout complete hota hai, Promise resolve hota hai.
4. `.then` chain chalta hai -> discount fetch start.
5. Discount Promise resolve hota hai -> next `.then`.
6. Price calculate hota hai -> response send.
7. Agar error ho, `.catch` me handle hota hai.

### Asynchronousity kaise milti hai?
- Same event loop + timers.
- Promise resolution microtask queue me `.then` callbacks chalte hain.
- Code "flat" dikhta hai, nesting kam ho jati hai.

### Diagram (simple)
```
Client -> Service
Service -> fetchItemFromDb => Promise pending
... time passes ...
Timer Queue -> resolve(item)
Microtask Queue -> .then (start discount)
Service -> fetchDiscountFromService => Promise pending
... time passes ...
Timer Queue -> resolve(discount)
Microtask Queue -> .then (compute + respond)
```

### Kaun wait karta hai, kab tak, kyun?
- **Wait kaun karta hai?** `then` wala part wait karta hai. Jab tak Promise resolve
  nahi hota, `.then` run nahi hota.
- **Kab tak wait?** Promise resolve hone tak (yahan timer delay ke baad).
- **Kyun wait?** Promise future result represent karta hai, isliye chain ko future me
  chalna hota hai.

### Wait ke dauran execution ka kya hota hai?
- Promise return ho jata hai, function finish ho jata hai.
- Main thread free rehti hai; event loop dusre kaam karta hai.
- Resolve hone par `.then` callbacks **microtask queue** me chale jate hain aur run hote
  hain.

### Kis method ka kya kaam?
- `fetchItemFromDb(itemId)`: Promise return karta hai jo item dega.
- `fetchDiscountFromService(itemId)`: Promise return karta hai jo discount dega.
- `.then(...)`: previous async result aane par next step chalata hai.
- `.catch(...)`: error aane par handle karta hai.

---

## 3) Async/Await Style

### Code (single file)
```js
const http = require("http");
const { URL } = require("url");

function fetchItemFromDb(itemId) {
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      if (!itemId) return reject(new Error("missing itemId"));
      resolve({ id: itemId, basePrice: 100 });
    }, 60);
  });
}

function fetchDiscountFromService(itemId) {
  return new Promise((resolve) => {
    setTimeout(() => resolve({ percent: 10 }), 40);
  });
}

async function handlePriceRequest(req, res) {
  const url = new URL(req.url, "http://localhost");
  const itemId = url.searchParams.get("itemId");

  try {
    const item = await fetchItemFromDb(itemId);
    const discount = await fetchDiscountFromService(item.id);

    const finalPrice = Math.round(
      item.basePrice * (1 - discount.percent / 100)
    );
    res.setHeader("Content-Type", "application/json");
    res.end(JSON.stringify({ itemId: item.id, price: finalPrice }));
  } catch (err) {
    res.statusCode = err.message === "missing itemId" ? 400 : 500;
    res.end(JSON.stringify({ error: err.message }));
  }
}

const server = http.createServer((req, res) => {
  if (req.url.startsWith("/price")) return handlePriceRequest(req, res);
  res.statusCode = 404;
  res.end("not found");
});

server.listen(8082, () => {
  console.log("async/await service on :8082");
});
```

### Execution sequence (simple steps)
1. Request aata hai.
2. `handlePriceRequest` start hota hai, `await fetchItemFromDb`.
3. `await` pe function pause hota hai (main thread block nahi hota).
4. Promise resolve hota hai -> function wapas resume hota hai.
5. `await fetchDiscountFromService` -> pause/resume same way.
6. Price calculate, response send.

### Asynchronousity kaise milti hai?
- `async/await` internally Promises use karta hai.
- `await` Promise resolve hone tak function ko pause karta hai.
- Event loop still free rehta hai, dusre requests handle ho sakte hain.

### Diagram (simple)
```
Client -> Service
Service -> await fetchItemFromDb
Function pauses (event loop free)
... time passes ...
Promise resolves -> function resumes
Service -> await fetchDiscountFromService
Function pauses (event loop free)
... time passes ...
Promise resolves -> function resumes
Service -> respond to Client
```

### Kaun wait karta hai, kab tak, kyun?
- **Wait kaun karta hai?** `await` ke baad ka code wait karta hai. Function **pause**
  hoti hai (sirf us function ka execution).
- **Kab tak wait?** Promise resolve/reject hone tak.
- **Kyun wait?** `await` Promise ke result ko synchronous-style me lene deta hai.

### Wait ke dauran execution ka kya hota hai?
- `handlePriceRequest` pause hota hai, lekin **main thread block nahi hota**.
- Event loop dusre requests handle karta hai.
- Promise resolve hote hi function resume hota hai.

### Kis method ka kya kaam?
- `async function handlePriceRequest`: function ko Promise-returning banata hai.
- `await fetchItemFromDb`: Promise result ka wait + unwrap.
- `await fetchDiscountFromService`: next async result ka wait.
- `try/catch`: async errors handle karta hai.

---

## Quick recap (kid-friendly)
- Callback: "Kaam khatam ho to mujhe bulao."
- Promise: "Result aane par mujhse baat karna."
- Async/await: "Main yahin wait kar raha hoon, par line jam nahi kar raha."

## Super Simple Analogy (kid-friendly)
Socho tumne pizza order kiya:
- Callback: "Jab pizza aaye, mujhe phone kar dena."
- Promise: "Tum mujhe slip de do, jab pizza ready ho, slip pe likh do."
- Async/await: "Main yahin wait kar raha hoon, lekin tum apna kaam karte raho."

Teenon me pizza **late** aata hai (async), bas wait karne ka tareeka alag hai.
