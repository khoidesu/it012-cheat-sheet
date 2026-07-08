# Ánh xạ C++ → MIPS Assembly

Tài liệu này trình bày cách các cấu trúc lệnh phổ biến trong C++ được dịch (biên dịch) sang hợp ngữ MIPS. Quy ước thanh ghi dùng trong tài liệu:

| Thanh ghi | Vai trò |
|---|---|
| `$s0 - $s7` | Biến "saved" (thường dùng cho biến địa phương, con trỏ mảng) |
| `$t0 - $t9` | Biến tạm (temporary) |
| `$a0 - $a3` | Tham số truyền vào hàm |
| `$v0 - $v1` | Giá trị trả về của hàm |
| `$ra` | Địa chỉ trả về (return address) |
| `$sp` | Con trỏ ngăn xếp (stack pointer) |
| `$zero` | Luôn bằng 0 |

---


## 0. Nhập / Xuất (Input / Output)

### 0.1 Nhập – xuất **một số nguyên**

C++:
```cpp
int a;
cin >> a;
cout << a;
```

MIPS:
```asm
.data
    msg_nhap: .asciiz "Nhap mot so nguyen: "

.text
main:
    # In thông báo nhập
    li   $v0, 4
    la   $a0, msg_nhap
    syscall

    # Nhập số nguyên -> lưu vào $s0
    li   $v0, 5
    syscall
    move $s0, $v0

    # Xuất lại số nguyên vừa nhập
    li   $v0, 1
    move $a0, $s0
    syscall

    li   $v0, 10
    syscall
```

### 0.2 Nhập – xuất **mảng số nguyên**

C++:
```cpp
int a[5];
for (int i = 0; i < 5; i++) cin >> a[i];
for (int i = 0; i < 5; i++) cout << a[i] << " ";
```

MIPS:
```asm
.data
    a:        .space 20          # mảng 5 phần tử int (5*4 = 20 byte)
    n:        .word  5
    space:    .asciiz " "

.text
main:
    la   $s0, a          # s0 = địa chỉ gốc mảng a
    lw   $s1, n          # s1 = n (số phần tử)
    li   $s2, 0          # s2 = i = 0

# ---- Vòng lặp NHẬP mảng ----
NHAP_LOOP:
    bge  $s2, $s1, NHAP_END

    li   $v0, 5           # syscall đọc số nguyên
    syscall                # v0 = giá trị vừa nhập

    sll  $t0, $s2, 2       # t0 = i * 4
    add  $t1, $s0, $t0     # t1 = &a[i]
    sw   $v0, 0($t1)       # a[i] = v0

    addi $s2, $s2, 1       # i++
    j    NHAP_LOOP
NHAP_END:

    li   $s2, 0            # reset i = 0

# ---- Vòng lặp XUẤT mảng ----
XUAT_LOOP:
    bge  $s2, $s1, XUAT_END

    sll  $t0, $s2, 2
    add  $t1, $s0, $t0
    lw   $t2, 0($t1)       # t2 = a[i]

    li   $v0, 1            # syscall in số nguyên
    move $a0, $t2
    syscall

    li   $v0, 4            # in dấu cách sau mỗi số
    la   $a0, space
    syscall

    addi $s2, $s2, 1
    j    XUAT_LOOP
XUAT_END:

    li   $v0, 10
    syscall
```

> ```asm
> slt  $t3, $s2, $s1
> beq  $t3, $zero, NHAP_END
> ```

---

## 1. Khai báo & gán biến đơn giản

| C++ | MIPS |
|---|---|
| `int a = 5;` | `li $t0, 5` |
| `int b;` (chưa gán) | *(không cần lệnh, chỉ cấp thanh ghi/ô nhớ)* |
| `a = b;` | `move $t0, $t1` |
| `a = 0;` | `li $t0, 0` hoặc `move $t0, $zero` |

---

## 2. Phép toán số học

| C++ | MIPS |
|---|---|
| `c = a + b;` | `add $t2, $t0, $t1` |
| `c = a - b;` | `sub $t2, $t0, $t1` |
| `c = a * b;` | `mul $t2, $t0, $t1` |
| `c = a / b;` | `div $t0, $t1`<br>`mflo $t2`  *(thương số)*<br>`mfhi $t3`  *(số dư, nếu cần)* |
| `c = a % b;` | `div $t0, $t1`<br>`mfhi $t2` |
| `a++;` | `addi $t0, $t0, 1` |
| `a--;` | `addi $t0, $t0, -1` |
| `a += 5;` | `addi $t0, $t0, 5` |
| `a = -a;` | `sub $t0, $zero, $t0` hoặc `neg $t0, $t0` |
| `c = a & b;` | `and $t2, $t0, $t1` |
| `c = a \| b;` | `or $t2, $t0, $t1` |
| `c = a ^ b;` | `xor $t2, $t0, $t1` |
| `c = ~a;` | `not $t2, $t0` |
| `c = a << 2;` | `sll $t2, $t0, 2` |
| `c = a >> 2;` | `sra $t2, $t0, 2`  *(dịch có dấu)* |

---

## 3. Ví dụ trọng tâm: `sum = sum + a[i];`

C++:
```cpp
sum = sum + a[i];
```

MIPS:
```asm
# $s0 = địa chỉ gốc mảng a
# $s1 = i
# $s2 = sum

sll  $t0, $s1, 2        # t0 = i * 4  (mỗi int chiếm 4 byte)
add  $t1, $s0, $t0      # t1 = địa chỉ &a[i] = a + i*4
lw   $t2, 0($t1)        # t2 = a[i]
add  $s2, $s2, $t2      # sum = sum + a[i]
```

**Giải thích từng bước:**
1. `sll $t0, $s1, 2` — dịch trái `i` đi 2 bit tương đương nhân với 4 (kích thước `int`), dùng để tính offset trong mảng.
2. `add $t1, $s0, $t0` — cộng offset vào địa chỉ gốc mảng để ra địa chỉ của `a[i]`.
3. `lw $t2, 0($t1)` — nạp (load word) giá trị `a[i]` từ bộ nhớ vào thanh ghi.
4. `add $s2, $s2, $t2` — cộng dồn vào `sum`.

---

## 4. Truy cập & gán mảng

| C++ | MIPS |
|---|---|
| `x = a[i];` | `sll $t0, $s1, 2`<br>`add $t1, $s0, $t0`<br>`lw  $t2, 0($t1)`<br>`move $t3, $t2` *(x)* |
| `a[i] = x;` | `sll $t0, $s1, 2`<br>`add $t1, $s0, $t0`<br>`sw  $t3, 0($t1)` |
| `a[0] = 10;` | `li $t0, 10`<br>`sw $t0, 0($s0)` |
| `a[i] = a[i] + 1;` | `sll $t0, $s1, 2`<br>`add $t1, $s0, $t0`<br>`lw  $t2, 0($t1)`<br>`addi $t2, $t2, 1`<br>`sw  $t2, 0($t1)` |
| Mảng `char b[i]` | Dùng `lb`/`sb` (load/store **byte**) thay vì `lw`/`sw`, và offset nhân **1** thay vì 4 |

---

## 5. Câu lệnh điều kiện `if / else`

C++:
```cpp
if (a == b) {
    c = 1;
} else {
    c = 2;
}
```

MIPS:
```asm
bne  $t0, $t1, ELSE    # nếu a != b thì nhảy tới ELSE
li   $t2, 1            # c = 1
j    END_IF
ELSE:
li   $t2, 2            # c = 2
END_IF:
```

### Bảng ánh xạ toán tử so sánh (điều kiện *đảo ngược* để nhảy khi sai)

| C++ | MIPS (nhảy khi điều kiện SAI, tới nhãn ELSE) |
|---|---|
| `if (a == b)` | `bne $t0, $t1, ELSE` |
| `if (a != b)` | `beq $t0, $t1, ELSE` |
| `if (a < b)` | `bge $t0, $t1, ELSE`  *(hoặc `slt` + `beq`)* |
| `if (a <= b)` | `bgt $t0, $t1, ELSE` |
| `if (a > b)` | `ble $t0, $t1, ELSE` |
| `if (a >= b)` | `blt $t0, $t1, ELSE` |

> Với bộ lệnh MIPS "chuẩn" (không có pseudo-instruction), `a < b` thường viết là:
> ```asm
> slt $t2, $t0, $t1   # t2 = 1 nếu a < b, ngược lại t2 = 0
> beq $t2, $zero, ELSE
> ```

---

## 6. Vòng lặp `for`

C++:
```cpp
for (i = 0; i < 10; i++) {
    sum = sum + a[i];
}
```

MIPS:
```asm
li   $s1, 0             # i = 0
FOR_LOOP:
slti $t0, $s1, 10        # t0 = 1 nếu i < 10
beq  $t0, $zero, END_FOR # nếu không (i >= 10) thoát vòng lặp

sll  $t1, $s1, 2
add  $t2, $s0, $t1
lw   $t3, 0($t2)
add  $s2, $s2, $t3       # sum += a[i]

addi $s1, $s1, 1         # i++
j    FOR_LOOP
END_FOR:
```

---

## 7. Vòng lặp `while`

C++:
```cpp
while (a > 0) {
    a = a - 1;
}
```

MIPS:
```asm
WHILE:
slti $t1, $t0, 1         # t1 = 1 nếu a < 1 (tức a <= 0)
bne  $t1, $zero, END_WHILE
addi $t0, $t0, -1        # a = a - 1
j    WHILE
END_WHILE:
```

## 7b. Vòng lặp `do...while`

C++:
```cpp
do {
    a = a - 1;
} while (a > 0);
```

MIPS:
```asm
DO_WHILE:
addi $t0, $t0, -1        # a = a - 1
slti $t1, $zero, $t0     # t1 = 1 nếu a > 0
bne  $t1, $zero, DO_WHILE
```

---

## 8. Gọi hàm (function call)

C++:
```cpp
int add(int x, int y) {
    return x + y;
}

int result = add(a, b);
```

MIPS:
```asm
# --- Nơi gọi hàm (caller) ---
move $a0, $t0        # x = a
move $a1, $t1        # y = b
jal  add              # gọi hàm, lưu $ra tự động
move $s2, $v0        # result = giá trị trả về

# --- Định nghĩa hàm add (callee) ---
add:
    add  $v0, $a0, $a1   # v0 = x + y
    jr   $ra              # trả về nơi gọi
```

### Hàm đệ quy có dùng ngăn xếp (ví dụ: `factorial`)
```cpp
int factorial(int n) {
    if (n <= 1) return 1;
    return n * factorial(n - 1);
}
```
```asm
factorial:
    addi $sp, $sp, -8      # cấp phát 8 byte trên stack
    sw   $ra, 4($sp)       # lưu địa chỉ trả về
    sw   $a0, 0($sp)       # lưu n

    slti $t0, $a0, 2
    beq  $t0, $zero, RECURSE
    li   $v0, 1
    addi $sp, $sp, 8
    jr   $ra

RECURSE:
    addi $a0, $a0, -1
    jal  factorial          # gọi factorial(n-1)

    lw   $a0, 0($sp)        # khôi phục n
    lw   $ra, 4($sp)        # khôi phục địa chỉ trả về
    addi $sp, $sp, 8

    mul  $v0, $a0, $v0      # v0 = n * factorial(n-1)
    jr   $ra
```

---

## 9. Con trỏ (pointer)

| C++ | MIPS |
|---|---|
| `int *p = &a;` | `la $t0, a`  *(nếu `a` là biến toàn cục)* hoặc `move $t0, $s0` |
| `x = *p;` | `lw $t1, 0($t0)` |
| `*p = 5;` | `li $t1, 5`<br>`sw $t1, 0($t0)` |
| `p++;` (con trỏ int) | `addi $t0, $t0, 4` |
| `p = p + i;` | `sll $t2, $s1, 2`<br>`add $t0, $t0, $t2` |

---

## 10. Cấu trúc một chương trình MIPS hoàn chỉnh

Một file hợp ngữ MIPS (`.asm` / `.s`) thường gồm **2 phần (segment)** chính:

```asm
.data
    # Khai báo biến, mảng, chuỗi, hằng số — tương tự vùng "biến toàn cục"
    # Được cấp phát sẵn trong bộ nhớ, có giá trị khởi tạo (nếu có)

.text
    # Phần chứa mã lệnh (code) thực thi
    # BẮT BUỘC phải có nhãn main: là điểm bắt đầu chương trình
```

### Ví dụ khung sườn tổng quát

```asm
.data
    sum:     .word   0                  # biến int, khởi tạo = 0
    arr:     .word   1, 2, 3, 4, 5       # mảng int có sẵn giá trị
    n:       .word   5                  # số phần tử mảng
    str1:    .asciiz "Nhap so: "        # chuỗi ký tự (thông báo)
    result:  .asciiz "Ket qua la: "

.text
main:
    # ... mã lệnh chính của chương trình ...

    li   $v0, 10        # syscall số 10 = kết thúc chương trình (exit)
    syscall
```

### Giải thích các thành phần

| Thành phần | Ý nghĩa |
|---|---|
| `.data` | Vùng khai báo dữ liệu tĩnh (biến, mảng, chuỗi) — giống biến toàn cục trong C++ |
| `.text` | Vùng chứa các lệnh máy (code), tương ứng với thân các hàm trong C++ |
| `main:` | Nhãn bắt buộc, là điểm vào (entry point) của chương trình, giống hàm `int main()` |
| `.word` | Khai báo số nguyên (int, 4 byte) |
| `.byte` | Khai báo số nguyên nhỏ / ký tự (1 byte) |
| `.asciiz` | Khai báo chuỗi ký tự có kết thúc bằng byte `\0` (giống `char[]` trong C++) |
| `.ascii` | Khai báo chuỗi **không** tự thêm `\0` |
| `.space n` | Cấp phát `n` byte bộ nhớ trống (dùng làm vùng đệm để nhập chuỗi) |
| `syscall` | Lệnh gọi hệ thống để thực hiện nhập/xuất, cấp phát bộ nhớ, kết thúc chương trình,... |

### Bảng các dịch vụ `syscall` thường dùng

| `$v0` | Chức năng | Tham số vào | Kết quả trả về |
|---|---|---|---|
| `1` | In số nguyên | `$a0` = số cần in | — |
| `4` | In chuỗi | `$a0` = địa chỉ chuỗi | — |
| `5` | Đọc số nguyên | — | `$v0` = số đã nhập |
| `8` | Đọc chuỗi | `$a0` = địa chỉ buffer, `$a1` = độ dài tối đa | chuỗi lưu vào buffer |
| `9` | Cấp phát bộ nhớ động (sbrk) | `$a0` = số byte cần cấp | `$v0` = địa chỉ vùng nhớ |
| `10` | Kết thúc chương trình | — | — |
| `11` | In 1 ký tự | `$a0` = mã ký tự | — |
| `12` | Đọc 1 ký tự | — | `$v0` = mã ký tự đọc được |

---


## 12. Xử lý xâu ký tự (String) trong MIPS

MIPS không có kiểu `string` như C++ (`std::string`); xâu được biểu diễn dưới dạng **mảng ký tự (char array) kết thúc bằng byte `\0`**, giống `char[]`/con trỏ `char*` trong C.

### 12.1 Khai báo xâu

| C++ | MIPS |
|---|---|
| `char s[] = "Hello";` (đã có giá trị) | `.data`<br>`s: .asciiz "Hello"` |
| `char s[20];` (chưa có giá trị, chỉ cấp chỗ) | `.data`<br>`s: .space 20` |
| `char c = 'A';` | `.data`<br>`c: .byte 'A'` |

### 12.2 Nhập xâu từ bàn phím

C++:
```cpp
char s[100];
cin.getline(s, 100);
```

MIPS:
```asm
.data
    s:      .space 100       # vùng đệm chứa xâu, tối đa 100 byte

.text
    li   $v0, 8         # syscall đọc chuỗi
    la   $a0, s          # địa chỉ buffer để lưu xâu
    li   $a1, 100        # độ dài tối đa được đọc
    syscall
    # Sau lệnh này, chuỗi đã nhập được lưu tại địa chỉ s, kết thúc bằng '\0' (và có thể có '\n')
```

### 12.3 Xuất xâu ra màn hình

C++:
```cpp
cout << s;
```

MIPS:
```asm
    li   $v0, 4          # syscall in chuỗi
    la   $a0, s           # địa chỉ chuỗi cần in
    syscall
```

### 12.4 Tính độ dài xâu (giống `strlen`)

C++:
```cpp
int len = strlen(s);
```

MIPS:
```asm
    la   $t0, s          # t0 = con trỏ duyệt xâu, bắt đầu từ đầu s
    li   $t1, 0           # t1 = độ dài (len) = 0

STRLEN_LOOP:
    lb   $t2, 0($t0)      # đọc 1 ký tự (byte) tại vị trí hiện tại
    beq  $t2, $zero, STRLEN_END   # nếu gặp '\0' thì dừng
    addi $t1, $t1, 1       # len++
    addi $t0, $t0, 1       # con trỏ trỏ tới ký tự kế tiếp
    j    STRLEN_LOOP
STRLEN_END:
    # $t1 chứa độ dài xâu
```

### 12.5 Duyệt / in từng ký tự của xâu

C++:
```cpp
for (int i = 0; s[i] != '\0'; i++) {
    cout << s[i];
}
```

MIPS:
```asm
    la   $t0, s

PRINT_CHAR_LOOP:
    lb   $t1, 0($t0)          # lấy ký tự hiện tại
    beq  $t1, $zero, PRINT_CHAR_END

    li   $v0, 11               # syscall in 1 ký tự
    move $a0, $t1
    syscall

    addi $t0, $t0, 1
    j    PRINT_CHAR_LOOP
PRINT_CHAR_END:
```

### 12.6 So sánh 2 xâu (giống `strcmp`)

C++:
```cpp
bool equal = (strcmp(s1, s2) == 0);
```

MIPS:
```asm
    la   $t0, s1
    la   $t1, s2

STRCMP_LOOP:
    lb   $t2, 0($t0)
    lb   $t3, 0($t1)
    bne  $t2, $t3, NOT_EQUAL       # khác ký tự -> khác nhau
    beq  $t2, $zero, EQUAL          # cùng gặp '\0' -> bằng nhau (kết thúc)
    addi $t0, $t0, 1
    addi $t1, $t1, 1
    j    STRCMP_LOOP

NOT_EQUAL:
    li   $s3, 0     # equal = false
    j    STRCMP_END
EQUAL:
    li   $s3, 1     # equal = true
STRCMP_END:
```

### 12.7 Sao chép xâu (giống `strcpy`)

C++:
```cpp
strcpy(dest, src);
```

MIPS:
```asm
    la   $t0, src
    la   $t1, dest

STRCPY_LOOP:
    lb   $t2, 0($t0)         # đọc ký tự từ src
    sb   $t2, 0($t1)         # ghi ký tự vào dest
    beq  $t2, $zero, STRCPY_END   # đã copy xong '\0'
    addi $t0, $t0, 1
    addi $t1, $t1, 1
    j    STRCPY_LOOP
STRCPY_END:
```

---

## 13. Bảng tổng hợp nhanh (cheat sheet)

| Nhóm | C++ | MIPS |
|---|---|---|
| Gán | `a = b;` | `move $t0, $t1` |
| Cộng | `a = b + c;` | `add $t0, $t1, $t2` |
| Trừ | `a = b - c;` | `sub $t0, $t1, $t2` |
| Nhân | `a = b * c;` | `mul $t0, $t1, $t2` |
| Chia | `a = b / c;` | `div $t1,$t2` + `mflo $t0` |
| So sánh nhỏ hơn | `a < b` | `slt $t0, $t1, $t2` |
| Đọc mảng | `x = a[i];` | `sll` → `add` → `lw` |
| Ghi mảng | `a[i] = x;` | `sll` → `add` → `sw` |
| If/else | `if(...){}else{}` | `beq/bne` + nhãn `ELSE`, `END_IF` |
| For | `for(...)` | nhãn lặp + `slt`/`slti` + `beq` thoát |
| While | `while(...)` | tương tự `for` nhưng không có bước khởi tạo/tăng cố định |
| Gọi hàm | `f(a,b)` | `$a0,$a1` → `jal` → kết quả ở `$v0` |
| Trả về | `return x;` | `move $v0, ...` + `jr $ra` |
| Con trỏ | `*p` | `lw`/`sw` với offset `0($reg)` |


---
add
```
I-mem -> mux -> reg -> mux -> alu -> mux -> reg
```

lw
```
I-mem -> mux -> reg -> mux -> alu -> D-mem -> mux -> reg
```

sw
```
I-mem -> reg -> alu -> D-mem
```

beq
```
I-mem -> reg -> mux -> alu -> mux
```

| Tên Lệnh / Nhóm | RegDst | ALUSrc | MemtoReg | RegWrite | MemRead | MemWrite | Branch | ALUOp |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :--- |
| **R-type**<br>(add, sub) | 1 (rd) | 0 (Reg) | 0 (ALU) | 1 | 0 | 0 | 0 | 10 (Theo funct) |
| **lw** (Nạp) | 0 (rt) | 1 (Hằng) | 1 (Mem) | 1 | 1 | 0 | 0 | 00 (Cộng) |
| **sw** (Lưu) | **X** | 1 (Hằng) | **X** | 0 | 0 | 1 | 0 | 00 (Cộng) |
| **beq**<br>(Nhảy) | **X** | 0 (Reg) | **X** | 0 | 0 | 0 | 1 | 01 (Trừ) |

*(Lưu ý: Dấu **X** mang ý nghĩa **"Don't Care"** - Nghĩa là tín hiệu này có mang giá trị 0 hay 1 cũng không hề ảnh hưởng đến hoạt động mạch, vì dòng dữ liệu đã bị chặn lại ở các công tắc điều khiển phía sau).*

---
| ALU Control | Operation                  |
|--------------|----------------------------|
| `0000`       | AND                        |
| `0001`       | OR                         |
| `0010`       | ADD (Cộng)                 |
| `0110`       | SUBTRACT (Trừ)             |
| `0111`       | SET ON LESS THAN (So sánh nhỏ hơn) |
| `1100`       | NOR                        |


---
```
performance = 1 / Execution Time
```

```
CPU time(s) = IC(số lệnh) * CPI(chu kì) * Tc(s)
```

```
CPU time(s) = (IC(số lệnh) * CPI(chu kì)) / f(Hz)
```

```
CPI trb = sum(CPI * %)
```

```
MIPS = IC / (Execution Time * 10^6)
```

```
MIPS = f / (CPI * 10^6)
```

| Tần số đập (f) | |
| :--- | ---: |
| KiloHertz (KHz) | 10<sup>3</sup> Hz |
| MegaHertz (MHz) | 10<sup>6</sup> Hz |
| GigaHertz (GHz) | 10<sup>9</sup> Hz |

<br>

| Thời gian (T) | |
| :--- | ---: |
| Milli giây (ms) | 10<sup>-3</sup> s |
| Micro giây (μs) | 10<sup>-6</sup> s |
| Nano giây (ns) | 10<sup>-9</sup> s |
| Pico giây (ps) | 10<sup>-12</sup> s |
