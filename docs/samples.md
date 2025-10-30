# Small end‑to‑end examples

**Branching with jmpif / jmpifnot and labels**

```
use log

let x = 10
let y = 7

// Compute condition once: c = (x > y) -> Bool
cmp c gt x y

// If c is true, jump to :GT
jmpif c -> :GT

// If c is false, jump to :LE_OR_EQ
jmpifnot c -> :LE_OR_EQ

// (Fallthrough guard — not reached if either jump happened)
jmp -> :END

:GT
call log.info "x > y"
jmp -> :END

:LE_OR_EQ
call log.info "x <= y"

:END
halt
```

**Simple counted loop using label + jmpif**

```
use log

let n = 3

:LOOP
// If n == 0 -> exit
cmp is_zero eq n 0
jmpif is_zero -> :END

call log.info n
sub n 1
jmp -> :LOOP

:END
halt
```

**Trap/untrap around a failing instruction (missing JSON key)**

```
use log

let doc = {"a":1}

// Push handler: any runtime error jumps to :CATCH with message in `err`
trap :CATCH -> err

// This will fail: key "b" does not exist -> triggers trap
let v = call json.get {"json": doc, "path": "b"}

// If we got here, no error happened — pop the trap
untrap
jmp -> :AFTER

:CATCH
// Always pop the trap when handling; otherwise it stays active
untrap
call log.warn {"caught": err}
let v = 0   // fallback

:AFTER
call log.info {"value": v}
halt
```

**Trap around async call+await to catch HTTP failures**

```
use log
use http

trap :HTTP_ERR -> e

let f = async call http.req {
  "method": "GET",
  "url": "https://httpbun.com/status/500",
  "timeout_ms": 1500,
  "accept_non_2xx": false   // force error on 500
}
let r = await f             // error here jumps to :HTTP_ERR

// Success path
untrap
call log.info r
jmp -> :END

:HTTP_ERR
untrap
call log.error {"http_failed": e}
:END
halt
```

**Generic try/catch template with a single exit**

```
trap :CATCH -> err
  // ... risky instructions ...
  untrap
  jmp -> :AFTER
:CATCH
  untrap
  // ... handle `err` ...
:AFTER
```

**Single-file inline module; one-way jump; no return to main**

```
use log

mod m {
  :RUN
  call log.info "m: hello from module"
  halt               // <-- stop VM here; no back edge
}

jmp -> :m::RUN       // jump once into module; program ends inside
// (unreachable if RUN halts)
```

**Import**

`lib.rasm`

```
/// Library file with a module that halts inside
use log

mod util {
  :RUN
  call log.info "lib::util RUN"
  halt               
}
```

`main.rasm`

```
/// Main that links to lib and does a one-way jump
use log
import "lib.rasm" as lib

call log.info "main: before jump"
jmp -> :lib::util::RUN   // control never returns
// (unreachable if RUN halts)
```

**Shell echo (async) + log**

```
use log
use sh { cwd_root: ".", allowed_cmds: ["echo"] } as sh

:ENTRY
let f = async call sh.run {
  "cmd": "echo",
  "args": ["hey"],
  "capture_stdout": true,
  "capture_stderr": true,
  "timeout_ms": 5000
}
let r = await f
call log.info r
halt
```

**Shell stdin → stdout (cat)**

```
use log
use sh { cwd_root: ".", allowed_cmds: ["cat"] } as sh

let f = async call sh.run {"cmd":"cat","stdin_utf8":"PING","capture_stdout":true}
let r = await f
call log.info r
halt
```

**Parse JSON then branch**

```
use log

let j = call json.parse "{\"n\": 7}"
let seven = call json.get {"json": j, "path": "n"}
cmp c1 gt seven 5
jmpif c1 -> :GT
call log.info "<=5"
jmp -> :END
:GT
call log.info ">5"
:END
halt
```

**Timed select**

```
use log
use http

let f1 = async call http.req {"method":"GET","url":"https://httpbingo.org/bytes/8","accept_non_2xx":true}
let f2 = async call http.req {"method":"GET","url":"https://httpbun.com/bytes/8","accept_non_2xx":true}

select (f1,f2) -> (i,v) timeout 2000
cmp have lt i 2
jmpif have -> :HAVE
call log.warn "timeout"
jmp -> :END
:HAVE
call log.info v
:END
halt
```

**HTTP GET: decode body_base64**

```
use log
use http
use b64

let f = async call http.req {
  "method": "GET",
  "url": "https://httpbingo.org/json",
  "accept_non_2xx": true,
  "headers": {"User-Agent":"relog-asm/cli"}
}
let r = await f
let body_b64 = call json.get {"json": r, "path": "body_base64"}
let body = call b64.decode body_b64
call log.info body
halt
```

**HTTP POST: json body**

```
use log
use http

let hdrs = {"Content-Type": "application/json"}
let body = {"hello": true, "n": 7}

let body_txt = call json.stringify body

let f = async call http.req {
  "method": "POST",
  "url": "https://httpbun.com/post",
  "headers": hdrs,
  "body_utf8": body_txt
}

let r = await
call log.info r
halt
```

**FS: write → read (utf8)**

```
use log
use fs { root: ".", read_only: false }

let _ = await call fs.write {"path":"tmp/hello.txt","data_utf8":"Hi","create":true}
let s = await call fs.read {"path":"tmp/hello.txt","encoding":"utf8"}
call log.info s
halt
```

**FS: list & stat**

```
use log
use fs { root: "." }

let ls = await call fs.list {"path":".","glob":"*.toml","max":10,"sort":"name"}
call log.info ls
let st = await call fs.stat {"path":"Cargo.toml"}
call log.info st
halt
```

**JSON ops: stringify / get / len**

```
use log

let doc = {"a":1,"b":[2,3,4]}
let s = call json.stringify doc
let arr = call json.get {"json": doc, "path": "b"}
let n = call json.len arr
call log.info {"json": s, "len": n}
halt
```

**RAND + MATH + STR (deterministic)**

```
use log
use rand {"engine":"chacha","seed": 42}
use math
use str

let n = call rand.i64 {"lo":0,"hi":10}
let f = call rand.f64 {"lo":0.0,"hi":1.0}
let items = ["relog","asm","vm"]
let one = call rand.choice {"items": items}
let up = call str.upper one
let p = call math.pow {"x": 2, "p": 8}
call log.info {"n": n, "f": f, "pick": up, "pow": p}
halt
```

**EVAL: run nested DSL and export**

```
use log
use eval

let inner = r"
use log
let x = 2
add x 5
call log.info x
"
let res = await call eval.dsl {"code": inner, "exports": ["x"], "allow": ["log"]}
call log.info res
halt
```

**LLM: simple chat (needs env or inline params) use dotenv**

```
use log
use env
use dotenv
use llm
use is

let _ = call dotenv.load null

let endpoint = call env.get "LLM_ENDPOINT"
let model    = call env.get "LLM_MODEL"

let api_token  = call env.get "LLM_API_KEY"
let missing_api = call is.unit api_token
jmpif missing_api -> :NO_API

let f = async call llm.chat {
  "endpoint": endpoint,
  "model":    model,
  "messages": [{"role":"user","content":"Say hi from Relog-ASM"}],
  "api_token":  api_token,
  "timeout_ms": 15000
}
let ans = await f
call log.info ans
halt

:NO_API
call log.error "LLM_API_KEY is missing (.env/process env)"
halt
```
