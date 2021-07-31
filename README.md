# PikaProject

A project for 32-bit RISC CPU architecture named PIKA.

Pika Project를 방문해주셔서 감사합니다. 
본 프로젝트는 CPU 아키텍처와 그를 기반으로 한 컴파일러와 프로세서를 만든 프로젝트입니다.

아래 세 가지로 구성됩니다.

1. PIKA CPU 아키텍처
2. PIKA LLVM 백엔드
3. PIKA RISC 프로세서


 32비트 RISC CPU의 컴파일러 백엔드와 Verilog 프로세서를 구현한 프로젝트입니다.

## 1. PIKA 아키텍처

PIKA라는 이름으로 간이 CPU 아키텍처를 정의하였습니다.

상용 아키텍처는 초보자가 접근하여 분석하기 어려운 두 가지 이유가 있습니다.

1. 명령어 개수가 너무 많다.
2. 명령어 비트 필드 구성이 복잡하다.

PIKA 아키텍처를 만들 때, 최대한 적은 명령어 수로 초보자가 직관적으로 접근하기 용이하도록 작성하려고 노력하였습니다.

약 1년간의 개발 기간 끝에 PIKA 아키텍처 v1.0이 탄생하였습니다.

### 1.1. Specification
PIKA 아키텍처의 스펙은 아래와 같습니다.
```
- 32-bit RISC CPU
- 18 Registers
	- 14 General Purpose Registers
	- 1 Stack Pointer
	- 1 Link Register
	- 1 Currrent Program Status Register
	- 1 Program Counter
- 38 Instructions
	- 27 ALU Instructions (rr-type, ri-type)
		- ADD, SUB, MUL, DIV,
		- AND, OR , NOT, XOR,
		- SHL, SHR, ASR, CMP,
		- MOV, MOVL, MOVH
	- 7 Jump Instructions
		- JMP (Unconditional Branch)
		- Jcc (Conditional Branch)
			- JEQ, JNE,
			- JLT, JLE, JGE, JGT
	- 2 Memory Instructions
		- LD  (Load)
		- STR (Store)
	- 2 Procedure Instructions
		- CALL (Call)
		- RET  (Return)
- Little Endian
- Supported features
	- Signed type
	- multiplication
	- division
	- ...
- Not supported features
	- 8bit or 16bit type
	- Unsigned type
	- Array type
	- Floating point
	- ...
```

### 1.2. Register Set

레지스터 명세표는 아래와 같습니다.

```
1. General Purpose Registers (Caller-Saved)
+----+----+----+----+----+----+
| r0 | r1 | r2 | r3 | r4 | r5 |
+----+----+----+----+----+----+

2. General Purpose Registers (Callee-Saved)
+----+----+----+----+-----+-----+-----+-----+
| r6 | r7 | r8 | r9 | r10 | r11 | r12 | r13 |
+----+----+----+----+-----+-----+-----+-----+

3. Specific Registers
+---------+---------+  +-----------+  +----+
| r14(lp) | r15(sp) |  | r16(cpsr) |  | pc |
+---------+---------+  +-----------+  +----+

```

<사진 첨부하기!>

<CPSR 명세 그림으로 표시하기!>


### 1.3. Instruction Set

명령어 세트는 아래와 같습니다.

<사진 첨부하기!>

### 1.4. 프로젝트 내려받기

```
git clone https://github.com/pikamonvvs/PikaProject.git
cd PikaProject
git submodule init && git submodule update
```

## 2. PIKA 컴파일러

LLVM 컴파일러의 백엔드를 수정하여 PIKA 아키텍처를 추가하였습니다.
Clang과 본 LLVM 백엔드를 이용하면 C언어로 작성된 소스 코드로부터 PIKA 아키텍처의 바이너리를 생성할 수 있습니다.

Clang 3.8.1 버전을 기반으로 만들었습니다.
~~(왜냐하면 참고했던 예제 코드가 LLVM 3.8.1 버전 베이스였기 때문입니다.)~~

### 2.1 빌드 환경

아래 환경에서 빌드하였습니다.

```
Ubuntu 20.04
gcc 9.3.0
cmake 3.16.3
ninja 1.10.0
make 4.2.1
```

위의 도구들이 설치되어있지 않다면, 아래의 명령어로 설치할 수 있습니다.

```
sudo apt install -y gcc g++ make cmake ninja-build libncurses5
```

### 2.2 빌드 방법

아래의 명령어들로 빌드할 수 있습니다.

```
// 프로젝트 내려받기
git clone https://github.com/pikamonvvs/PikaProject.git
cd PikaProject
git submodule init && git submodule update
cd PikaLLVM-backend

// 빌드하기
mkdir build
cd build
cmake -G Ninja -DCMAKE_BUILD_TYPE="Debug" -DLLVM_TARGETS_TO_BUILD=PIKA ../
ninja
```

환경마다 상이하나, 빌드되는 데에 약 1~2시간 정도 소요됩니다.
시간을 단축시키고 싶다면, ninja 명령어 입력 시 -j 옵션을 추가할 수 있습니다.
Out-of-memory가 발생하지 않도록 주의하여 추가하여야 합니다.

~~(시도해보진 않았지만, 아래의 명령어로 가장 빠른 빌드 속도를 볼 수 있지 않을까?)~~
```
ninja -j 16 || ninja -j 8 || ninja -j 4 || ninja -j 2 || ninja -j 1
```

### 2.3 사용법

빌드가 완료되면 build/bin 폴더 안에 llc 등의 프로그램이 생성된 것을 볼 수 있습니다.
아래의 명령어들을 이용하여 원하는 파일을 생성할 수 있습니다.

#### 2.3.1. 환경 변수 등록

본 프로젝트에서 제공되는 프로그램을 사용하기 위해 환경 변수 등록이 필요합니다.
아래 명령어들로 등록할 수 있습니다.

```
< 순서대로 입력해주세요 >
export PATH=/path/to/clang-3.8.1/bin:$PATH
export PATH=/path/to/PikaLLVM-backend/build/bin:$PATH
```

#### 2.3.2. 파일 생성

아래 명령어들로 원하는 파일을 생성할 수 있습니다.

예) 파일명을 test.c 라고 할 때,
```
< .c -> .ll >
clang -emit-llvm -S -O0 -o test.ll test.c

< .ll -> .bc >
llvm-as -o test.bc test.ll

< .ll -> .s >
llc -march=pika -o test.s test.ll

< .ll -> .o >
llc -march=pika -filetype=obj -o test.o test.ll

< .bc -> .s >
llc -march=pika -o test.s test.bc

< .bc -> .o >
llc -march=pika -filetype=obj -o test.o test.bc

< .o -> .hex >
llvm-objdump -s -j .text test.o > test.hex

```

## 3. PIKA 프로세서

Verilog HDL을 이용하여 PIKA 아키텍처에 정의된 명령어를 수행하는 프로세서를 구현하였습니다.

### 3.1. Specification

프로세서 스펙은 아래와 같습니다.

```
* 32-bit CPU
* 5-Stage Single-Cycle Processor
* IMEM Size: 1KB
* DMEM Size: 1KB
```
### 3.2. 컴파일 및 시뮬레이션 환경

아래 환경에서 컴파일 및 시뮬레이션 하였습니다.

```
Ubuntu 20.04
iverilog 10.3 (stable)
gtkwave 3.3.103 // GUI required
```

### 3.2. 컴파일 및 시뮬레이션 방법

gtkwave로 시뮬레이션을 하려면 GUI 환경에서 실행되어야 합니다.

<test.hex를 헤더 떼고 인풋으로 넣어서 컴파일 후 testbench를 실행시킬 방법!>
```
iverilog -I sources/ -DFOR_TEST -o test.vvp testbench.v
vvp test.vvp
gtkwave test.vcd
```
<이걸 make로 바꿀지!>

## 4. 결론

본 프로젝트를 통해 C언어로 작성된 소스 코드를 PIKA 아키텍처 바이너리로 컴파일하여 PIKA 프로세서 위에서 동작시킬 수 있습니다.

## 5. Future works

### 1. 컴파일러

3. 베이스로 한 llvm 버전이 옛날 버전임.

버전이 올라가면서 보다 많은 안정성 및 최적화 기능들을 제공할 가능성이 있으니, 높은 버전의 LLVM 베이스로 포팅하는 것은 가치 있어보임.

2. FPGA 위에서 합성 가능한지 테스트 안해봤음.

집에 xilinx 보드가 있으니, vivado 깔아서 돌려보면서 수정하면 될듯.

3. 시뮬레이터 부재

컴파일러 코드의 동작을 PC에서 확인할 수 있도록 Simulator를 만들어도 될듯.

### 2. 프로세서

1. 아직 CALL, RET 명령어의 동작이 구현되지 않음.

~~컨택 시기를 놓칠까봐 조급해져서..~~

2. 구조 최적화가 필요해 보임.

wire가 난잡하게 많은 것 같아서, 구조를 단순하게 고쳐보기

## 6. 참고문헌
 
 <엄청 많음!>
