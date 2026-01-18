# JavaScript Multithreaded Mega Bible (Node.js + Worker Threads)

Yeh example Node.js me "multi-threading" dikhata hai. Worker threads ko tum helper
robots samajh sakte ho. Main thread boss hai, workers kaam karte hain. Isme classes,
abstract class, interface (JSDoc), objects, promises, async/await, event emitter,
shared memory, and kaafi async concepts dikhaye gaye hain.

Note: Explanation extra simple aur playful hai, jaise bachcha samjhe.

---

## File 1: main.js

```js
"use strict";
const os = require("os");
const path = require("path");
const { Worker, MessageChannel, isMainThread } = require("worker_threads");
const { EventEmitter } = require("events");

if (!isMainThread) throw new Error("main.js must run on main thread");

const WORKER_PATH = path.join(__dirname, "worker.js");
const DEFAULT_TIMEOUT_MS = 1500;

/** @interface */
class Logger {
  info(message) {}
  warn(message) {}
  error(message) {}
}

/** @implements {Logger} */
class ConsoleLogger {
  info(message) { console.log(`[INFO] ${message}`); }
  warn(message) { console.warn(`[WARN] ${message}`); }
  error(message) { console.error(`[ERROR] ${message}`); }
}

class AbstractService {
  constructor(name) {
    if (new.target === AbstractService) throw new Error("AbstractService is abstract");
    this.name = name;
  }
  start() { throw new Error("start() must be implemented"); }
  stop() { throw new Error("stop() must be implemented"); }
}

class Task {
  constructor(id, payload) {
    this.id = id;
    this.payload = Object.freeze({ ...payload });
  }
}

const delay = (ms) => new Promise((resolve) => setTimeout(resolve, ms));

function readConfig(cb) { setTimeout(() => cb(null, { jobs: 6, timeoutMs: 1200 }), 10); }

function readConfigAsync() {
  return new Promise((resolve, reject) => {
    readConfig((err, cfg) => (err ? reject(err) : resolve(cfg)));
  });
}

function withTimeout(promise, ms, label) {
  let timerId;
  const timeout = new Promise((_, reject) => {
    timerId = setTimeout(() => reject(new Error(`${label} timed out`)), ms);
  });
  return Promise.race([promise, timeout]).finally(() => clearTimeout(timerId));
}

class WorkerPool extends EventEmitter {
  #workers = new Map();
  #idle = new Set();
  #queue = [];
  #pending = new Map();
  #shared = null;
  #nextId = 1;

  static create({ size, logger }) {
    const sharedBuffer = new SharedArrayBuffer(Int32Array.BYTES_PER_ELEMENT * 2);
    return new WorkerPool(size, logger, sharedBuffer);
  }

  constructor(size, logger, sharedBuffer) {
    super();
    this.size = size;
    this.logger = logger;
    this.#shared = new Int32Array(sharedBuffer);
    for (let i = 0; i < size; i += 1) this.#spawnWorker();
  }

  #spawnWorker() {
    const worker = new Worker(WORKER_PATH, { workerData: { sharedBuffer: this.#shared.buffer } });
    const { port1, port2 } = new MessageChannel();

    worker.on("message", (msg) => this.#handleMessage(worker, msg));
    worker.on("error", (err) => this.logger.error(`worker error: ${err.message}`));
    worker.on("exit", (code) => {
      this.logger.warn(`worker exit: ${code}`);
      this.#workers.delete(worker.threadId);
      this.#idle.delete(worker);
      this.#spawnWorker();
    });

    worker.postMessage({ type: "init", port: port2 }, [port2]);
    port1.on("message", (m) => this.logger.info(`worker-log: ${m}`));

    this.#workers.set(worker.threadId, worker);
    this.#idle.add(worker);
  }

  #handleMessage(worker, msg) {
    if (msg.type === "result") {
      const entry = this.#pending.get(msg.id);
      if (entry) entry.resolve(msg.result);
      this.#pending.delete(msg.id);
      this.#idle.add(worker);
      this.#schedule();
      return;
    }
    if (msg.type === "error") {
      const entry = this.#pending.get(msg.id);
      if (entry) entry.reject(new Error(msg.error));
      this.#pending.delete(msg.id);
      this.#idle.add(worker);
      this.#schedule();
    }
  }

  runTask(payload, timeoutMs = DEFAULT_TIMEOUT_MS) {
    const task = new Task(this.#nextId++, payload);
    const taskPromise = new Promise((resolve, reject) => {
      this.#pending.set(task.id, { resolve, reject });
      this.#queue.push(task);
      this.#schedule();
    });
    return withTimeout(taskPromise, timeoutMs, `task ${task.id}`);
  }

  #schedule() {
    if (this.#queue.length === 0 || this.#idle.size === 0) return;
    const worker = this.#idle.values().next().value;
    const task = this.#queue.shift();
    this.#idle.delete(worker);
    worker.postMessage({ type: "task", task });
  }

  getProgress() {
    const done = Atomics.load(this.#shared, 0);
    const failed = Atomics.load(this.#shared, 1);
    return { done, failed };
  }
}

class ToyService extends AbstractService {
  constructor({ pool, logger }) {
    super("toy");
    this.pool = pool;
    this.logger = logger;
  }
  async start() { this.logger.info("service start"); await delay(5); }
  async runBatch(tasks, timeoutMs) {
    const promises = tasks.map((payload) => this.pool.runTask(payload, timeoutMs));
    return Promise.allSettled(promises);
  }
  async stop() { this.logger.info("service stop"); await delay(5); }
}

async function main() {
  const config = await readConfigAsync();
  const logger = new ConsoleLogger();
  const workerCount = Math.max(2, Math.min(os.cpus().length, 8));
  const pool = WorkerPool.create({ size: workerCount, logger });
  const service = new ToyService({ pool, logger });

  await service.start();

  const tasks = Array.from({ length: config.jobs }, (_, i) => ({
    base: 100 + i,
    discount: 0.05 + i * 0.01,
  }));

  const results = await service.runBatch(tasks, config.timeoutMs);
  logger.info(`progress: ${JSON.stringify(pool.getProgress())}`);
  logger.info(`done: ${results.length}`);

  await service.stop();
}

main().catch((err) => {
  console.error(err);
  process.exit(1);
});
```

### Line-by-line (main.js)
1. `"use strict";` - Rulebook on, galti pakadna easy.
2. `const os = require("os");` - CPU info lane ka tool.
3. `const path = require("path");` - path banana easy ho jata hai.
4. `const { Worker, MessageChannel, isMainThread } = require("worker_threads");` - thread tools la rahe.
5. `const { EventEmitter } = require("events");` - events wala bell system.
6. `(blank line)` - thoda saans lene ka gap.
7. `if (!isMainThread) throw new Error("main.js must run on main thread");` - agar galat jagah se run hua to stop.
8. `(blank line)` - small break.
9. `const WORKER_PATH = path.join(__dirname, "worker.js");` - worker file ka address.
10. `const DEFAULT_TIMEOUT_MS = 1500;` - default wait time 1.5 sec.
11. `(blank line)` - gap.
12. `/** @interface */` - yeh "interface" ka label hai.
13. `class Logger {` - Logger ka shape define.
14. `  info(message) {}` - info method ka wada.
15. `  warn(message) {}` - warning method ka wada.
16. `  error(message) {}` - error method ka wada.
17. `}` - Logger class band.
18. `(blank line)` - gap.
19. `/** @implements {Logger} */` - bol raha hai "main Logger follow karta hun".
20. `class ConsoleLogger {` - real logger class start.
21. `  info(message) { console.log(\`[INFO] ${message}\`); }` - info print karo.
22. `  warn(message) { console.warn(\`[WARN] ${message}\`); }` - warning print.
23. `  error(message) { console.error(\`[ERROR] ${message}\`); }` - error print.
24. `}` - ConsoleLogger band.
25. `(blank line)` - gap.
26. `class AbstractService {` - abstract class start.
27. `  constructor(name) {` - constructor start.
28. `    if (new.target === AbstractService) throw new Error("AbstractService is abstract");` - direct banana mana.
29. `    this.name = name;` - name store.
30. `  }` - constructor end.
31. `  start() { throw new Error("start() must be implemented"); }` - bachcha class ko implement karna hoga.
32. `  stop() { throw new Error("stop() must be implemented"); }` - stop bhi implement karo.
33. `}` - AbstractService band.
34. `(blank line)` - gap.
35. `class Task {` - Task ka blueprint.
36. `  constructor(id, payload) {` - task banate waqt.
37. `    this.id = id;` - id set.
38. `    this.payload = Object.freeze({ ...payload });` - payload ko lock karo.
39. `  }` - constructor end.
40. `}` - Task class band.
41. `(blank line)` - gap.
42. `const delay = (ms) => new Promise((resolve) => setTimeout(resolve, ms));` - wait karne ka magic.
43. `(blank line)` - gap.
44. `function readConfig(cb) { setTimeout(() => cb(null, { jobs: 6, timeoutMs: 1200 }), 10); }` - callback style config.
45. `(blank line)` - gap.
46. `function readConfigAsync() {` - promise style config.
47. `  return new Promise((resolve, reject) => {` - promise ka box.
48. `    readConfig((err, cfg) => (err ? reject(err) : resolve(cfg)));` - error ya success.
49. `  });` - promise band.
50. `}` - function end.
51. `(blank line)` - gap.
52. `function withTimeout(promise, ms, label) {` - timeout wrapper start.
53. `  let timerId;` - timer ka handle.
54. `  const timeout = new Promise((_, reject) => {` - timeout promise.
55. `    timerId = setTimeout(() => reject(new Error(\`${label} timed out\`)), ms);` - time over to reject.
56. `  });` - timeout promise end.
57. `  return Promise.race([promise, timeout]).finally(() => clearTimeout(timerId));` - jo pehle aaye wahi jeet.
58. `}` - function end.
59. `(blank line)` - gap.
60. `class WorkerPool extends EventEmitter {` - worker pool class start.
61. `  #workers = new Map();` - sab workers ka list.
62. `  #idle = new Set();` - free workers ka set.
63. `  #queue = [];` - pending tasks ki line.
64. `  #pending = new Map();` - in-flight promises.
65. `  #shared = null;` - shared memory pointer.
66. `  #nextId = 1;` - task id counter.
67. `(blank line)` - gap.
68. `  static create({ size, logger }) {` - factory method.
69. `    const sharedBuffer = new SharedArrayBuffer(Int32Array.BYTES_PER_ELEMENT * 2);` - shared board.
70. `    return new WorkerPool(size, logger, sharedBuffer);` - pool banake wapas do.
71. `  }` - static end.
72. `(blank line)` - gap.
73. `  constructor(size, logger, sharedBuffer) {` - constructor start.
74. `    super();` - EventEmitter ka super call.
75. `    this.size = size;` - size store.
76. `    this.logger = logger;` - logger store.
77. `    this.#shared = new Int32Array(sharedBuffer);` - shared board ko view banao.
78. `    for (let i = 0; i < size; i += 1) this.#spawnWorker();` - itne workers banao.
79. `  }` - constructor end.
80. `(blank line)` - gap.
81. `  #spawnWorker() {` - naya worker banane ka method.
82. `    const worker = new Worker(WORKER_PATH, { workerData: { sharedBuffer: this.#shared.buffer } });` - worker create.
83. `    const { port1, port2 } = new MessageChannel();` - private chat pipe.
84. `(blank line)` - gap.
85. `    worker.on("message", (msg) => this.#handleMessage(worker, msg));` - worker se message suno.
86. `    worker.on("error", (err) => this.logger.error(\`worker error: ${err.message}\`));` - error log karo.
87. `    worker.on("exit", (code) => {` - worker band ho gaya to.
88. `      this.logger.warn(\`worker exit: ${code}\`);` - warning.
89. `      this.#workers.delete(worker.threadId);` - list se hatao.
90. `      this.#idle.delete(worker);` - idle se hatao.
91. `      this.#spawnWorker();` - naya worker laao.
92. `    });` - exit handler end.
93. `(blank line)` - gap.
94. `    worker.postMessage({ type: "init", port: port2 }, [port2]);` - worker ko port do.
95. `    port1.on("message", (m) => this.logger.info(\`worker-log: ${m}\`));` - worker log suno.
96. `(blank line)` - gap.
97. `    this.#workers.set(worker.threadId, worker);` - worker map me add.
98. `    this.#idle.add(worker);` - worker ko free mark.
99. `  }` - spawn method end.
100. `(blank line)` - gap.
101. `  #handleMessage(worker, msg) {` - message handle function.
102. `    if (msg.type === "result") {` - agar result aaya.
103. `      const entry = this.#pending.get(msg.id);` - pending list se nikalo.
104. `      if (entry) entry.resolve(msg.result);` - promise ko resolve.
105. `      this.#pending.delete(msg.id);` - map saaf.
106. `      this.#idle.add(worker);` - worker free.
107. `      this.#schedule();` - agla task.
108. `      return;` - yahin se wapas.
109. `    }` - if end.
110. `    if (msg.type === "error") {` - agar error aaya.
111. `      const entry = this.#pending.get(msg.id);` - pending nikalo.
112. `      if (entry) entry.reject(new Error(msg.error));` - promise reject.
113. `      this.#pending.delete(msg.id);` - map saaf.
114. `      this.#idle.add(worker);` - worker free.
115. `      this.#schedule();` - agla task.
116. `    }` - error if end.
117. `  }` - handler end.
118. `(blank line)` - gap.
119. `  runTask(payload, timeoutMs = DEFAULT_TIMEOUT_MS) {` - task bhejne ka method.
120. `    const task = new Task(this.#nextId++, payload);` - naya task banao.
121. `    const taskPromise = new Promise((resolve, reject) => {` - promise ka packet.
122. `      this.#pending.set(task.id, { resolve, reject });` - pending me rakho.
123. `      this.#queue.push(task);` - queue me daalo.
124. `      this.#schedule();` - schedule kar do.
125. `    });` - promise end.
126. `    return withTimeout(taskPromise, timeoutMs, \`task ${task.id}\`);` - timeout ke sath do.
127. `  }` - method end.
128. `(blank line)` - gap.
129. `  #schedule() {` - schedule function.
130. `    if (this.#queue.length === 0 || this.#idle.size === 0) return;` - kuch nahi to chhodo.
131. `    const worker = this.#idle.values().next().value;` - ek free worker lo.
132. `    const task = this.#queue.shift();` - queue se task nikalo.
133. `    this.#idle.delete(worker);` - worker busy.
134. `    worker.postMessage({ type: "task", task });` - task bhejo.
135. `  }` - schedule end.
136. `(blank line)` - gap.
137. `  getProgress() {` - progress puchne ka method.
138. `    const done = Atomics.load(this.#shared, 0);` - done count.
139. `    const failed = Atomics.load(this.#shared, 1);` - failed count.
140. `    return { done, failed };` - object bana ke do.
141. `  }` - method end.
142. `}` - WorkerPool class end.
143. `(blank line)` - gap.
144. `class ToyService extends AbstractService {` - service class start.
145. `  constructor({ pool, logger }) {` - constructor.
146. `    super("toy");` - parent constructor.
147. `    this.pool = pool;` - pool store.
148. `    this.logger = logger;` - logger store.
149. `  }` - constructor end.
150. `  async start() { this.logger.info("service start"); await delay(5); }` - start method.
151. `  async runBatch(tasks, timeoutMs) {` - batch method.
152. `    const promises = tasks.map((payload) => this.pool.runTask(payload, timeoutMs));` - har task ka promise.
153. `    return Promise.allSettled(promises);` - sab ka result.
154. `  }` - runBatch end.
155. `  async stop() { this.logger.info("service stop"); await delay(5); }` - stop method.
156. `}` - ToyService end.
157. `(blank line)` - gap.
158. `async function main() {` - main function start.
159. `  const config = await readConfigAsync();` - config ka wait.
160. `  const logger = new ConsoleLogger();` - logger banao.
161. `  const workerCount = Math.max(2, Math.min(os.cpus().length, 8));` - thread count decide.
162. `  const pool = WorkerPool.create({ size: workerCount, logger });` - pool banao.
163. `  const service = new ToyService({ pool, logger });` - service banao.
164. `(blank line)` - gap.
165. `  await service.start();` - service start.
166. `(blank line)` - gap.
167. `  const tasks = Array.from({ length: config.jobs }, (_, i) => ({` - tasks list banao.
168. `    base: 100 + i,` - base value.
169. `    discount: 0.05 + i * 0.01,` - discount value.
170. `  }));` - object band.
171. `(blank line)` - gap.
172. `  const results = await service.runBatch(tasks, config.timeoutMs);` - batch run karo.
173. `  logger.info(\`progress: ${JSON.stringify(pool.getProgress())}\`);` - progress log.
174. `  logger.info(\`done: ${results.length}\`);` - result count log.
175. `(blank line)` - gap.
176. `  await service.stop();` - service stop.
177. `}` - main end.
178. `(blank line)` - gap.
179. `main().catch((err) => {` - main me error pakdo.
180. `  console.error(err);` - error print.
181. `  process.exit(1);` - program stop with error code.
182. `});` - catch end.

---

## File 2: worker.js

```js
"use strict";
const { parentPort, workerData, threadId } = require("worker_threads");

const shared = new Int32Array(workerData.sharedBuffer);
let logPort = null;

const delay = (ms) => new Promise((resolve) => setTimeout(resolve, ms));

function fib(n, memo = new Map()) {
  if (memo.has(n)) return memo.get(n);
  if (n <= 1) return n;
  const value = fib(n - 1, memo) + fib(n - 2, memo);
  memo.set(n, value);
  return value;
}

async function fakeIo(base) {
  await delay(15);
  return base * 2;
}

async function compute(task) {
  const { base, discount } = task.payload;
  const cpu = fib(24 + (task.id % 5));
  const [io] = await Promise.all([fakeIo(base), delay(5)]);
  const price = Math.round((base + (cpu % 10) + io) * (1 - discount));
  return { id: task.id, price, threadId };
}

parentPort.on("message", async (msg) => {
  if (msg.type === "init") {
    logPort = msg.port;
    logPort.postMessage(`ready ${threadId}`);
    return;
  }
  if (msg.type === "task") {
    try {
      const result = await compute(msg.task);
      Atomics.add(shared, 0, 1);
      parentPort.postMessage({ type: "result", id: msg.task.id, result });
    } catch (err) {
      Atomics.add(shared, 1, 1);
      parentPort.postMessage({ type: "error", id: msg.task.id, error: err.message });
    }
  }
});
```

### Line-by-line (worker.js)
1. `"use strict";` - rulebook on.
2. `const { parentPort, workerData, threadId } = require("worker_threads");` - worker tools.
3. `(blank line)` - gap.
4. `const shared = new Int32Array(workerData.sharedBuffer);` - shared board ka view.
5. `let logPort = null;` - log pipe start me empty.
6. `(blank line)` - gap.
7. `const delay = (ms) => new Promise((resolve) => setTimeout(resolve, ms));` - wait helper.
8. `(blank line)` - gap.
9. `function fib(n, memo = new Map()) {` - fib function start.
10. `  if (memo.has(n)) return memo.get(n);` - cache se lo.
11. `  if (n <= 1) return n;` - base case.
12. `  const value = fib(n - 1, memo) + fib(n - 2, memo);` - recursion.
13. `  memo.set(n, value);` - cache me rakho.
14. `  return value;` - answer do.
15. `}` - fib end.
16. `(blank line)` - gap.
17. `async function fakeIo(base) {` - fake I/O start.
18. `  await delay(15);` - thoda wait.
19. `  return base * 2;` - fake result.
20. `}` - fakeIo end.
21. `(blank line)` - gap.
22. `async function compute(task) {` - compute start.
23. `  const { base, discount } = task.payload;` - bag se cheeze nikalo.
24. `  const cpu = fib(24 + (task.id % 5));` - CPU heavy part.
25. `  const [io] = await Promise.all([fakeIo(base), delay(5)]);` - async kaam saath saath.
26. `  const price = Math.round((base + (cpu % 10) + io) * (1 - discount));` - price banao.
27. `  return { id: task.id, price, threadId };` - result object.
28. `}` - compute end.
29. `(blank line)` - gap.
30. `parentPort.on("message", async (msg) => {` - parent se message suno.
31. `  if (msg.type === "init") {` - init message.
32. `    logPort = msg.port;` - log pipe set.
33. `    logPort.postMessage(\`ready ${threadId}\`);` - ready bolo.
34. `    return;` - yahin ruk jao.
35. `  }` - init end.
36. `  if (msg.type === "task") {` - task message.
37. `    try {` - galti se bachne ka shield.
38. `      const result = await compute(msg.task);` - compute call.
39. `      Atomics.add(shared, 0, 1);` - done counter ++.
40. `      parentPort.postMessage({ type: "result", id: msg.task.id, result });` - result bhejo.
41. `    } catch (err) {` - error pakdo.
42. `      Atomics.add(shared, 1, 1);` - failed counter ++.
43. `      parentPort.postMessage({ type: "error", id: msg.task.id, error: err.message });` - error bhejo.
44. `    }` - try/catch end.
45. `  }` - task if end.
46. `});` - message handler end.

---

## Best practices in this example (kid-friendly)
- Worker pool: unlimited workers mat banao, fixed team rakho.
- Timeout wrapper: agar task atak jaye to chhod do.
- Error handling: try/catch aur promise reject se safe rehte ho.
- Immutability: `Object.freeze` se data lock.
- Shared counters: `Atomics` se safe update.
- Promise.allSettled: kuch fail ho tab bhi baaki results milte hain.
- Logging: simple aur clear logs se debugging easy.

---

## Concepts map (kid-friendly)
- Worker threads: helper robots, jo alag kamre me kaam karte hain.
- Event loop: traffic police, kaam ko line me lagata hai.
- Promise: parcel tracking slip, baad me result aata hai.
- async/await: wait karo lekin road block mat karo.
- EventEmitter: doorbell system, events par bell bajta hai.
- SharedArrayBuffer + Atomics: shared blackboard, safe chalk se likhna.
- Map/Set: special box jisme fast dhoondna.
- Class: blueprint of a toy.
- Abstract class: blueprint jiska toy direct nahi banta.
- Interface (JSDoc): promise of shape, jaise "is toy ke ye parts honge".
- Object.freeze: toy ko lock kar dena, koi tod na sake.
- Destructuring: bag se cheeze jaldi nikaalna.
- Spread: items ko copy/paste style me failana.
