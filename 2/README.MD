[Using Microsoft Visual Studio 2019 Building A LLVM Out-Source-Tree Pass](https://llvm.org/docs/GettingStartedVS.html)(续)
=========
  * [Command Line Options](#command-line-options)
  * [clang-cl](#clang-cl)
  * [Writing a StringObfuscator LLVM Pass](#writing-a-stringobfuscator-llvm-pass)
  * [Using the StringObfuscator LLVM Pass Processing IR](#using-the-stringobfuscator-llvm-pass-processing-ir)
  * [End](#end)

Command Line Options
==========
  * [clang](https://clang.llvm.org/docs/CommandGuide/clang.html#cmdoption-c)
    ```bash
    $LLVM_DIR/bin/clang.exe -help
    ```
  * [clang-cl](https://clang.llvm.org/docs/UsersManual.html#id9)
    ```bash
    $LLVM_DIR/bin/clang-cl.exe /?
    ```
  * [-cc1 options](https://clang.llvm.org/docs/FAQ.html#i-run-clang-cc1-and-get-weird-errors-about-missing-headers)
    ```bash
    $LLVM_DIR/bin/clang.exe -cc1 -help
    $LLVM_DIR/bin/clang-cl.exe -cc1 -help
    # LLVM官网是说不建议去自己设置 -cc1 的编译选项,容易出问题.
    ```

clang-cl
========
`Visual Studio 2019下使用clang-cl.exe来编译链接`.

`clang-cl`是基于`clang`的扩展, 旨在兼容`Visual C++编译器cl.exe`, 帮你关联头文件和库文件.

```bash
PS K:\llvm\DebugLLVMPass> K:\llvm\build\Debug\bin\clang-cl.exe -### hello1.c -o .\hello1.exe
clang version 10.0.0
Target: x86_64-pc-windows-msvc
Thread model: posix
InstalledDir: K:\llvm\build\Debug\bin

# obj
"K:\\llvm\\build\\Debug\\bin\\clang-cl.exe" "-cc1" "-triple" "x86_64-pc-windows-msvc19.25.28614" "-emit-obj" "-mrelax-all" "-mincremental-linker-compatible" "-disable-free" "-main-file-name" "hello1.c" "-mrelocation-model" "pic" "-pic-level" "2" "-mthread-model" "posix" "-mframe-pointer=none" "-relaxed-aliasing" "-fmath-errno" "-fno-rounding-math" "-masm-verbose" "-mconstructor-aliases" "-munwind-tables" "-target-cpu" "x86-64" "-mllvm" "-x86-asm-syntax=intel" "-D_MT" "-flto-visibility-public-std" "--dependent-lib=libcmt" "--dependent-lib=oldnames" "-stack-protector" "2" "-fms-volatile" "-fdiagnostics-format" "msvc" "-dwarf-column-info" "-resource-dir" "K:\\llvm\\build\\Debug\\lib\\clang\\10.0.0" "-internal-isystem" "K:\\llvm\\build\\Debug\\lib\\clang\\10.0.0\\include" "-internal-isystem" "C:\\Program Files (x86)\\Microsoft Visual Studio\\2019\\Enterprise\\VC\\Tools\\MSVC\\14.25.28610\\include" "-internal-isystem" "C:\\Program Files (x86)\\Microsoft Visual Studio\\2019\\Enterprise\\VC\\Tools\\MSVC\\14.25.28610\\atlmfc\\include" "-internal-isystem" "C:\\Program Files (x86)\\Windows Kits\\10\\Include\\10.0.18362.0\\ucrt" "-internal-isystem" "C:\\Program Files (x86)\\Windows Kits\\10\\include\\10.0.18362.0\\shared" "-internal-isystem" "C:\\Program Files (x86)\\Windows Kits\\10\\include\\10.0.18362.0\\um" "-internal-isystem" "C:\\Program Files (x86)\\Windows Kits\\10\\include\\10.0.18362.0\\winrt" "-fdebug-compilation-dir" "K:\\llvm\\DebugLLVMPass" "-ferror-limit" "19" "-fmessage-length" "147" "-fno-use-cxa-atexit" "-fms-extensions" "-fms-compatibility" "-fms-compatibility-version=19.25.28614" "-fdelayed-template-parsing" "-fobjc-runtime=gcc" "-fdiagnostics-show-option" "-fcolor-diagnostics" "-faddrsig" "-o" "C:\\Users\\XXX\\AppData\\Local\\Temp\\hello1-1b03fd.obj" "-x" "c" "hello1.c"

# link
"C:\\Program Files (x86)\\Microsoft Visual Studio\\2019\\Enterprise\\VC\\Tools\\MSVC\\14.25.28610\\bin\\Hostx64\\x64\\link.exe" "-out:.\\hello1.exe" "-libpath:C:\\Program Files (x86)\\Microsoft Visual Studio\\2019\\Enterprise\\VC\\Tools\\MSVC\\14.25.28610\\lib\\x64" "-libpath:C:\\Program Files (x86)\\Microsoft Visual Studio\\2019\\Enterprise\\VC\\Tools\\MSVC\\14.25.28610\\atlmfc\\lib\\x64" "-libpath:C:\\Program Files (x86)\\Windows Kits\\10\\Lib\\10.0.18362.0\\ucrt\\x64" "-libpath:C:\\Program Files (x86)\\Windows Kits\\10\\Lib\\10.0.18362.0\\um\\x64" "-nologo" "C:\\Users\\XXX\\AppData\\Local\\Temp\\hello1-1b03fd.obj"
```

有关更多详细信息，请阅读 [clang-cl官方文件资料](https://clang.llvm.org/docs/UsersManual.html#clang-cl).

Writing a StringObfuscator LLVM Pass
=============
这边纯粹从小白的角度讲起，说说这个StringObfuscator Pass Code怎么一步一步构建而成，叙述稍微繁复，至于编译调试啥的，你就参照上一篇来整。

以下代码为例：

```
<llvm-tutor source dir>/
    |
    CMakeLists.txt
    <HelloWorld>/
        |
        HelloWorld.cpp
        CMakeLists.txt
    <StringObfuscator>/
        |
        StringObfuscator.cpp
        CMakeLists.txt
```

往llvm-tutor项目创建`StringObfuscator目录`, 目录下创建`StringObfuscator.cpp`，`CMakeLists.txt`，并把`StringObfuscator`项目加入`llvm-tutor\CMakeLists.txt`中编译。

## StringObfuscator/StringObfuscator.cpp

```C
#include "llvm/IR/PassManager.h"
#include "llvm/Passes/PassBuilder.h"
#include "llvm/Passes/PassPlugin.h"
#include "llvm/Support/raw_ostream.h"
#include "llvm/IR/Constants.h"
#include "llvm/IR/ValueSymbolTable.h"
#include "llvm/IR/IRBuilder.h"
#include "llvm/Transforms/Utils/Cloning.h"
#include <typeinfo>

using namespace std;
using namespace llvm;


struct StringObfuscatorModPass : public PassInfoMixin<StringObfuscatorModPass>
{
    PreservedAnalyses run(Module &M, ModuleAnalysisManager &MAM)
    {
        return PreservedAnalyses::all();
    };
};
} // end anonymous namespace

extern "C" ::llvm::PassPluginLibraryInfo LLVM_ATTRIBUTE_WEAK
llvmGetPassPluginInfo()
{
    return {
        LLVM_PLUGIN_API_VERSION, "StringObfuscatorPass", "v0.1", [](PassBuilder &PB) {
            PB.registerPipelineParsingCallback([](StringRef Name, ModulePassManager &MPM,
                     ArrayRef<PassBuilder::PipelineElement>) {
                    if (Name == "string-obfuscator-pass") {
                        MPM.addPass(StringObfuscatorModPass());
                        return true;
                    }
                    return false;
                }
            );
        }
    };
}
```
上面是一个空的`StringObfuscator`插件，什么都没做，接下来要设计这个`Pass`，作用是对全局字符串简单加密。

### `等等！！！`

到这就很懵了，怎么设计你可能知道，但是怎么把设计的写成`Pass`，就让人无从下手了。

恭喜你，抬起头，你会看到LLVM PASS这座大山就再你眼前，到此处，怎么整就是有很大问题了，这是因为你欠缺了很多必备的基础知识，下面我罗列下，如果你都眼熟，那么这山就不是山了是坦途。

  * [LLVM Programmer’s Manual](https://llvm.org/docs/ProgrammersManual.html#important-and-useful-llvm-apis)
  * [LLVM Language Reference Manual](https://llvm.org/docs/LangRef.html)
  * [Writing an LLVM Pass](https://llvm.org/docs/WritingAnLLVMPass.html)
  * Writing an LLVM Pass: 101 [[Video](https://youtu.be/ar7cJl2aBuU)] [[Slides](https://llvm.org/devmtg/2019-10/slides/Warzynski-WritingAnLLVMPass.pdf)]
  * LLVM-IR 层次机构
    ```
    llvm::Module
      llvm::GlobalVariable
      llvm::Function
          llvm::BasicBlock
              llvm::instruction
                  llvm::ICmpInst
                  llvm::BranchInst         
    ```
  * LLVM-IR 命名方式
    ```
    global symbols start with "@"
    local symbols start with "%"
    Basic block names start with "%" when used
    Basic block names end with "%" when defined
    ```
  * 每个Basic block都需要有个Terminator指令为结尾
    ```
    # Terminator
    Ret
    Br
    Switch
    IndirectBr
    Invoke
    Resume
    Unreachable
    CleanupRet
    CatchRet
    CatchSwitch
    CallBr
    ```
  * IRBuilder源码
  * llvm unittests源码
  
  详细看完上面的估计也得有个把月吧，接着我们回到设计，对就是设计，一个简单的字符串加解密。

```C
char *EncodeString(const char *Data, unsigned int lenght)
{
    char *NewData = (char *)malloc(lenght);
    for (unsigned int i = 0; i < lenght; i++) {
        NewData[i] = Data[i] + 1;
    }

    return NewData;
}

void decode(char *Data, unsigned int lenght)
{
    if (Data) {        
        for (unsigned int i = 0; i < lenght; i++) {
            Data[i] -= 1;
        }
    }
}
```

我们的源码`hello1.c`通过`StringObfuscator`的处理加密了全局字符串，为了hello1.exe最终的执行结果正确，我们需要在hello1.exe中存在解密处理。

```C
// 伪代码，实则IR处理
// Befor
char *str1 = "I love ";
int main() {
    printf("%s", str1);
    return 0;
}

// 通过StringObfuscator处理
// After
char *str1 = "J!mpwf!\x01";
void decode(char *Data, unsigned int lenght)
{
    if (Data) {        
        for (unsigned int i = 0; i < lenght; i++) {
            Data[i] -= 1;
        }
    }
}

int main() {
    Decode(str1, 8);
    printf("%s", str1);
    return 0;
}
```

在`StringObfuscator.cpp`中我们需要遍历全局的字符串，通过`EncodeString`函数加密全局字符串并替换，使得全局的字符串以密文形式呈现，在`hello1.ll`中加入`Decode Function`，并在`main`中优先对每个全局字符串调用`Decode`解密，这样就是一个简单的字符串加解密设计。
  * 编译源码中的全局字符串并替换成加密字符串
  * 在`hello1.ll`的`module`中加入`decode函数IR`
  * 在`main`中优先调用

### `生成decode函数IR`

```bash
# Generate an LLVM IR file
$LLVM_DIR/bin/clang-cl.exe -Xclang -emit-llvm -c Decode.c -o Decode.ll
```

```C
; Function Attrs: noinline nounwind optnone sspstrong uwtable
define dso_local void @decode(i8* %Data, i32 %lenght) #1 {
entry:
  %lenght.addr = alloca i32, align 4
  %Data.addr = alloca i8*, align 8
  %i = alloca i32, align 4
  store i32 %lenght, i32* %lenght.addr, align 4
  store i8* %Data, i8** %Data.addr, align 8
  %0 = load i8*, i8** %Data.addr, align 8
  %tobool = icmp ne i8* %0, null
  br i1 %tobool, label %if.then, label %if.end

if.then:                                          ; preds = %entry
  store i32 0, i32* %i, align 4
  br label %for.cond

for.cond:                                         ; preds = %for.inc, %if.then
  %1 = load i32, i32* %i, align 4
  %2 = load i32, i32* %lenght.addr, align 4
  %cmp = icmp ult i32 %1, %2
  br i1 %cmp, label %for.body, label %for.end

for.body:                                         ; preds = %for.cond
  %3 = load i8*, i8** %Data.addr, align 8
  %4 = load i32, i32* %i, align 4
  %idxprom = zext i32 %4 to i64
  %arrayidx = getelementptr inbounds i8, i8* %3, i64 %idxprom
  %5 = load i8, i8* %arrayidx, align 1
  %conv = sext i8 %5 to i32
  %sub = sub nsw i32 %conv, 1
  %conv1 = trunc i32 %sub to i8
  store i8 %conv1, i8* %arrayidx, align 1
  br label %for.inc

for.inc:                                          ; preds = %for.body
  %6 = load i32, i32* %i, align 4
  %inc = add i32 %6, 1
  store i32 %inc, i32* %i, align 4
  br label %for.cond

for.end:                                          ; preds = %for.cond
  br label %if.end

if.end:                                           ; preds = %for.end, %entry
  ret void
}
```

看着是不是很头疼，没事儿，如果你把上面的必备知识都仔细看过，还是能找到入手的地方(`IRBuilder源码`)，LLVM大部分使用`IRBuilder`的来生成`IR代码`，所以参照`IRBuilder源码`可以一葫芦画瓢。

```C
Function *createDecodeFunc(Module &M)
{
    auto &Ctx = M.getContext();

    // Add Decode function
    FunctionType *fn_type = FunctionType::get(Type::getVoidTy(Ctx), {Type::getInt8PtrTy(Ctx), Type::getInt32Ty(Ctx)}, false);
    FunctionCallee DecodeFuncCallee = M.getOrInsertFunction("decode", fn_type);
    Function *DecodeFunc = cast<Function>(DecodeFuncCallee.getCallee());
    DecodeFunc->setCallingConv(CallingConv::C);

    // Name DecodeFunc arguments
    Function::arg_iterator Args = DecodeFunc->arg_begin();
    Value *Data = Args++;
    Data->setName("Data");
    Value *lenght = Args++;
    lenght->setName("lenght");

    // Create blocks 
    BasicBlock *BEntry = BasicBlock::Create(Ctx, "entry", DecodeFunc);
    BasicBlock *BIfThen = BasicBlock::Create(Ctx, "if.then", DecodeFunc);
    BasicBlock *BIfEnd = BasicBlock::Create(Ctx, "if.end", DecodeFunc);
    BasicBlock *BForCond = BasicBlock::Create(Ctx, "for.cond", DecodeFunc);
    BasicBlock *BForBody = BasicBlock::Create(Ctx, "for.body", DecodeFunc);
    BasicBlock *BForInc = BasicBlock::Create(Ctx, "for.inc", DecodeFunc);
    BasicBlock *BForEnd = BasicBlock::Create(Ctx, "for.end", DecodeFunc);

    //// Entry block
    IRBuilder<> B1(BEntry);
    auto lenght_addr = B1.CreateAlloca(B1.getInt32Ty(), nullptr, "lenght.addr");
    auto Data_addr = B1.CreateAlloca(B1.getInt8PtrTy(), nullptr, "Data.addr");
    auto i = B1.CreateAlloca(B1.getInt32Ty(), nullptr, "i");

    B1.CreateStore(lenght, lenght_addr);
    B1.CreateStore(Data, Data_addr);

    auto var0 = B1.CreateLoad(Data_addr);
    auto tobool = B1.CreateICmpNE(var0, Constant::getNullValue(Type::getInt8PtrTy(Ctx)), "tobool");
    B1.CreateCondBr(tobool, BIfThen, BIfEnd);

    // if.then block
    IRBuilder<> B2(BIfThen);
    B2.CreateStore(B2.getInt32(0), i);
    B2.CreateBr(BForCond);

    // for.cond block
    IRBuilder<> B3(BForCond);
    auto var1 = B3.CreateLoad(i);
    auto var2 = B3.CreateLoad(lenght_addr);
    auto cmp = B3.CreateICmpULT(var1, var2, "cmp");
    B3.CreateCondBr(cmp, BForBody, BForEnd);

    // for.body block
    IRBuilder<> B4(BForBody);
    auto var3 = B4.CreateLoad(Data_addr);
    auto var4 = B4.CreateLoad(i);
    auto idxprom = B4.CreateZExt(var4, B4.getInt64Ty(), "idxprom");
    auto arrayidx = B4.CreateInBoundsGEP(B4.getInt8Ty(), var3, idxprom, "arrayidx");

    auto var5 = B4.CreateLoad(arrayidx);
    auto conv = B4.CreateSExt(var5, B4.getInt32Ty(), "conv");
    auto sub = B4.CreateSub(conv, B4.getInt32(1), "sub", false, true);
    auto conv1 = B4.CreateTrunc(sub, B4.getInt8Ty(), "conv1");
    B4.CreateStore(conv1, arrayidx);
    B4.CreateBr(BForInc);

    // for.inc block
    IRBuilder<> B5(BForInc);
    auto var6 = B5.CreateLoad(i);
    auto inc = B5.CreateAdd(var6, B5.getInt32(1), "inc");
    B5.CreateStore(inc, i);
    B5.CreateBr(BForCond);

    // for.end block
    IRBuilder<> B6(BForEnd);
    B6.CreateBr(BIfEnd);

    // End block
    IRBuilder<> B7(BIfEnd);
    B7.CreateRetVoid();

    return DecodeFunc;
}
```

### `遍历全局字符串加密并替换`

这边就贴代码了，没有涉及IR处理，基本上属于llvm::GlobalVariable的获取替换处理。

```C
// 全局字符串存在的形式
char *str1 = "I love "; 
char *str2 = "bananas\n";

char *str3[] = {"11111", "2222"};
```

在`StringObfuscator.cpp`中处理这样的字符串。

```C
class GlobalString
{
public:
    GlobalVariable *Glob;
    unsigned int index;
    int type;

    static const int SIMPLE_STRING_TYPE = 1;
    static const int STRUCT_STRING_TYPE = 2;

    GlobalString(GlobalVariable *Glob) : Glob(Glob), index(-1), type(SIMPLE_STRING_TYPE) {}
    GlobalString(GlobalVariable *Glob, unsigned int index) : Glob(Glob), index(index), type(STRUCT_STRING_TYPE) {}
};

vector<GlobalString *> encodeGlobalStrings(Module &M)
{
    vector<GlobalString *> GlobalStrings;
    auto &Ctx = M.getContext();

    // Encode all global strings
    for (GlobalVariable &Glob : M.globals()) {
        // Ignore external globals & uninitialized globals.
        if (!Glob.hasInitializer() || Glob.hasExternalLinkage())
            continue;

        // Unwrap the global variable to receive its value
        Constant *Initializer = Glob.getInitializer();

        // Check if its a string
        if (isa<ConstantDataArray>(Initializer)) { // 处理str1， str2
            auto CDA = cast<ConstantDataArray>(Initializer);
            if (!CDA->isString())
                continue;

            // Extract raw string
            StringRef StrVal = CDA->getAsString();
            const char *Data = StrVal.begin();
            const int Size = StrVal.size();

            // Create encoded string variable. Constants are immutable so we must override with a new Constant.
            char *NewData = EncodeString(Data, Size);
            Constant *NewConst = ConstantDataArray::getString(Ctx, StringRef(NewData, Size), false);

            // Overwrite the global value
            Glob.setInitializer(NewConst);

            GlobalStrings.push_back(new GlobalString(&Glob));
            Glob.setConstant(false);
        }
        else if (isa<ConstantStruct>(Initializer)) { // 处理str3
            // Handle structs
            auto CS = cast<ConstantStruct>(Initializer);
            if (Initializer->getNumOperands() > 1)
                continue; // TODO: Fix bug when removing this: "Constant not found in constant table!"
            for (unsigned int i = 0; i < Initializer->getNumOperands(); i++) {
                auto CDA = dyn_cast<ConstantDataArray>(CS->getOperand(i));
                if (!CDA || !CDA->isString())
                    continue;

                // Extract raw string
                StringRef StrVal = CDA->getAsString();
                const char *Data = StrVal.begin();
                const int Size = StrVal.size();

                // Create encoded string variable
                char *NewData = EncodeString(Data, Size);
                Constant *NewConst = ConstantDataArray::getString(Ctx, StringRef(NewData, Size), false);

                // Overwrite the struct member
                CS->setOperand(i, NewConst);

                GlobalStrings.push_back(new GlobalString(&Glob, i));
                Glob.setConstant(false);
            }
        }
    }

    return GlobalStrings;
}
```

### `遍历字符串并调用decode解密函数IR`

```C
Function *createDecodeStubFunc(Module &M, vector<GlobalString *> &GlobalStrings, Function *DecodeFunc)
{
    auto &Ctx = M.getContext();
    // Add DecodeStub function
    FunctionCallee DecodeStubCallee = M.getOrInsertFunction("decode_stub", Type::getVoidTy(Ctx));
    Function *DecodeStubFunc = cast<Function>(DecodeStubCallee.getCallee());
    DecodeStubFunc->setCallingConv(CallingConv::C);

    // Create entry block
    BasicBlock *BB = BasicBlock::Create(Ctx, "entry", DecodeStubFunc);
    IRBuilder<> Builder(BB);

    // Add calls to decode every encoded global
    for (GlobalString *GlobString : GlobalStrings) {
        if (GlobString->type == GlobString->SIMPLE_STRING_TYPE) {
            Constant *Zero = ConstantInt::get(Type::getInt32Ty(Ctx), 0);
            Value *Idx[] = {Zero, Zero};
            auto StrPtr = Builder.CreateInBoundsGEP(GlobString->Glob->getValueType(), GlobString->Glob, Idx);

            // Extract raw string
            auto CDA = dyn_cast<ConstantDataArray>(GlobString->Glob->getInitializer());
            StringRef StrVal = CDA->getAsString();
            size_t Size = StrVal.size();

            Builder.CreateCall(DecodeFunc, {StrPtr, ConstantInt::get(Builder.getInt32Ty(), Size)});
        }
        else if (GlobString->type == GlobString->STRUCT_STRING_TYPE) {
            auto StrPtr = Builder.CreateStructGEP(GlobString->Glob, GlobString->index);

            // Extract raw string
            auto CDA = dyn_cast<ConstantDataArray>(GlobString->Glob->getInitializer()->getOperand(GlobString->index));
            StringRef StrVal = CDA->getAsString();
            size_t Size = StrVal.size();

            Builder.CreateCall(DecodeFunc, {StrPtr, ConstantInt::get(Builder.getInt32Ty(), Size)});
        }
    }

    Builder.CreateRetVoid();

    // log IR info
    std::string str;
    raw_string_ostream OS(str);
    DecodeStubFunc->print(OS);
    errs() << str;

    return DecodeStubFunc;
}
```

### `在main中优先调用`

```C
void createDecodeStubBlock(Module &M, Function *DecodeStubFunc)
{
    Function *F = M.getFunction("main");
    auto &Ctx = F->getContext();
    BasicBlock &EntryBlock = F->getEntryBlock();

    auto Inst = EntryBlock.begin();
    // // Create new block
    // BasicBlock *NewBB = BasicBlock::Create(Ctx, "DecodeStub", EntryBlock.getParent(), &EntryBlock);
    // IRBuilder<> Builder(NewBB);
    // 
    // // Call stub func instruction
    // Builder.CreateCall(DecodeStubFunc);
    // // Jump to original entry block
    // Builder.CreateBr(&EntryBlock);

    IRBuilder<> Builder(&*Inst);
    Builder.CreateCall(DecodeStubFunc);
}
```

```
#原始                #注释部分                  #现有
[ entry ]           [ DecodeStub ]            [ entry ]                      
    |                     |                       |
    v                     v                       v
                      [ entry ]               DecodeStub
                          |                       |
                          v                       v
```

### `在struct StringObfuscatorModPass : public PassInfoMixin<StringObfuscatorModPass>接口run中完成这一系列流程`

```C
struct StringObfuscatorModPass : public PassInfoMixin<StringObfuscatorModPass>
{
    PreservedAnalyses run(Module &M, ModuleAnalysisManager &MAM)
    {
        // Transform the strings
        auto GlobalStrings = encodeGlobalStrings(M);

        // Inject functions
        Function *DecodeFunc = createDecodeFunc(M);
        Function *DecodeStub = createDecodeStubFunc(M, GlobalStrings, DecodeFunc);

        // Inject a call to DecodeStub from main
        createDecodeStubBlock(M, DecodeStub);

        return PreservedAnalyses::all();
    };
};

```

至此，整个`StringObfuscator LLVM Pass`总算是成型了，千里之行始于足下。


Using the StringObfuscator LLVM Pass Processing IR
=============
### 生成HelloWorld.o

Visual Studio 2019下需要使用`clang-cl.exe`来编译连接.

```bash
# Generate an LLVM IR file
$LLVM_DIR/bin/clang-cl.exe -Xclang -emit-llvm -c hello1.c -o hello1.ll

# HelloWorld pass Processing an LLVM IR file
$LLVM_DIR/bin/opt.exe -load-pass-plugin StringObfuscator.dll -passes=string-obfuscator-pass -S hello1.ll -o output_hello.ll
```

### 生成output_hello.o文件
```bash
# 方法一
# Generate an LLVM obj file
$LLVM_DIR/bin/bin/llc.exe output_hello.ll -o output_hello.s
$LLVM_DIR/bin/bin/clang-cl.exe -c output_hello.s -o output_hello.o

# 方法二
# Generate an LLVM obj file
$LLVM_DIR/bin/clang-cl.exe -c output_hello.ll -o output_hello.o
```

### 生成output_hello.exe文件
```bash
# Generate an LLVM exe file
$LLVM_DIR/bin/clang-cl.exe output_hello.ll -o output_hello.exe

# 详情参考 clang-cl -### 编译链接构建过程参数细节
$LLVM_DIR/bin/clang-cl.exe -### hello.c -o hello.exe
```

### 原
![output_hello1](https://github.com/Amoystyle/LLVM-VS/blob/master/2/1.png)

### 改
![output_hello2](https://github.com/Amoystyle/LLVM-VS/blob/master/2/2.png)
![output_hello2.1](https://github.com/Amoystyle/LLVM-VS/blob/master/2/2.1.png)

End
==========
  * `StringObfuscator`项目借鉴于此[Github](https://github.com/tsarpaul/llvm-string-obfuscator)，不过在学习调试过程发现原项目的Decode函数解密过程中有问题，便有了此篇文章，也算是归纳总结。

## 有问题可以关注下我的[Github](https://github.com/Amoystyle/LLVM-VS)
