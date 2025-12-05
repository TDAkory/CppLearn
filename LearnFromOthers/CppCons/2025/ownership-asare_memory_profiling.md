# [Is Your C++ Code Leaking Memory? Discover the Power of Ownership-Aware Profiling - Alecto Irene Perez - CppCon 2025](https://github.com/CppCon/CppCon2025/blob/main/Presentations/Where'd_all_that_RAM_Go.pdf)

- [youtube](https://www.youtube.com/watch?v=U23WkMWIkkE)
- [github repo](https://github.com/codeinred/mem_profile)

**作者的核心思路是利用编译器插件技术，在对象析构时进行内存统计分析**。下述总结来自deepseek

创建一个让 GCC 编译器重写对象析构函数的库涉及到 GCC 插件开发或编译器传递（pass）编写。以下是几种实现方法的详细说明：

## 1. **使用 GCC 插件 API（推荐）**

### 基本插件结构

```c
/* gcc_destructor_plugin.c */
#include <gcc-plugin.h>
#include <plugin-version.h>
#include <tree.h>
#include <tree-pass.h>
#include <context.h>
#include <function.h>
#include <gimple.h>
#include <gimple-iterator.h>

int plugin_is_GPL_compatible = 1;

/* 插件信息 */
static struct plugin_info my_plugin_info = {
    .version = "1.0",
    .help = "Rewrite object destructors plugin",
};

/* 我们的析构函数重写传递 */
static unsigned int rewrite_destructors_execute(void)
{
    struct cgraph_node *node;
    
    /* 遍历所有函数 */
    FOR_EACH_FUNCTION(node) {
        tree fndecl = node->decl;
        const char *name = IDENTIFIER_POINTER(DECL_NAME(fndecl));
        
        /* 查找析构函数 */
        if (strstr(name, "~")) {
            printf("Found destructor: %s\n", name);
            
            /* 获取函数体 */
            tree fntype = TREE_TYPE(fndecl);
            tree fnbody = DECL_SAVED_TREE(fndecl);
            
            if (fnbody) {
                /* 在这里修改函数体 */
                /* ... */
            }
        }
    }
    
    return 0;
}

/* 传递定义 */
static struct gimple_opt_pass rewrite_destructors_pass = {
    .pass.type = GIMPLE_PASS,
    .pass.name = "rewrite-destructors",
    .pass.gate = NULL,  /* 总是运行 */
    .pass.execute = rewrite_destructors_execute,
    .pass.sub = NULL,
    .pass.next = NULL,
    .pass.static_pass_number = 0,
    .pass.tv_id = TV_NONE,
    .pass.properties_required = 0,
    .pass.properties_provided = 0,
    .pass.properties_destroyed = 0,
    .pass.todo_flags_start = 0,
    .pass.todo_flags_finish = 0
};

/* 插件初始化函数 */
int plugin_init(struct plugin_name_args *plugin_info,
                struct plugin_gcc_version *version)
{
    if (!plugin_default_version_check(version, &gcc_version))
        return 1;
    
    /* 注册插件信息 */
    register_callback(plugin_info->base_name,
                      PLUGIN_INFO,
                      NULL,
                      &my_plugin_info);
    
    /* 注册我们的传递 */
    struct register_pass_info pass_info;
    pass_info.pass = &rewrite_destructors_pass.pass;
    pass_info.reference_pass_name = "ssa";
    pass_info.ref_pass_instance_number = 1;
    pass_info.pos_op = PASS_POS_INSERT_AFTER;
    
    register_callback(plugin_info->base_name,
                      PLUGIN_PASS_MANAGER_SETUP,
                      NULL,
                      &pass_info);
    
    return 0;
}
```

### 编译插件

```bash
# 编译插件
gcc -I`gcc -print-file-name=plugin`/include -fPIC -shared \
    -o gcc_destructor_plugin.so gcc_destructor_plugin.c

# 使用插件编译代码
gcc -fplugin=./gcc_destructor_plugin.so -c myprogram.c
```

## 2. **使用 LLVM Pass（替代方案）**

```cpp
/* RewriteDestructors.cpp - LLVM Pass */
#include "llvm/Pass.h"
#include "llvm/IR/Function.h"
#include "llvm/IR/Module.h"
#include "llvm/IR/Instructions.h"
#include "llvm/IR/IRBuilder.h"
#include "llvm/Support/raw_ostream.h"
#include "llvm/Transforms/Utils/Cloning.h"

using namespace llvm;

namespace {
    class RewriteDestructorsPass : public ModulePass {
    public:
        static char ID;
        RewriteDestructorsPass() : ModulePass(ID) {}
        
        bool runOnModule(Module &M) override {
            bool changed = false;
            
            for (Function &F : M) {
                if (isDestructor(F)) {
                    errs() << "Found destructor: " << F.getName() << "\n";
                    
                    // 重写析构函数
                    if (rewriteDestructor(F)) {
                        changed = true;
                    }
                }
            }
            
            return changed;
        }
        
    private:
        bool isDestructor(Function &F) {
            StringRef name = F.getName();
            
            // 检查C++析构函数命名模式
            if (name.startswith("_ZN") && name.contains("D")) {
                // 简化的析构函数检测
                return name.contains("D1Ev") ||  // 完全析构
                       name.contains("D2Ev") ||  // 基类析构
                       name.contains("D0Ev");    // 删除析构
            }
            
            return false;
        }
        
        bool rewriteDestructor(Function &F) {
            // 创建一个新的函数体
            BasicBlock *BB = BasicBlock::Create(F.getContext(), "entry", &F);
            IRBuilder<> Builder(BB);
            
            // 获取this指针
            Argument *ThisPtr = &*F.arg_begin();
            
            // 添加日志调用
            FunctionType *LogType = FunctionType::get(
                Builder.getVoidTy(),
                {Builder.getInt8PtrTy()},
                false
            );
            
            // 创建或获取日志函数
            Function *LogFunc = dyn_cast<Function>(
                F.getParent()->getOrInsertFunction(
                    "log_destructor_call",
                    LogType
                ).getCallee()
            );
            
            // 调用日志函数
            Builder.CreateCall(LogFunc, {
                Builder.CreateBitCast(ThisPtr, Builder.getInt8PtrTy())
            });
            
            // 返回
            if (F.getReturnType()->isVoidTy()) {
                Builder.CreateRetVoid();
            } else {
                Builder.CreateRet(ThisPtr);
            }
            
            // 删除旧的基本块
            for (auto &BB : make_early_inc_range(F)) {
                if (&BB != BB) {  // 跳过我们创建的块
                    BB.dropAllReferences();
                    BB.eraseFromParent();
                }
            }
            
            return true;
        }
    };
}

char RewriteDestructorsPass::ID = 0;

// 注册Pass
static RegisterPass<RewriteDestructorsPass> X(
    "rewrite-destructors",
    "Rewrite C++ destructors",
    false,  // 只查看CFG
    false   // 是分析Pass
);
```

### 编译和使用 LLVM Pass

```bash
# 编译为共享库
clang++ -shared -fPIC `llvm-config --cxxflags --ldflags --system-libs --libs core` \
    -o RewriteDestructors.so RewriteDestructors.cpp

# 使用opt工具应用Pass
clang -S -emit-llvm myprogram.cpp -o myprogram.ll
opt -load ./RewriteDestructors.so -rewrite-destructors myprogram.ll -S -o modified.ll
llc modified.ll -o modified.s
g++ modified.s -o myprogram
```

## 3. **使用 GCC 属性扩展**

```c
/* destructor_hook.h */
#ifndef DESTRUCTOR_HOOK_H
#define DESTRUCTOR_HOOK_H

#ifdef __cplusplus
extern "C" {
#endif

/* GCC属性声明 */
#define DESTRUCTOR_HOOK __attribute__((destructor_hook))

/* 注册析构函数钩子的函数 */
void register_destructor_hook(void *object, void (*hook)(void*));

/* 移除析构函数钩子 */
void unregister_destructor_hook(void *object);

#ifdef __cplusplus
}
#endif

#endif /* DESTRUCTOR_HOOK_H */
```

```c
/* destructor_hook_plugin.c - GCC插件实现属性 */
#include <gcc-plugin.h>
#include <plugin-version.h>
#include <tree.h>
#include <tree-pass.h>
#include <gimple.h>

/* 自定义属性处理器 */
static tree handle_destructor_hook_attribute(tree *node, tree name,
                                            tree args, int flags, bool *no_add_attrs)
{
    /* 验证属性使用 */
    if (TREE_CODE(*node) != FUNCTION_DECL) {
        error("%qE attribute can only be applied to functions", name);
        *no_add_attrs = true;
    }
    
    return NULL_TREE;
}

/* 属性定义 */
static struct attribute_spec destructor_hook_attr = {
    .name = "destructor_hook",
    .min_length = 0,
    .max_length = 0,
    .decl_required = true,
    .type_required = false,
    .function_type_required = false,
    .affects_type_identity = false,
    .handler = handle_destructor_hook_attribute
};

/* 插件初始化 */
int plugin_init(struct plugin_name_args *plugin_info,
                struct plugin_gcc_version *version)
{
    if (!plugin_default_version_check(version, &gcc_version))
        return 1;
    
    /* 注册自定义属性 */
    register_attribute(&destructor_hook_attr);
    
    return 0;
}
```

## 4. **使用 LD_PRELOAD 和符号拦截**

```cpp
/* destructor_intercept.cpp - 运行时库 */
#define _GNU_SOURCE
#include <dlfcn.h>
#include <iostream>
#include <map>
#include <memory>

// 原始析构函数类型
typedef void (*DestructorFunc)(void*);

// 全局钩子映射
static std::map<void*, std::function<void(void*)>> g_destructor_hooks;

extern "C" {

// 拦截operator delete
void operator delete(void* ptr) noexcept {
    // 检查是否有钩子
    auto it = g_destructor_hooks.find(ptr);
    if (it != g_destructor_hooks.end()) {
        // 调用自定义钩子
        it->second(ptr);
        g_destructor_hooks.erase(it);
    }
    
    // 调用原始的operator delete
    static void (*real_delete)(void*) = nullptr;
    if (!real_delete) {
        real_delete = (void(*)(void*))dlsym(RTLD_NEXT, "_ZdlPv");
    }
    
    if (real_delete) {
        real_delete(ptr);
    }
}

// 拦截operator delete[]
void operator delete[](void* ptr) noexcept {
    // 类似处理...
    operator delete(ptr);
}

// 注册析构函数钩子
void __attribute__((visibility("default")))
register_destructor_hook(void* obj, void (*hook)(void*)) {
    g_destructor_hooks[obj] = hook;
}

// 带上下文的注册函数
void __attribute__((visibility("default")))
register_destructor_hook_ex(void* obj, void* context, 
                            void (*hook)(void*, void*)) {
    g_destructor_hooks[obj] = [context, hook](void* ptr) {
        hook(ptr, context);
    };
}

} // extern "C"
```

编译和使用：
```bash
# 编译为共享库
g++ -shared -fPIC -ldl -o libdestructor_hook.so destructor_intercept.cpp

# 使用预加载
LD_PRELOAD=./libdestructor_hook.so ./your_program
```

## 5. **完整的 GCC 插件示例：重写析构函数**

```c
/* complete_destructor_plugin.c */
#include <gcc-plugin.h>
#include <plugin-version.h>
#include <tree.h>
#include <tree-pass.h>
#include <gimple.h>
#include <gimple-iterator.h>
#include <stringpool.h>
#include <attribs.h>
#include <cgraph.h>
#include <print-tree.h>

int plugin_is_GPL_compatible = 1;

/* 找到析构函数调用并重写它们 */
static gimple *find_destructor_call(gimple_stmt_iterator *gsi)
{
    gimple *stmt = gsi_stmt(*gsi);
    
    if (gimple_code(stmt) == GIMPLE_CALL) {
        tree fndecl = gimple_call_fndecl(stmt);
        if (fndecl) {
            const char *name = IDENTIFIER_POINTER(DECL_NAME(fndecl));
            
            // 查找析构函数调用
            if (strstr(name, "~") || 
                strstr(name, "_ZN") || 
                strstr(name, "D1Ev") ||
                strstr(name, "D2Ev")) {
                return stmt;
            }
        }
    }
    
    return NULL;
}

/* 创建日志调用语句 */
static gimple *create_log_call(tree this_ptr, const char *class_name)
{
    // 构建日志函数声明
    tree log_func_type = build_function_type_list(
        void_type_node,
        ptr_type_node,  // this指针
        build_pointer_type(build_type_variant(char_type_node, 1)), // 类名
        NULL_TREE
    );
    
    tree log_func = build_fn_decl("__log_destructor", log_func_type);
    TREE_PUBLIC(log_func) = 1;
    DECL_EXTERNAL(log_func) = 1;
    
    // 创建调用
    tree class_name_str = build_string_literal(strlen(class_name) + 1, class_name);
    
    return gimple_build_call(log_func, 2,
                            build1(ADDR_EXPR, ptr_type_node, this_ptr),
                            class_name_str);
}

/* 主传递执行函数 */
static unsigned int rewrite_destructors_execute(void)
{
    struct cgraph_node *node;
    
    FOR_EACH_FUNCTION(node) {
        tree fndecl = node->decl;
        
        // 检查是否是析构函数
        if (TREE_CODE(fndecl) == FUNCTION_DECL) {
            const char *name = IDENTIFIER_POINTER(DECL_NAME(fndecl));
            
            if (strstr(name, "D1Ev") || strstr(name, "D2Ev")) {
                printf("Processing destructor: %s\n", name);
                
                // 获取函数体
                push_cfun(DECL_STRUCT_FUNCTION(fndecl));
                
                basic_block bb;
                gimple_stmt_iterator gsi;
                
                // 遍历所有基本块
                FOR_EACH_BB_FN(bb, cfun) {
                    for (gsi = gsi_start_bb(bb); !gsi_end_p(gsi); gsi_next(&gsi)) {
                        gimple *stmt = gsi_stmt(gsi);
                        
                        // 查找析构函数调用
                        if (find_destructor_call(&gsi)) {
                            // 获取this指针（通常是第一个参数）
                            tree this_ptr = gimple_call_arg(stmt, 0);
                            
                            // 创建日志调用
                            gimple *log_call = create_log_call(this_ptr, name);
                            
                            // 在原始调用前插入日志
                            gsi_insert_before(&gsi, log_call, GSI_SAME_STMT);
                        }
                    }
                }
                
                pop_cfun();
            }
        }
    }
    
    return 0;
}

/* 传递定义 */
static struct gimple_opt_pass rewrite_destructors_pass = {
    .pass = {
        .type = GIMPLE_PASS,
        .name = "rewrite-destructors",
        .gate = NULL,
        .execute = rewrite_destructors_execute,
        .sub = NULL,
        .next = NULL,
        .static_pass_number = 0,
        .tv_id = TV_NONE,
        .properties_required = 0,
        .properties_provided = 0,
        .properties_destroyed = 0,
        .todo_flags_start = 0,
        .todo_flags_finish = 0
    }
};

/* 插件初始化 */
int plugin_init(struct plugin_name_args *plugin_info,
                struct plugin_gcc_version *version)
{
    if (!plugin_default_version_check(version, &gcc_version))
        return 1;
    
    /* 注册传递 */
    struct register_pass_info pass_info = {
        .pass = &rewrite_destructors_pass.pass,
        .reference_pass_name = "ssa",
        .ref_pass_instance_number = 1,
        .pos_op = PASS_POS_INSERT_AFTER
    };
    
    register_callback(plugin_info->base_name,
                      PLUGIN_PASS_MANAGER_SETUP,
                      NULL,
                      &pass_info);
    
    return 0;
}
```

## 6. **Makefile 自动化构建**

```makefile
# Makefile for GCC destructor plugin

PLUGIN_NAME = destructor_rewrite
PLUGIN_SO = $(PLUGIN_NAME).so
PLUGIN_SRC = $(PLUGIN_NAME).c

# GCC plugin headers path
GCC_PLUGIN_DIR := $(shell gcc -print-file-name=plugin)/include

CFLAGS = -fPIC -O2 -Wall -I$(GCC_PLUGIN_DIR)
LDFLAGS = -shared

# Detect GCC version
GCC_VERSION := $(shell gcc -dumpversion)
GCC_MAJOR := $(word 1, $(subst ., ,$(GCC_VERSION)))

# Version-specific flags
ifeq ($(shell expr $(GCC_MAJOR) \>= 10), 1)
    CFLAGS += -DGCC_10_OR_LATER
endif

all: $(PLUGIN_SO)

$(PLUGIN_SO): $(PLUGIN_SRC)
	$(CC) $(CFLAGS) $(LDFLAGS) -o $@ $<

test: $(PLUGIN_SO)
	gcc -fplugin=./$(PLUGIN_SO) -c test_program.cpp -o test_program.o

clean:
	rm -f $(PLUGIN_SO) *.o

.PHONY: all test clean
```

## 7. **测试程序**

```cpp
/* test_program.cpp */
#include <iostream>

class TestClass {
public:
    TestClass() { std::cout << "TestClass constructed" << std::endl; }
    ~TestClass() { std::cout << "TestClass destructed" << std::endl; }
    
    void method() { std::cout << "Method called" << std::endl; }
};

class Derived : public TestClass {
public:
    Derived() { std::cout << "Derived constructed" << std::endl; }
    ~Derived() { std::cout << "Derived destructed" << std::endl; }
};

int main() {
    std::cout << "Creating objects..." << std::endl;
    
    TestClass obj1;
    Derived obj2;
    TestClass *ptr = new TestClass();
    
    std::cout << "Deleting dynamic object..." << std::endl;
    delete ptr;
    
    std::cout << "Exiting scope..." << std::endl;
    return 0;
}
```

## 8. **Python 包装器脚本**

```python
#!/usr/bin/env python3
"""
GCC Destructor Rewriter - Python wrapper
"""
import subprocess
import os
import sys
import argparse

class GCCDestructorRewriter:
    def __init__(self, plugin_path):
        self.plugin_path = os.path.abspath(plugin_path)
        
    def compile_with_plugin(self, source_files, output=None, flags=None):
        """使用插件编译源代码"""
        if flags is None:
            flags = []
        
        # 构建GCC命令
        cmd = ['gcc', '-fplugin=' + self.plugin_path]
        cmd.extend(flags)
        cmd.extend(source_files)
        
        if output:
            cmd.extend(['-o', output])
        
        print(f"运行: {' '.join(cmd)}")
        
        # 执行编译
        result = subprocess.run(cmd, capture_output=True, text=True)
        
        if result.returncode != 0:
            print(f"编译失败:\n{result.stderr}")
            return False
        
        print("编译成功")
        if result.stdout:
            print(f"输出:\n{result.stdout}")
        
        return True
    
    def generate_report(self, source_file):
        """生成析构函数重写报告"""
        # 编译为GIMPLE中间表示
        gimple_file = source_file + '.gimple'
        
        cmd = ['gcc', '-fplugin=' + self.plugin_path,
               '-fdump-tree-gimple', '-c', source_file]
        
        subprocess.run(cmd, capture_output=True)
        
        # 解析GIMPLE输出
        if os.path.exists(gimple_file):
            with open(gimple_file, 'r') as f:
                content = f.read()
            
            # 提取析构函数信息
            lines = content.split('\n')
            destructors = []
            
            for line in lines:
                if '~' in line or 'D1Ev' in line or 'D2Ev' in line:
                    destructors.append(line.strip())
            
            return destructors
        
        return []

def main():
    parser = argparse.ArgumentParser(
        description='GCC Destructor Rewriter Tool'
    )
    parser.add_argument('--plugin', default='destructor_rewrite.so',
                       help='插件路径')
    parser.add_argument('--compile', nargs='+', 
                       help='要编译的源文件')
    parser.add_argument('--output', help='输出文件')
    parser.add_argument('--report', help='生成报告的文件')
    parser.add_argument('--flags', nargs='+', default=['-O2'],
                       help='GCC编译标志')
    
    args = parser.parse_args()
    
    rewriter = GCCDestructorRewriter(args.plugin)
    
    if args.compile:
        rewriter.compile_with_plugin(
            args.compile, 
            output=args.output,
            flags=args.flags
        )
    
    if args.report:
        destructors = rewriter.generate_report(args.report)
        print(f"\n在 {args.report} 中找到的析构函数:")
        for dtor in destructors:
            print(f"  - {dtor}")

if __name__ == '__main__':
    main()
```

## 使用步骤总结：

1. **编写插件**：选择 GCC 插件或 LLVM Pass 方案
2. **编译插件**：
   ```bash
   # GCC插件
   gcc -I`gcc -print-file-name=plugin`/include -fPIC -shared -o plugin.so plugin.c
   
   # LLVM Pass
   clang++ -shared -fPIC `llvm-config --cxxflags --ldflags --system-libs --libs core` -o pass.so pass.cpp
   ```

3. **使用插件编译**：
   ```bash
   # GCC
   gcc -fplugin=./plugin.so -c your_program.cpp
   
   # LLVM
   clang -Xclang -load -Xclang ./pass.so your_program.cpp
   ```

4. **运行时拦截**（可选）：
   ```bash
   LD_PRELOAD=./libdestructor_hook.so ./your_program
   ```

## 注意事项：

1. **GCC版本兼容性**：不同版本的GCC插件API可能不同
2. **调试难度**：编译器插件调试困难，建议使用大量日志
3. **性能影响**：插件会增加编译时间
4. **安全性**：生产环境谨慎使用，可能引入不稳定因素

这种技术通常用于：
- 代码插桩（instrumentation）
- 安全增强（如内存安全检查）
- 性能分析
- 代码转换和优化研究