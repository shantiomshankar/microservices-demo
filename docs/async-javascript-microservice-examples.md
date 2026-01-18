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

### Line-by-line (callbacks)
1. `const http = require("http");` — Node ka HTTP module import hota hai.
2. `const { URL } = require("url");` — URL parsing ke liye URL class import hota hai.
3. *(blank line)* — readability ke liye gap.
4. `function fetchItemFromDb(itemId, cb) {` — DB fetch ka helper function start.
5. `  // Simulated DB I/O: completes later` — comment: yeh fake async kaam hai.
6. `  setTimeout(() => {` — async delay start; callback baad me chalega.
7. `    if (!itemId) return cb(new Error("missing itemId"));` — itemId missing ho to error callback.
8. `    cb(null, { id: itemId, basePrice: 100 });` — success me item data callback.
9. `  }, 60);` — delay 60ms set.
10. `}` — function end.
11. *(blank line)* — readability gap.
12. `function fetchDiscountFromService(itemId, cb) {` — discount service helper.
13. `  // Simulated HTTP call: completes later` — comment: fake network call.
14. `  setTimeout(() => {` — async delay start.
15. `    cb(null, { percent: 10 });` — success me discount callback.
16. `  }, 40);` — delay 40ms set.
17. `}` — function end.
18. *(blank line)* — readability gap.
19. `function handlePriceRequest(req, res) {` — request handler start.
20. `  const url = new URL(req.url, "http://localhost");` — incoming URL parse.
21. `  const itemId = url.searchParams.get("itemId");` — query se itemId nikalo.
22. *(blank line)* — readability gap.
23. `  // Step A: start async DB work` — comment: pehla async step.
24. `  fetchItemFromDb(itemId, (dbErr, item) => {` — DB call + callback.
25. `    if (dbErr) {` — error check.
26. `      res.statusCode = 400;` — client error set.
27. `      return res.end(JSON.stringify({ error: dbErr.message }));` — error response send.
28. `    }` — if end.
29. *(blank line)* — readability gap.
30. `    // Step B: start async discount work` — comment: dusra async step.
31. `    fetchDiscountFromService(item.id, (discErr, discount) => {` — discount call + callback.
32. `      if (discErr) {` — error check.
33. `        res.statusCode = 500;` — server error set.
34. `        return res.end(JSON.stringify({ error: "discount failed" }));` — error response.
35. `      }` — if end.
36. *(blank line)* — readability gap.
37. `      // Step C: compute and respond` — comment: final step.
38. `      const finalPrice = Math.round(` — final price start.
39. `        item.basePrice * (1 - discount.percent / 100)` — discount apply.
40. `      );` — Math.round close.
41. `      res.setHeader("Content-Type", "application/json");` — response type set.
42. `      res.end(JSON.stringify({ itemId: item.id, price: finalPrice }));` — success response.
43. `    });` — discount callback end.
44. `  });` — DB callback end.
45. `}` — handler end.
46. *(blank line)* — readability gap.
47. `const server = http.createServer((req, res) => {` — HTTP server create.
48. `  if (req.url.startsWith("/price")) return handlePriceRequest(req, res);` — /price route handle.
49. `  res.statusCode = 404;` — unknown path error.
50. `  res.end("not found");` — 404 response.
51. `});` — server callback end.
52. *(blank line)* — readability gap.
53. `server.listen(8080, () => {` — server port 8080 pe start.
54. `  console.log("callback service on :8080");` — startup log.
55. `});` — listen callback end.

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

### Line-by-line (promises)
1. `const http = require("http");` — HTTP module import.
2. `const { URL } = require("url");` — URL parser import.
3. *(blank line)* — readability gap.
4. `function fetchItemFromDb(itemId) {` — DB helper start.
5. `  return new Promise((resolve, reject) => {` — Promise ban raha hai.
6. `    setTimeout(() => {` — async delay start.
7. `      if (!itemId) return reject(new Error("missing itemId"));` — error reject.
8. `      resolve({ id: itemId, basePrice: 100 });` — success resolve.
9. `    }, 60);` — delay 60ms.
10. `  });` — Promise end.
11. `}` — function end.
12. *(blank line)* — readability gap.
13. `function fetchDiscountFromService(itemId) {` — discount helper start.
14. `  return new Promise((resolve) => {` — Promise ban raha hai.
15. `    setTimeout(() => resolve({ percent: 10 }), 40);` — delay + resolve.
16. `  });` — Promise end.
17. `}` — function end.
18. *(blank line)* — readability gap.
19. `function handlePriceRequest(req, res) {` — request handler start.
20. `  const url = new URL(req.url, "http://localhost");` — URL parse.
21. `  const itemId = url.searchParams.get("itemId");` — itemId read.
22. *(blank line)* — readability gap.
23. `  fetchItemFromDb(itemId)` — Promise start.
24. `    .then((item) => {` — item milne par next step.
25. `      return fetchDiscountFromService(item.id).then((discount) => ({` — discount + combine.
26. `        item,` — item ko object me rakhna.
27. `        discount,` — discount ko object me rakhna.
28. `      }));` — combined object return.
29. `    })` — first then end.
30. `    .then(({ item, discount }) => {` — combined result se next step.
31. `      const finalPrice = Math.round(` — final price start.
32. `        item.basePrice * (1 - discount.percent / 100)` — discount apply.
33. `      );` — Math.round close.
34. `      res.setHeader("Content-Type", "application/json");` — response type set.
35. `      res.end(JSON.stringify({ itemId: item.id, price: finalPrice }));` — success response.
36. `    })` — second then end.
37. `    .catch((err) => {` — error handler start.
38. `      res.statusCode = err.message === "missing itemId" ? 400 : 500;` — status decide.
39. `      res.end(JSON.stringify({ error: err.message }));` — error response.
40. `    });` — catch end.
41. `}` — handler end.
42. *(blank line)* — readability gap.
43. `const server = http.createServer((req, res) => {` — server create.
44. `  if (req.url.startsWith("/price")) return handlePriceRequest(req, res);` — /price route.
45. `  res.statusCode = 404;` — 404 set.
46. `  res.end("not found");` — 404 response.
47. `});` — server callback end.
48. *(blank line)* — readability gap.
49. `server.listen(8081, () => {` — server start 8081.
50. `  console.log("promise service on :8081");` — startup log.
51. `});` — listen end.

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

### Line-by-line (async/await)
1. `const http = require("http");` — HTTP module import.
2. `const { URL } = require("url");` — URL parser import.
3. *(blank line)* — readability gap.
4. `function fetchItemFromDb(itemId) {` — DB helper start.
5. `  return new Promise((resolve, reject) => {` — Promise ban raha hai.
6. `    setTimeout(() => {` — async delay.
7. `      if (!itemId) return reject(new Error("missing itemId"));` — error reject.
8. `      resolve({ id: itemId, basePrice: 100 });` — success resolve.
9. `    }, 60);` — delay 60ms.
10. `  });` — Promise end.
11. `}` — function end.
12. *(blank line)* — readability gap.
13. `function fetchDiscountFromService(itemId) {` — discount helper start.
14. `  return new Promise((resolve) => {` — Promise ban raha hai.
15. `    setTimeout(() => resolve({ percent: 10 }), 40);` — delay + resolve.
16. `  });` — Promise end.
17. `}` — function end.
18. *(blank line)* — readability gap.
19. `async function handlePriceRequest(req, res) {` — async handler start.
20. `  const url = new URL(req.url, "http://localhost");` — URL parse.
21. `  const itemId = url.searchParams.get("itemId");` — itemId read.
22. *(blank line)* — readability gap.
23. `  try {` — error handling start.
24. `    const item = await fetchItemFromDb(itemId);` — item ke liye wait.
25. `    const discount = await fetchDiscountFromService(item.id);` — discount ke liye wait.
26. *(blank line)* — readability gap.
27. `    const finalPrice = Math.round(` — final price start.
28. `      item.basePrice * (1 - discount.percent / 100)` — discount apply.
29. `    );` — Math.round close.
30. `    res.setHeader("Content-Type", "application/json");` — response type set.
31. `    res.end(JSON.stringify({ itemId: item.id, price: finalPrice }));` — success response.
32. `  } catch (err) {` — error capture.
33. `    res.statusCode = err.message === "missing itemId" ? 400 : 500;` — status decide.
34. `    res.end(JSON.stringify({ error: err.message }));` — error response.
35. `  }` — try/catch end.
36. `}` — handler end.
37. *(blank line)* — readability gap.
38. `const server = http.createServer((req, res) => {` — server create.
39. `  if (req.url.startsWith("/price")) return handlePriceRequest(req, res);` — /price route.
40. `  res.statusCode = 404;` — 404 set.
41. `  res.end("not found");` — 404 response.
42. `});` — server callback end.
43. *(blank line)* — readability gap.
44. `server.listen(8082, () => {` — server start 8082.
45. `  console.log("async/await service on :8082");` — startup log.
46. `});` — listen end.

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
