# CallingNimFromRust-Tauri
Let's define the objective

Goal: To make Nim Calls from Rust.

Why: Rust can be a huge pain in the ass for beginners. Nailing it takes time. It is unforgiving. Nim is a pretty beginner friendly general purpose systems (glue) language which comes with great perfomance and C type export capabilities.

Target: Tauri.

Since  Tauri is my target it is enough to pass strings as arguments and return strings.
The whole Tauri's Rust business is bit wacky without bindings. Rust is a modern language being somewhere between C and C++  which is very low level considering javascript. 

What this is not for: If you need to make humongous amounts of calls - this is a bit closer to socket communication considering performance which means type conversions and serialization. 


To be considered: I'd think that in case you need more speed and ease of use `protobuf` could serve you better. (https://github.com/PMunch/protobuf-nim). This afaik should use websockets. If communication is not constrained, I'd prefer not to occupy sockets. However, even this phase can be done via websockets provided by Nim either as a static lib or an external binary (you'd probably have to embed the the server binary inside static lib, then write and execute it/ use threading). An external binary should make it easier but a static library makes it easier to claim port numbers and such. 


Brief version --- more details coming


There are some sources that use nim generated dynamic libraries. I'm going to take a different route. Once we use a śtatic library we are dealing less with configuration, plausible directory changes, versioning and  other files.


Backend documentation

https://nim-lang.org/docs/backends.html

Here is the `parse_equation.nim`
```
import std/[os,osproc,strutils, tempfiles, base64], jsony, supersnappy,nimpy, pixie/fileformats/[svg, png], pixie

const fileName = "parse_equation.py"

proc getModule():string{.compileTime.} =
    return compress(readFile(fileName))

const parseEquationModule = getModule()
var 
    parseEquation : PyObject
    sys : PyObject
    scriptLoaded = false

type 
    LaTeXPath = object 
        exists: bool
        path : string

    PngEncode = object 
        result: bool 
        base64: string

    Call = enum 
        runLatex = 0
        hasLatex = 1
        write = 2
        init = 3
        parse = 4
        calculate = 5
        saveSvgAsPng = 6
        getPngFromSvg = 7

    CallObject = object 
        case call : Call
        of runLatex, saveSVGasPNG, write:
            content,target: string
        of hasLatex:
            discard
        of init:
            folder: string
        of parse, calculate:
            argument: string
        of getPngFromSvg:
            svgString: string


when defined windows:
    const lateXBins = ["xelatex.exe", "pdflatex.exe"]
else:
    const lateXBins = ["xelatex", "pdflatex"]


#let None = pyBuiltins().None

# template with_py(context: typed, name: untyped, body: untyped) =
#   try:
#     # Can't use the `.` template
#     let name = callMethod(context, "__enter__")  # context.__enter__()
#     body
#   finally:
#     discard callMethod(context, "__exit__", None, None, None)

template withExecptions(actions: untyped): cstring =
    var output : cstring
    try:
        actions
        output = "true".cstring
    except:
        output = "false".cstring
    output

template withOutputExecption(action: untyped): cstring =
    var output : cstring
    try:
        output = action
    except:
        output = "false".cstring
    output


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
        path.writeFile parseEquationModule.uncompress
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

proc runLaTeX(content: string, target: string)=
    let latexBin = getLatexPath()
    if latexBin.exists:
        let 
            curDir = getCurrentDir()
            tempDir = createTempDir("errorshavelimits", "_temp")
        tempDir.setCurrentDir
        "temp.tex".writeFile content
        let exitCode = execCmd(latexBin.path & " temp.tex")
        if exitCode == 0:
            "temp.pdf".moveFile target
            curDir.setCurrentDir
            tempDir.removeDir
        else:
            curDir.setCurrentDir
            tempDir.removeDir

proc writeFileToPath(content: string, target: string)=
    target.writeFile content


proc parse(equation:string):cstring=
    return parseEquation.parse(equation).to(string).cstring


proc calculate(equation:string):cstring=
    return parseEquation.calculate(equation).to(string).cstring


proc svgToPng(svg:string):string =
    let 
        img =  newImage(parseSvg(svg))
        wt = img.width.float32
        ht = img.height.float32
        scaleF = 0.02
        x = wt * scaleF
        y = ht * scaleF
        image = newImage(x.int, y.int)
    image.fill(rgba(255, 255, 255, 255))

    image.draw(img,scale(vec2(scaleF, scaleF)))
    result = encodePng(image)

proc getPngFromSvg(svg:string):cstring=
    try:
        let png = svgToPng(svg)
        result = PngEncode( result: true, base64: encode(png)).toJson.cstring
    except:
        result = PngEncode( result: false, base64: "").toJson.cstring

proc saveSvgAsPng(content: string, target: string)=
    let png = svgToPng(content)
    target.writeFile png


proc callNim*(call: cstring):cstring{.exportc.}=
    #echo "CALLER"
    var calling : CallObject 
    try:
        #echo "NimPy: " & $call
        calling = ($call).fromJson(CallObject)
    except:
        discard

    echo calling
    case calling.call:
    of runLaTex:
        result = withExecptions: 
            runLaTeX(calling.content, calling.target)
    of hasLatex:
        return hasLatex()
    of write:
        result = withExecptions: 
            writeFileToPath(calling.content, calling.target)
    of init:
        echo "initializing Python"
        if not scriptLoaded:
            scriptLoaded = loadScript(calling.folder)
            if scriptLoaded:
                echo "Succesfully loaded SymPy"
                return "true".cstring
            else:
                echo "Failed to load Sympy"
                return "false".cstring
        else:
            echo "Succesfully loaded SymPy"
            return "true".cstring
    of parse:
        result = withOutputExecption:
            parse(calling.argument)
            
    of calculate:
        result = withOutputExecption:
            calculate(calling.argument)
        
    of saveSVGasPNG:
        result = withExecptions: 
            saveSvgAsPng(calling.content, calling.target)
    of getPngFromSvg:
        return getPngFromSvg(calling.svgString)


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
