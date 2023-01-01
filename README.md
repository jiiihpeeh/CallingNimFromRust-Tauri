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

Here is the parse_equation.nim
```
import std/os, jsony, supersnappy,nimpy, std/strutils


const fileName = "parse_equation.py"

proc getModule():string{.compileTime.} =
    return compress(readFile(fileName))

const parseEquationModule = getModule()
var 
    parseEquation : PyObject
    sys : PyObject
    scriptLoaded = false

type SympyCall = object 
    call : string
    argument: string
        
proc loadScript(folder:string): bool=
    try:
        echo "loading sys"
        let pyEnv = getEnv("PYTHONHOME")
        #check PYTHONHOME to solve problems with AppImage
        if pyEnv.startsWith("/tmp/.mount"):
            let newPath = "/" & pyEnv.split("/")[3..^1].join("/")
            putEnv("PYTHONHOME", newPath)
        echo "PYTHONHOME " & getEnv("PYTHONHOME")
        sys = pyImport("sys")
        echo "imported sys"
        let path = joinPath(folder, fileName)
        path.writeFile(uncompress(parseEquationModule))
        discard sys.path.append(folder.cstring)
        let moduleName = fileName[0..^4].cstring
        parseEquation = pyImport(moduleName)
        return true
    except:
        return false


proc parse(equation:string):cstring=
    try:
        return parseEquation.parse(equation).to(string).cstring
    except:
        return "Failed".cstring

proc calculate(equation:string):cstring=
    try:
        return parseEquation.calculate(equation).to(string).cstring
    except:
        return "False".cstring

proc callSympy*(call: cstring):cstring{.exportc.}=
    #echo "CALLER"
    var calling : SympyCall 
    try:
        echo "NimPy: " & $call
        calling = ($call).fromJson(SympyCall)
    except:
        discard

    echo calling
            
    if scriptLoaded:
        if calling.call == "parse":
            return parse(calling.argument)
        elif calling.call  == "calculate":
            return calculate(calling.argument)
        elif calling.call == "init":
            return "Initialized".cstring
        else:
            return "Unknown method".cstring
    else:
        if calling.call == "init":
            echo "initializing Python"
            scriptLoaded = loadScript(calling.argument)
            if scriptLoaded:
                echo "Succesfully loaded SymPy"
                return "Initialized".cstring
            else:
                echo "Failed to load Sympy"
                return "Failed".cstring
    return "Can not run Python call".cstring

#echo callSympy("""{"call":"init", "argument": "/tmp"}""")

```
So what this script does: It reads a python module at the compile time and when initialized it writes it back to a disk making a usable module. Yes, I probably should use dunder methods here aka `__method__` but Nim makes it hard and the time penalty is rather low. After the initialization it does parsing and calls to Python. Sure it has some overhead but it is not critical at all.
To compile a nim file as a static library all what is needed  is

```
nim c -d:release --app:staticLib --noMain  parse_equation.nim
```

Simple!


Let's look at src/main.rs
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
So, why didn't I use Pyo3? I did at first, it shows how complicated it can become. For instance grabbing a variable outside gil's scope is hard if you are not used to it. Besides once figured it is  easy to make these Rust to Nim calls now - see the goal - it becomes more universal.

In order to build it succesfully, build.rs file is needed.
````
fn main() {
  // tell rustc to link with some libhello.a library
  println!("cargo:rustc-link=parse_equation");
  // and it should search the Cargo.toml directory for that library
  println!("cargo:rustc-link-search={}", std::env::var("CARGO_MANIFEST_DIR").unwrap());
  //Note that line below has to be uncommented in case for Tauri
  //tauri_build::build()
}
````
Compiled static library libparse_equation.a needs to be renamed as liblibparse_equation.a and needs to be put into the project's root directory.


Release binary can be complied by running
````
cargo run build --release
````




Sources

https://github.com/MhadhbiXissam/Example-call-nim-string-from-rust/blob/main/src/main.rs

https://stackoverflow.com/questions/59879692/how-to-call-a-nim-function-from-rust-through-c-ffi#59883589

https://users.rust-lang.org/t/converting-str-to-const-c-char/23115

https://stackoverflow.com/questions/70005417/converting-raw-string-to-string-in-rust

https://nim-lang.org/docs/backends.html

https://users.rust-lang.org/t/how-to-call-the-c-lib-static-library-from-rust/63333/6
