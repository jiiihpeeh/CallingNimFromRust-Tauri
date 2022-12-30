# CallingNimFromRust-Tauri
Let's define the objective

Goal: To make Nim Calls from Rust.

Why: Rust can be a huge pain in the ass for beginners. Nailing it takes time. It is unforgiving. Nim is a pretty beginner friendly general purpose systems (glue) language which comes with great perfomance and C type export capabilities.

Target: Tauri.

Since  Tauri is my target it is enough to pass strings as arguments and return strings.
The whole Tauri's Rust business is bit wacky without bindings. Rust is a modern language being somewhere between C and C++  which is very low level considering javascript. 

What this is not for: If you need to make humongous amounts of calls - this is a bit closer to socket communication considering performance which means type conversions and serialization.


Brief version --- more details coming


There are some sources that use nim generated dynamic libraries. I'm going to take a different route. Once we use a śtatic library we are dealing less with configuration, plausible directory changes, versioning and  other files.


Backend documentation

https://nim-lang.org/docs/backends.html

Here is parse_equation.nim
```
import nimpy,std/os, jsony
proc getModule():string{.compileTime.} =
    return readFile("parse_equation.py")

const parseEquationModule = getModule()
const fileName = "parse_equation.py"
var 
    parseEquation : PyObject
    sys : PyObject

var scriptLoaded = false

type sympyCall = object 
    call : string
    argument: string
        


proc loadScript(folder:cstring)=
    try:
        sys = pyImport("sys")
        let path = joinPath($folder, fileName)
        path.writeFile(parseEquationModule)
        discard sys.path.append(folder)
        let moduleName = fileName[0..^4].cstring
        parseEquation = pyImport(moduleName)
        scriptLoaded = true
    except:
        scriptLoaded = false


proc parse(equation:cstring):cstring=
    return parseEquation.parse(equation).to(string).cstring

proc calculate(equation:cstring):cstring=
    return parseEquation.calculate(equation).to(string).cstring

proc callSympy*(call: cstring):cstring{.exportc.}=
    let calling = ($call).fromJson(sympyCall)
    echo calling
            
    if scriptLoaded:
        if calling.call == "parse":
            return parse(calling.argument)
        elif calling.call  == "calculate":
            return calculate(calling.argument)
        else:
            return "Unknown method".cstring
    else:
        if calling.call == "init":
            echo "initializing Python"
            loadScript(calling.argument)
            if scriptLoaded:
                echo "Succesfully loaded SymPy"
                return "Success".cstring
            else:
                echo "Failed to load Sympy"
                return "Failed".cstring
    return "Can not run Python call".cstring
```
So what this script does: It reads a python module at the compile time and when initialized it writes it back to disk making a usable module. After the initialization it does parsing and calls to Python. Sure it has some overhead but it is not critical at all.
To compile a nim file as a static library all what is needed  is

```
nim c -d:release --app:staticLib --noMain  parse_equation.nim
```

Simple!


Let's look at main.rs
```
use libc::c_char;
use std::ffi::CStr;
use std::str;

#[link(name = "libparse_equation", kind = "static")]
extern "C" {
    fn NimMain();
    fn callSympy(a: *const c_char) -> *const c_char;
}

fn run()-> String {
    // initialize nim gc memory, types and stack
    unsafe {
        NimMain();
    }
    let mut method_str : &str = "{\"call\":\"init\",\"argument\":\"/tmp\"}";
    let mut method: *mut c_char = method_str.as_ptr() as *mut c_char;
    let init_buf: *const c_char = unsafe { callSympy(method) };
    let init_str: &CStr = unsafe { CStr::from_ptr(init_buf) };
    let init_slice: &str = init_str.to_str().unwrap();
    let init: String = init_slice.to_owned(); 
    return init;
}

fn main(){
    let res = run();
    print!("{}", res);
}
````
The hardest part here is to map cstring in a right way.
So, why didn't I use Pyo3? I did at first, it shows how complicated it can become. Like grabbing a variable outside gil scope is hard if you are not used to it. Besides once figured it is now easy to make these Rust to Nim calls - see the goal - it becomes more universal.


Sources

https://github.com/MhadhbiXissam/Example-call-nim-string-from-rust/blob/main/src/main.rs

https://stackoverflow.com/questions/59879692/how-to-call-a-nim-function-from-rust-through-c-ffi#59883589

https://users.rust-lang.org/t/converting-str-to-const-c-char/23115

https://stackoverflow.com/questions/70005417/converting-raw-string-to-string-in-rust

https://nim-lang.org/docs/backends.html

https://users.rust-lang.org/t/how-to-call-the-c-lib-static-library-from-rust/63333/6
