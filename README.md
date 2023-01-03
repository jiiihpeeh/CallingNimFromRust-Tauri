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

Here is the `parse_equation.nim`
```
import std/[os,osproc,strutils, tempfiles], jsony, supersnappy,nimpy

const fileName = "parse_equation.py"

proc getModule():string{.compileTime.} =
    return compress(readFile(fileName))

const parseEquationModule = getModule()
var 
    parseEquation : PyObject
    sys : PyObject
    scriptLoaded = false

type 
    NimCall = object 
        call, argument: string

    LaTeXPath = object 
        exists: bool 
        path: string

    LaTeXFile = object 
        content,target: string 

when defined windows:
    const lateXBins = ["xelatex.exe", "pdflatex.exe"]
else:
    const lateXBins = ["xelatex", "pdflatex"]

proc loadScript(folder:string): bool=
    try:
        #check PYTHONHOME to solve problems with AppImage
        echo "loading sys"
        let pyEnv = getEnv("PYTHONHOME")
        if pyEnv.startsWith("/tmp/.mount"):
            let newPath = "/" & pyEnv.split("/")[3..^1].join("/")
            putEnv("PYTHONHOME", newPath)
        echo "PYTHONHOME " & getEnv("PYTHONHOME")
        sys = pyImport("sys")
        echo "imported sys"
        let path = joinPath(folder, fileName)
        if fileExists(path):
            removeFile(path)
        let pychacheDir = joinPath(folder,"__pycache__")
        if dirExists(pychacheDir):
            removeDir(pychacheDir)
        path.writeFile(uncompress(parseEquationModule))
        discard sys.path.append(folder.cstring)
        let moduleName = fileName[0..^4].cstring
        parseEquation = pyImport(moduleName)
        return true
    except:
        return false

proc getLateXPath():LaTeXPath=
    let paths = getEnv("PATH").split(":")
    for d in paths:
        for l in lateXBins:
            let path = joinPath(d, l)
            if fileExists(path):
                result = LaTeXPath(exists: true, path : path)
                break

proc hasLatex():cstring=
    return toJson(getLatexPath()).cstring

proc runLaTeX(args: string):cstring=
    try:
        let latexBin = getLatexPath()
        if latexBin.exists:
            let 
                fileInfo = args.fromJson(LaTeXFile)
                curDir = getCurrentDir()
                tempDir = createTempDir("errorshavelimits", "_temp")
            setCurrentDir(tempDir)
            writeFile("temp.tex", fileInfo.content)
            let exitCode = execCmd(latexBin.path & " temp.tex")
            if exitCode == 0:
                moveFile("temp.pdf", fileInfo.target)
                setCurrentDir(curDir)
                removeDir(tempDir)
                return "true".cstring
            else:
                setCurrentDir(curDir)
                removeDir(tempDir)
        return "false".cstring
    except:
        return "false".cstring

proc writeFileToPath(args:string):cstring=
    try:
        let fileInfo = args.fromJson(LaTeXFile)
        writeFile(fileInfo.target, fileInfo.content)
        return "true".cstring
    except:
        return "false".cstring

proc parse(equation:string):cstring=
    try:
        return parseEquation.parse(equation).to(string).cstring
    except:
        return "false".cstring

proc calculate(equation:string):cstring=
    try:
        return parseEquation.calculate(equation).to(string).cstring
    except:
        return "false".cstring
    
proc callNim*(call: cstring):cstring{.exportc.}=
    #echo "CALLER"
    var calling : NimCall 
    try:
        #echo "NimPy: " & $call
        calling = ($call).fromJson(NimCall)
    except:
        discard

    echo calling
    case calling.call:
    of "runLaTex":
        return runLaTeX(calling.argument)
    of "hasLatex":
        return hasLatex()
    of "write":
        return writeFileToPath(calling.argument)
    of "init":
        echo "initializing Python"
        if not scriptLoaded:
            scriptLoaded = loadScript(calling.argument)
            if scriptLoaded:
                echo "Succesfully loaded SymPy"
                return "true".cstring
            else:
                echo "Failed to load Sympy"
                return "false".cstring
        else:
            echo "Succesfully loaded SymPy"
            return "true".cstring
    of "parse":
        return parse(calling.argument)
    of "calculate":
        return calculate(calling.argument)
    
    return "false".cstring

#echo callSympy("""{"call":"init", "argument": "/tmp"}""")

```
So what this script does: It reads a python module at the compile time and when initialized it writes it back to a disk making a usable module. Yes, I probably should use dunder methods here aka `__method__` but Nim makes it hard and the time penalty is rather low. After the initialization it does parsing and calls to Python. Sure it has some overhead but it is not critical at all. And why I run commands through nim? Well, it is easy and needs some setup which would require lost of await and import stuff via Tauri api.

To compile a nim file as a static library all what is needed  is

```
nim c -d:release --app:staticLib --noMain  parse_equation.nim
```

Simple!


Let's look at `src/main.rs`
```
use libc::c_char;
use std::ffi::CStr;
use std::str;
use std::env;

fn rstr_to_cchar (s:&str) ->   *const c_char {
    let cchar: *const c_char = s.as_ptr() as *const c_char;
    return cchar;
}
fn cchar_to_string (s: *const c_char) -> String {
    let init_str: &CStr = unsafe { CStr::from_ptr(s) };
    let init_slice: &str = init_str.to_str().unwrap();
    let init: String = init_slice.to_owned();
    return init;
}

#[link(name = "libparse_equation", kind = "static")]
extern "C" {
    fn NimMain();
    fn callNim(a: *const c_char) -> *const c_char;
}



fn run_nim(args: &str)-> String {
    // initialize nim gc memory, types and stack
    unsafe {
        NimMain();
    }
    return cchar_to_string(unsafe { callNim(rstr_to_cchar(args)) });
}

#[cfg_attr(
all(not(debug_assertions), target_os = "windows"),
windows_subsystem = "windows"
)]

fn main() {
    tauri::Builder::default()
        .invoke_handler(tauri::generate_handler![nim_caller,get_env, file_exists])
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}

#[tauri::command]
fn nim_caller(name: &str) -> String {
    //println!("{}", name);
    return run_nim(name);
}

#[tauri::command]
fn get_env(name: String) -> String {
    return  env::var(name).unwrap_or("none".to_string());
}
#[tauri::command]
fn file_exists(name: &str) -> bool {
    return std::path::Path::new(name).exists();
}

````
The hardest part here is to map cstring in a right way.
So, why didn't I use Pyo3? I did at first, it shows how complicated it can become. For instance grabbing a variable outside gil's scope is hard if you are not used to it. Besides once figured it is  easy to make these Rust to Nim calls now - see the goal - it becomes more universal.

In order to build it succesfully, `build.rs` file is needed (Rust part of the project's root directory).
````
fn main() {
  // tell rustc to link with some libhello.a library
  println!("cargo:rustc-link=parse_equation");
  // and it should search the Cargo.toml directory for that library
  println!("cargo:rustc-link-search={}", std::env::var("CARGO_MANIFEST_DIR").unwrap());
  //Note that line below has to be uncommented in case for Tauri
  tauri_build::build()
}
````
Compiled static library `libparse_equation.a` needs to be renamed as `liblibparse_equation.a` and needs to be put into the project's root directory (the Rust part).


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
