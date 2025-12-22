一、AVX2 世界观：256 位寄存器与 Lanes
欢迎来到 AVX2（Advanced Vector Extensions 2）的并行世界。简单来说，AVX2 允许 CPU 在一条指令里同时操作多个数据，就像用一辆大卡车一次运送一堆包裹，而不是一次只运一个。这种技术被称为 SIMD（Single Instruction, Multiple Data）。
理解 AVX2 的核心在于理解它的“卡车”——256 位 YMM 寄存器。你可以把一个 YMM 寄存器想象成一个能装 32 个字节（256 位）的大盒子。为了管理方便，这个大盒子在逻辑上被分成两个 128 位的小隔间，我们称之为 lane（车道）。
- YMM 寄存器：一个 256 位的容器。
- Lane：YMM 寄存器内部的两个 128 位分区（Lane 0 和 Lane 1）。
- 元素（Element）：寄存器中存放的数据单元，可以是 8 位整数（字节）、16 位、32 位或 64 位整数。
暂时无法在飞书文档外展示此内容
大多数 AVX2 指令都是在各自的 lane 内部独立执行的，仿佛两条并行流水线。但也有些特殊指令允许数据跨 lane 移动，这是实现复杂算法的关键。

---
二、装卸货物：载入与存储
一切操作都始于将数据从内存搬进寄存器（载入，Load），并在计算后将其搬回内存（存储，Store）。
1. 基础载入/存储 (_mm256_loadu_si256 / _mm256_storeu_si256)
这是最常用的载入/存储方式。loadu 中的 u 代表 unaligned（未对齐），意味着它可以从内存的任何地址开始读取 32 个字节。
暂时无法在飞书文档外展示此内容
// 示例：将内存中的数据载入、加一、然后存回
#include <immintrin.h>
#include <iostream>
#include <array>

void print_arr(const std::array<char, 32>& arr) {
    for (int i = 0; i < 32; ++i) {
        std::cout << (int)arr[i] << " ";
    }
    std::cout << std::endl;
}

int main() {
    // 准备一块内存数据 (32 字节)
    alignas(32) std::array<char, 32> my_data;
    for(int i = 0; i < 32; ++i) my_data[i] = i;

    // 1. 从内存载入到 YMM 寄存器
    // 对应图解中的“内存 -> 寄存器”
    const __m256i data_vec = _mm256_loadu_si256(reinterpret_cast<const __m256i*>(my_data.data()));

    // 2. 对寄存器中的每个字节加 1
    const __m256i ones = _mm256_set1_epi8(1);
    const __m256i result_vec = _mm256_add_epi8(data_vec, ones);

    // 3. 将结果从寄存器存回内存
    // 对应图解中的“寄存器 -> 内存”
    alignas(32) std::array<char, 32> output_data;
    _mm256_storeu_si256(reinterpret_cast<__m256i*>(output_data.data()), result_vec);

    std::cout << "Original data: ";
    print_arr(my_data);
    std::cout << "Output data:   ";
    print_arr(output_data);

    return 0;
}

对齐（Aligned） vs. 未对齐（Unaligned）
- _mm256_load_si256 (对齐版本): 要求内存地址必须是 32 字节的倍数。如果不是，程序会崩溃。它的性能通常比未对齐版本稍好，因为 CPU 读取起来更高效。
- _mm256_loadu_si256 (未对齐版本): 对地址没有要求，更灵活，但性能可能略有损失（尤其是在旧架构上）。
现代 CPU 对未对齐访问的惩罚已经很小，因此除非你对性能有极致要求并能确保数据对齐，否则使用 loadu 通常是更安全的选择。

---
三、复制与填充：广播
广播（Broadcast） 是一个非常高频的操作：将一个标量值（单个数据）复制到寄存器的所有元素中。这在需要用一个常数与整个向量进行计算时特别有用。
- _mm256_set1_epiX：将一个 X 位的整数广播到所有对应的元素位置。
暂时无法在飞书文档外展示此内容
#include <immintrin.h>
#include <iostream>

int main() {
    // 将整数 5 广播到 8 个 32-bit 元素中
    int scalar_value = 5;
    __m256i broadcast_vec = _mm256_set1_epi32(scalar_value); // 传入一个 32-bit 整数

    // 打印结果 (为简化，只打印第一个和最后一个元素)
    int* ptr = (int*)&broadcast_vec;
    std::cout << "Element 0: " << ptr[0] << std::endl; // 输出 5
    std::cout << "Element 7: " << ptr[7] << std::endl; // 输出 5

    // 将字符 'A' (ASCII 65) 广播到 32 个 8-bit 元素中
    char char_value = 'A';
    __m256i broadcast_chars = _mm256_set1_epi8(char_value);

    char* cptr = (char*)&broadcast_chars;
    std::cout << "Char 0: " << cptr[0] << std::endl;   // 输出 'A'
    std::cout << "Char 31: " << cptr[31] << std::endl; // 输出 'A'

    return 0;
}


---
四、并行计算：算术与逻辑
SIMD 的核心价值在于并行计算。AVX2 提供了丰富的指令，可以对寄存器中的每个元素同时执行加、减、乘、位移以及与、或、异或等逻辑操作。
- 原则：所有算术和逻辑操作都是 按元素（element-wise） 进行的，并且在各自的 lane 内独立发生，互不干扰。
暂时无法在飞书文档外展示此内容
#include <immintrin.h>
#include <iostream>

void print_epi32(__m256i vec) {
    int* ptr = (int*)&vec;
    for (int i = 0; i < 8; ++i) {
        std::cout << ptr[i] << " ";
    }
    std::cout << std::endl;
}

int main() {
    // 两个源向量
    __m256i vec_a = _mm256_setr_epi32(1, 2, 3, 4, 5, 6, 7, 8);
    __m256i vec_b = _mm256_set1_epi32(10); // {10, 10, 10, 10, 10, 10, 10, 10}

    // 1. 加法: _mm256_add_epi32
    __m256i sum = _mm256_add_epi32(vec_a, vec_b);
    std::cout << "Sum: "; print_epi32(sum); // 输出: 11 12 13 14 15 16 17 18

    // 2. 减法: _mm256_sub_epi32
    __m256i diff = _mm256_sub_epi32(vec_a, vec_b);
    std::cout << "Diff: "; print_epi32(diff); // 输出: -9 -8 -7 -6 -5 -4 -3 -2

    // 3. 位与: _mm256_and_si256
    __m256i a = _mm256_setr_epi32(0b1100, 2, 3, 4, 5, 6, 7, 8);
    __m256i b = _mm256_setr_epi32(0b1010, 10, 10, 10, 10, 10, 10, 10);
    __m256i and_res = _mm256_and_si256(a, b);
    std::cout << "AND: "; print_epi32(and_res); // 输出: 8 2 2 0 0 2 2 8 (二进制 1000 10 10 ...)

    // 4. 左移: _mm256_slli_epi32 (立即数版本)
    // 将 vec_a 的每个 32-bit 元素左移 1 位
    __m256i shifted = _mm256_slli_epi32(vec_a, 1);
    std::cout << "Shifted: "; print_epi32(shifted); // 输出: 2 4 6 8 10 12 14 16
    
    return 0;
}

关于整数乘法
AVX2 对整数乘法的支持有点“特别”。它没有提供一条指令就算完 32 位整数乘法并得到 32 位结果。
- _mm256_mullo_epi16 / _mm256_mullo_epi32: epi16 版本支持 16 位整数乘法，取低 16 位结果。epi32 版本则将每对 32 位整数相乘，得到 64 位结果，但只保留低 32 位。这在需要模拟 32x32->32 乘法时很有用（但需要两条指令）。
- _mm256_mul_epi32: 将两个源向量中 32 位的元素（仅偶数位）相乘，得到 64 位结果。
- 完整的 32 位整数乘法通常需要多条指令组合，或者依赖 AVX-512 等更新的指令集。

---
五、条件判断：比较与掩码
要在 SIMD 中实现 if-else 逻辑，我们不能简单地让某些元素计算而另一些不计算。取而代之的是“全量计算、按需挑选”的策略。这个过程分为两步：比较（Compare） 和 按掩码选择（Blend）。
1. 比较（Compare）
比较指令会按元素比较两个向量，并生成一个掩码（Mask）向量。在掩码中，如果对应元素满足条件，则所有位都为 1（即 0xFF...）；否则所有位都为 0。
- _mm256_cmpeq_epiX: 比较 X 位整数是否相等。
- _mm256_cmpgt_epiX: 比较 X 位整数是否大于（前者 > 后者）。
暂时无法在飞书文档外展示此内容
#include <immintrin.h>
#include <iostream>

void print_hex_epi32(__m256i vec) {
    int* ptr = (int*)&vec;
    std::cout << std::hex;
    for (int i = 0; i < 8; ++i) {
        std::cout << "0x" << ptr[i] << " ";
    }
    std::cout << std::dec << std::endl;
}

int main() {
    __m256i vec_a = _mm256_setr_epi32(10, 20, 30, 5,   15, 25, 35, 8);
    __m256i vec_b = _mm256_setr_epi32(10, 10, 40, 5,   15, 30, 30, 10);

    // 比较 vec_a 和 vec_b 中相等的 32-bit 元素
    __m256i mask_eq = _mm256_cmpeq_epi32(vec_a, vec_b);
    std::cout << "Mask (Equal):     "; print_hex_epi32(mask_eq);
    // 输出: 0xffffffff 0x0 0x0 0xffffffff   0xffffffff 0x0 0x0 0x0

    // 比较 vec_a 中大于 vec_b 的 32-bit 元素
    __m256i mask_gt = _mm256_cmpgt_epi32(vec_a, vec_b);
    std::cout << "Mask (Greater Than): "; print_hex_epi32(mask_gt);
    // 输出: 0x0 0xffffffff 0x0 0x0   0x0 0x0 0xffffffff 0x0
}

2. 按掩码选择（Blend）
得到掩码后，blend 指令就派上用场了。它会根据掩码的最高位（sign bit）来决定从两个源向量中挑选哪个元素，组合成最终的结果。
- _mm256_blendv_epi8: v 代表 variable，表示使用一个变量（即掩码向量）来控制选择。如果掩码的对应字节最高位是 1，就从第二个源向量取值；如果是 0，就从第一个源向量取值。
暂时无法在飞书文档外展示此内容
#include <immintrin.h>
#include <iostream>

// (print_hex_epi32 函数同上)

int main() {
    // 假设我们要实现: if (a > b) then a else b  (即取较大值)
    __m256i vec_a = _mm256_setr_epi32(10, 20, 30, 5,   15, 25, 35, 8);
    __m256i vec_b = _mm256_setr_epi32(10, 10, 40, 5,   15, 30, 30, 10);

    // 1. 生成掩码: a > b ?
    __m256i mask_a_gt_b = _mm256_cmpgt_epi32(vec_a, vec_b);
    // mask_a_gt_b: {0, 0xFFFFFFFF, 0, 0,   0, 0, 0xFFFFFFFF, 0}

    // 2. 使用 blendv 按掩码选择
    // 如果 mask 位为 1 (a > b)，选择 vec_a；否则选择 vec_b。
    // blendv 的行为是：mask位为1，选第二个参数；为0，选第一个参数。
    // 所以，为了实现 if(a > b) a else b，我们需要把 a 作为第二个参数传入
    __m256i max_vals = _mm256_blendv_epi8(vec_b, vec_a, mask_a_gt_b);

    int* ptr = (int*)&max_vals;
    std::cout << "Max values: ";
    for (int i = 0; i < 8; ++i) {
        std::cout << ptr[i] << " ";
    }
    std::cout << std::endl; // 输出: 10 20 40 5   15 30 35 10
}


---
六、数据重排：Shuffle 与 Permute
这是 SIMD 编程中最灵活也最复杂的部分。Shuffle 和 Permute 指令允许你在寄存器内部、甚至跨 lane 移动数据。
1. Lane 内重排 (In-Lane Shuffle)
_mm256_shuffle_epi8 (pshufb)
这是最强大的 lane 内字节级重排指令。它使用一个控制向量（shuffle control mask）来决定如何在一个 128 位 lane 中重新组织字节。
- 控制掩码规则：
  - 控制向量的每个字节都是一个索引。
  - 如果索引 i 的最高位是 1（即 i >= 128），则目标位置的字节将被清零。
  - 如果最高位是 0，则目标位置的字节将被替换为源向量中索引为 i & 0x0F (即 i % 16) 的字节。
- 关键：pshufb 在两个 128 位 lane 内独立并行地执行相同的重排操作。
暂时无法在飞书文档外展示此内容
#include <immintrin.h>
#include <iostream>

void print_epi8(__m256i vec) {
    char* p = (char*)&vec;
    for(int i = 0; i < 32; ++i) std::cout << (int)p[i] << " ";
    std::cout << std::endl;
}

int main() {
    // 源数据，在两个 lane 中都是 0 到 15
    __m256i data = _mm256_setr_epi8(
        0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15,
        0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15
    );

    // 控制掩码：反转每个 lane 中的字节顺序
    // 比如第一个字节取源的第15个，第二个字节取第14个...
    // 0x80 是最高位为1的字节，用于清零
    __m256i shuffle_mask = _mm256_setr_epi8(
        15, 14, 13, 12, 11, 10, 9, 8, 7, 6, 5, 4, 3, 2, 1, 0,
        15, 14, 13, 12, 11, 10, 9, 8, 7, 6, 5, 4, 3, 2, 0x80, 0 // 最后一个元素清零
    );

    __m256i result = _mm256_shuffle_epi8(data, shuffle_mask);
    std::cout << "Shuffled: " << std::endl;
    print_epi8(result);
    // 输出:
    // 15 14 13 12 11 10 9 8 7 6 5 4 3 2 1 0 15 14 13 12 11 10 9 8 7 6 5 4 3 2 0 0
}

_mm256_shuffle_epi32 (pshufd)
这是一个更简单的 lane 内 32 位元素重排指令。它使用一个 8 位的立即数（imm8）作为控制掩码，在每个 lane 内独立地重排 4 个 32 位元素。
- 控制掩码规则：imm8 的每 2 位控制一个目标位置。例如，_mm256_shuffle_epi32(vec, 0b11100100) 表示：
  - 目标位置 0 取源位置 00
  - 目标位置 1 取源位置 01
  - 目标位置 2 取源位置 10
  - 目标位置 3 取源位置 11
  - 这个 0b11100100 模式会同时应用于两个 lane。
暂时无法在飞书文档外展示此内容
2. 跨 Lane 重排 (Cross-Lane Permute)
shuffle 指令通常受限于 lane 边界，而 permute 指令则专为跨 lane 数据移动而设计。
_mm256_permutevar8x32_epi32 (vpermd)
这是 32 位元素级别的“自由”重排。它使用一个索引向量，可以从源向量的全部 8 个 32 位元素中任意挑选，放入目标位置。
- 控制掩码规则：索引向量中的每个 32 位整数 i（取值范围 0-7）指定了目标向量的对应位置应该从源向量的哪个位置 i 获取数据。
暂时无法在飞书文档外展示此内容
#include <immintrin.h>
#include <iostream>

// (print_epi32 函数同上)

int main() {
    __m256i data = _mm256_setr_epi32(10, 20, 30, 40, 50, 60, 70, 80);

    // 索引：将高低 lane 数据对调
    // 目标[0] = 源[4], 目标[1] = 源[5]...
    __m256i indices = _mm256_setr_epi32(4, 5, 6, 7, 0, 1, 2, 3);

    __m256i result = _mm256_permutevar8x32_epi32(data, indices);

    std::cout << "Permuted: "; print_epi32(result);
    // 输出: 50 60 70 80 10 20 30 40
}

vpermd vs vpermq
- _mm256_permutevar8x32_epi32 (vpermd): 按 32位 元素进行重排。
- _mm256_permute4x64_epi64 (vpermq): 按 64位 元素进行重排，功能类似，但操作粒度更大。
_mm256_permute2x128_si256 (vperm2i128)
这是一个重量级指令，它在 128 位 lane 级别进行重排。它可以从两个源向量中各取出两个 128 位 lane，然后按一个立即数掩码的指示，将它们组合成一个新的 256 位向量。
- 控制掩码规则：imm8 的高 4 位和低 4 位分别控制目标向量的高 lane 和低 lane。
  - 每 4 位中的前 2 位选择源（00=a, 01=b, 10=a, 11=b），后 2 位选择 lane（0=low, 1=high）。
  - 0b1... 表示将对应 lane 清零。
暂时无法在飞书文档外展示此内容

---
七、数据拼接与提取
1. 字节级对齐拼接 (_mm256_alignr_epi8)
alignr 指令非常适合处理流式数据。它将两个向量看作一个连续的字节序列，然后从中提取出一个 16 字节（lane 内）或 32 字节（跨 lane）的结果。操作在两个 lane 内独立进行。
- 控制掩码规则：imm8 立即数指定了从拼接后的 a:b 序列的哪个字节位置开始拷贝。
暂时无法在飞书文档外展示此内容
2. 提取/插入128位 Lane
有时你需要单独操作一个 lane。AVX2 提供了直接的指令来提取 YMM 寄存器的高低 128 位，或将一个 128 位向量插入 YMM 寄存器的特定 lane。
- _mm256_extracti128_si256(vec, 1): 提取高 128 位 lane。
- _mm256_inserti128_si256(vec, sub_vec, 1): 将 sub_vec 插入 vec 的高 128 位 lane。

---
八、内存采集：Gather
Gather 是 AVX2 中一个强大的特性。与 load 一次性读取连续内存不同，gather 可以根据一个索引向量，从内存的离散位置 "采集" 数据，然后将它们紧凑地排列到目标寄存器中。
- 工作方式：它使用一个基地址（base_addr）和一个包含多个偏移量（offsets）的索引向量。最终访问的地址是 base_addr + offsets[i]。
- 掩码：gather 还接受一个掩码向量。如果掩码中某个元素为 0，则对应的内存读取操作将被跳过，目标寄存器中的该位置将保持不变（或清零，取决于具体指令）。
暂时无法在飞书文档外展示此内容
#include <immintrin.h>
#include <iostream>

// (print_epi32 函数同上)

int main() {
    // 内存中有一组离散数据
    alignas(32) int source_data[] = {100, 101, 102, 103, 104, 105, 106, 107, 108, 109};

    // 索引向量，指定要从 source_data 中采集哪些元素的索引
    // 注意：这里是索引，不是字节偏移！
    __m256i indices = _mm256_setr_epi32(8, 0, 3, 1, 9, 2, 5, 4);

    // 掩码，决定哪些采集操作有效 (最高位为1则有效)
    __m256i mask = _mm256_setr_epi32(
        -1, -1, 0, -1,  // 第3个无效
        -1, 0, -1, -1   // 第6个无效
    );
    
    // 执行 Gather 操作
    // _mm256_i32gather_epi32(基地址, 索引向量, 缩放因子)
    // 缩放因子 sizeof(int) = 4
    __m256i result = _mm256_mask_i32gather_epi32(
        _mm256_setzero_si256(), // 目标寄存器初始值
        source_data,            // 基地址
        indices,                // 偏移索引
        mask,                   // 掩码
        4                       // scale (每个索引乘以4字节)
    );

    std::cout << "Gathered data: "; print_epi32(result);
    // 输出: 108 100 0 101 109 0 105 104 (无效的位置为0)
}


---
九、实践提示与常见陷阱
性能提示
- 对齐：尽可能使用对齐的内存载入/存储。虽然现代 CPU 对未对齐访问的惩罚减小，但在高吞吐场景下，对齐依然能带来优势。
- 指令延迟与吞吐：不同的指令执行速度不同。例如，add/and 通常很快（延迟低，吞吐高），而 mul 和 gather 则慢得多。写代码时心中有数，避免在关键循环中过度使用慢指令。
- 数据依赖：避免长链条的数据依赖（下一步的计算必须等上一步完成）。SIMD 的威力在于并行，尽量组织代码，让多个操作可以同时进行。
常见陷阱
- Lane 边界：时刻牢记 128 位的 lane 边界！shuffle 指令通常无法跨越它，需要 permute 指令帮忙。混用它们是初学者最常见的错误之一。
- 立即数 vs. 向量掩码：注意区分需要立即数（imm8）作控制掩码的指令（如 pshufd）和需要向量作掩码的指令（如 pshufb）。前者更紧凑，但灵活性低。
- 有符号 vs. 无符号：比较和乘法等操作对有符号和无符号整数的处理方式不同，确保使用正确的指令版本。
- blend 的逻辑：_mm256_blendv_epi8 的掩码判断的是最高位，而不是整个字节是否为0。这在处理非 cmpeq/cmpgt 生成的自定义掩码时需要特别注意。