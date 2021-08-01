
# A Pika Project!

Pika Project를 방문해주셔서 감사합니다.

본 프로젝트에서는 PIKA라는 자작 "32비트 RISC CPU 아키텍처"를 소개합니다.
CPU 아키텍처와 그를 기반으로 한 컴파일러 백엔드 및 프로세서를 구현한 프로젝트입니다.

크게 아래 세 가지로 구성됩니다.

1. PIKA 프로젝트 개요
2. CPU 아키텍처
3. LLVM 백엔드
4. RISC 프로세서

## 1. PIKA 프로젝트 개요

### 1.1. 목표

PIKA 프로젝트의 목표는 최대한 간단한 CPU 아키텍처를 나름대로 직접 정의해보는 것입니다. 정의한 아키텍처의 명세에 맞춰 기계어 코드를 생성해내는 컴파일러와, 생성된 기계어 코드를 실행하는 프로세서를 만드는 것입니다.

### 1.2. 기간

약 6개월 정도로 추산됩니다. 2020년 8월부터 2021년 8월까지 진행하였으나, 중간에 몇 달간 프로젝트를 중단한 점, 프로젝트를 주말에만 진행하였다는 점 등을 감안하여 합리적으로 계산하였습니다.

### 1.3. 전체 흐름

PIKA 프로젝트의 전체적인 흐름은 아래와 같습니다.

![PIKA Flow Diagram](https://github.com/pikamonvvs/PikaProject/blob/master/resources/Flow%20Diagram.png)

C언어로 작성된 소스 코드를 입력으로 받습니다. Clang을 통하여 소스 코드를 LLVM IR로 변환하며, PIKA 컴파일러 백엔드를 이용하여 LLVM IR을 PIKA Target Dependent 기계어 코드로 변환합니다. 생성된 파일 형태의 기계어 코드를 PIKA 프로세서의 Testbench의 입력으로 넣으면 Gtkwave를 통해 시각적인 Waveform 형태로 Simulation할 수 있습니다.

## 1. PIKA 아키텍처

PIKA라는 이름으로 간이 CPU 아키텍처를 디자인하였습니다.

상용 아키텍처는 초보자가 접근하여 분석하기 어려운 두 가지 이유가 있습니다.

1. 명령어 개수가 너무 많다.
2. 명령어 비트 필드 구성이 복잡하다.

PIKA 아키텍처를 만들 때, 최대한 적은 명령어 수로 초보자가 직관적으로 접근하기 용이하도록 작성하려고 노력하였습니다.

약 6개월간의 개발 기간 끝에 PIKA 아키텍처가 탄생하였습니다.

### 1.1. Specification

PIKA 아키텍처의 스펙은 아래와 같습니다.

* Specification

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
```

* Supported Features

```
	- Signed Type
	- Multiplication
	- Division
	- ...
```

- Not Supported Features

```
	- 8bit or 16bit Type
	- Unsigned type
	- Array type
	- Floating point
	- ...
```

### 1.2. Register Set

총 18개의 32비트 레지스터로 구성됩니다. 14개의 범용 레지스터와 4개의 특수 목적 레지스터로 이루어져 있습니다.

레지스터 명세표는 아래와 같습니다.

* Register Table

![PIKA Register Set](https://github.com/pikamonvvs/PikaProject/blob/master/resources/Register%20Set.png)

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
| r14(sp) | r15(lr) |  | r16(cpsr) |  | pc |
+---------+---------+  +-----------+  +----+

```

* CPSR Bit Field Format

```
  bit31    ...    bit 4   bit 3   bit 2   bit 1   bit 0 
+-------+-------+-------+-------+-------+-------+-------+
|   -   |   -   |   -   |   n   |   z   |   c   |   v   |
+-------+-------+-------+-------+-------+-------+-------+
  n = negative bit
  z = zero bit
  c = carry bit
  v = overflow bit / underflow bit
 (- = reserved)
```

### 1.3. Instruction Set

총 38개의 명령어로 구성되어 있습니다. 산술/논리/시프트/이동 연산, 비교 연산, 분기, 메모리 접근, 프로시저 호출/복귀 명령어로 구성되어 있습니다.

명령어 명세표는 아래와 같습니다.

![PIKA Instruction Set](https://github.com/pikamonvvs/PikaProject/blob/master/resources/Instruction%20Set.png)

* Conditional Branch Instruction Bit Field Format

```
   31-26   25-22            21-0
+--------+-------+------------------------+
| 100011 |  cond |         offset         |
+--------+-------+------------------------+
 cond
  JEQ = 0001
  JGT = 0010
  JGE = 0011
  JLT = 0100
  JLE = 0101
  JNE = 0110
```

### 1.4. 프로젝트 내려받기

아래의 방법으로 전체 프로젝트를 내려받을 수 있습니다.

```
git clone https://github.com/pikamonvvs/PikaProject.git
cd PikaProject
git submodule init && git submodule update
```

## 2. PIKA 컴파일러

Clang-3.8.1과 본 LLVM 백엔드를 이용하면 C언어로 작성된 소스 코드로부터 PIKA 아키텍처의 바이너리를 생성할 수 있습니다.
Clang-3.8.1은 루트 프로젝트 내에 첨부해두었습니다.

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

### 2.2. 빌드 방법

아래의 방법으로 빌드할 수 있습니다.

```
# 프로젝트 내려받기
git clone https://github.com/pikamonvvs/PikaProject.git
cd PikaProject
git submodule init && git submodule update
cd PikaLLVM-backend

# 빌드하기
mkdir build
cd build
cmake -G Ninja -DCMAKE_BUILD_TYPE="Debug" -DLLVM_TARGETS_TO_BUILD=PIKA ../
ninja (또는 ninja -j 8)
```

빌드되는 데에 약 1~2시간 정도 소요됩니다. (물론 환경에 따라 다릅니다.)
시간 단축을 위해 ninja 명령어에 -j 옵션을 줄 수 있습니다. 다만 Out-of-memory가 발생하지 않도록 주의하여야 합니다.
만약 OOM이 발생하여도, 다시 ninja를 실행함으로써 이어서 빌드할 수 있습니다.

### 2.3. 사용법

빌드가 완료되면 build/bin 폴더 안에 llc 등의 프로그램이 생성됩니다.

#### 2.3.1. 환경 변수 등록

생성된 프로그램을 이용하기 위해서는 bin 디렉토리의 경로를 환경 변수에 추가해주어야 합니다.
주의할 점은 해당 디렉토리가 기존에 PC에 설치되어있을 수 있는 clang이나 llc보다 먼저 조회되기 위해 PATH 변수의 가장 앞에 추가해주어야 한다는 것입니다.
뒤에 추가하게 되면 원치 않는 결과를 직면하게 될 것입니다. ~~(절대 경험담은 맞습니다.. T.T)~~

아래 명령어들로 등록할 수 있습니다.

```
< 순서대로 입력해주세요! >
export PATH=/path/to/clang-3.8.1/bin:$PATH
export PATH=/path/to/PikaLLVM-backend/build/bin:$PATH
```

#### 2.3.2. 파일 생성

아래 명령어들로 원하는 파일을 생성할 수 있습니다.

```
*.c		= C Source Code
*.ll		= LLVM Intermediate Representation
*.bc		= LLVM Bitcode
*.s		= Target Assembly Code (PIKA)
*.o		= Target Object (PIKA)
test.hex	= Target Dependent Code (PIKA) - PikaRISC의 입력으로 사용됨
```

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

< .o 파일 덤프 출력 >
llvm-objdump -s test.o

< .o -> .hex > - PikaRISC의 입력으로 사용하기 위한 형식
llvm-objdump -s -j .text test.o | cut -d' ' -f3,4,5,6 | sed '1,4d' > test.hex

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

### 3.2. 전체 구조

PIKA 프로세서의 전체적인 구조는 아래와 같습니다.

![PIKA Structural Diagram](https://github.com/pikamonvvs/PikaProject/blob/master/resources/Structural%20Diagram.png)

Fetch, Decode, Execute, Memory, Writeback으로 총 5단계로 구분되어 있습니다. PC와 CPSR은 레지스터 파일 안에 들어있으며, 레지스터 파일은 프로세서 안에 내장되어 있습니다. 반면에 명령어 메모리와 데이터 메모리는 프로세서 바깥으로 빼놓았습니다. 이는 테스트벤치를 통해 디버깅을 쉽게 하기 위함입니다.

다섯 단계로 나뉘어져 있지만 아쉽게도 파이프라인은 아닙니다. 성능보다는 단순한 구조를 중점으로 두었기 때문에 싱글 사이클 모델로 설계하였습니다.

### 3.3. 컴파일 및 시뮬레이션 환경

아래 환경에서 컴파일 및 시뮬레이션 하였습니다.

```
Windows 10
iverilog 12.0 (devel)
gtkwave 3.3.108
```

(※주의: gtkwave는 반드시 GUI 환경에서 실행되어야 합니다.)

### 3.4. 컴파일 및 시뮬레이션 방법

컴파일은 iverilog를 이용하였으며, 시뮬레이션은 vvp와 gtkwave를 이용하였습니다.

아래의 방법으로 컴파일 및 시뮬레이션할 수 있습니다. testbench.v

```
iverilog -I sources/ -DFOR_TEST -o test.vvp testbench.v
vvp test.vvp
gtkwave test.vcd
```
또는 프로젝트 내에 Makefile을 만들어 두었으므로, make를 통해 동일한 명령을 수행할 수 있습니다.

```
make
```

## 4. Future works

PIKA 아키텍처는 아직 부족한 점이 많습니다. 이는 차후에 future works로서 보완되면 좋을 것 같습니다.

### 4.1. PIKA 컴파일러

1. 옛날 버전의 LLVM 베이스 버전

2021년 7월 31일 기준, LLVM의 최신 버전은 12.0.1이며, 본 프로젝트에서는 3.8.1 버전을 사용하였습니다. 이는 참고했던 코드의 LLVM 버전이 3.8.1이었기 때문에, 코드 호환성 유지를 위해 동일 버전에 포팅하였기 때문입니다. 보다 많은 기능 및 안정성을 위해 높은 버전으로의 포팅하는 것은 의미있어 보입니다.

2. 에뮬레이터의 부재

실물 하드웨어 없이 기계어 코드를 검증할 수 있도록 PC용 에뮬레이터를 만드는 것도 개발 및 디버깅에 많은 도움이 될 것으로 보입니다.

3. 지원하는 자료형 부족

아직 signed int 타입 이외에 검증되지 않았습니다. 실제 C언어에서 사용되는 타입들이 모두 이용 가능하도록 수정이 필요합니다.

### 4.2. PIKA 프로세서

1. CALL, RET 동작 미구현

아직 CALL, RET 명령어의 동작이 구현되지 않았습니다. 가장 합리적인 구현 방법을 구상하고 있습니다.

2. FPGA 합성 가능 여부 미확인

본 코드가 실제 FPGA 위에서 합성 가능한지 검증되지 않았습니다. 추후 Xilinx Vivado 툴을 이용하여 합성 및 Program 예정입니다.

## 5. 참고문헌
 
본 프로젝트를 진행하는데 도움을 준 수많은 자료들 및 기여자분들께 감사합니다!

### 5.1. Web Sites

* Tutorial: Creating an LLVM Backend for  the Cpu0 Architecture
http://jonathan2251.github.io/lbd/TutorialLLVMBackendCpu0.pdf

* Writing an LLVM Backend — LLVM 10 documentation
https://llvm.org/docs/WritingAnLLVMBackend.html

* The LLVM Target-Independent Code Generator — LLVM 10 documentation
https://llvm.org/docs/CodeGenerator.html

* LLVM Instruction Set
https://llvm.org/docs/LangRef.html#instruction-reference

* LLVM SDNode 종류
https://llvm.org/doxygen/namespacellvm_1_1ISD.html#a22ea9cec080dd5f4f47ba234c2f59110a0158ee47dfa868be5d28e2cbef70d5d0

* TableGen — LLVM 10 documentation
http://llvm.org/docs/TableGen/index.html

* What is MC Layer?
https://llvm.org/docs/CodeGenerator.html#the-mc-layer

* Generating object files — Tutorial: Creating an LLVM Backend for the Cpu0 Architecture
https://jonathan2251.github.io/lbd/genobj.html

* Global variables — Tutorial: Creating an LLVM Backend for the Cpu0 Architecture
https://jonathan2251.github.io/lbd/globalvar.html

* LLVM: llvm::TargetLowering Class Reference
https://llvm.org/doxygen/classllvm_1_1TargetLowering.html

* LLVM Language Reference Manual — LLVM 10 documentation
https://llvm.org/docs/LangRef.html#terminator-instructions

* B, BL, BX and BLX - TRACE32
http://trace32.com/wiki/index.php/B,_BL,_BX_and_BLX

* Chapter2 - Instruction Set
http://aelmahmoudy.users.sourceforge.net/electronix/arm/chapter2.htm

* D17063 Add LLVM_FALLTHROUGH macro to Compiler.h
https://reviews.llvm.org/D17063

* The Design of a Custom 32-bit RISC CPU and LLVM Compiler Backend
https://scholarworks.rit.edu/cgi/viewcontent.cgi?article=10699&context=theses

* Data types — FPGA designs with Verilog and SystemVerilog documentation
https://verilogguide.readthedocs.io/en/latest/verilog/datatype.html

* SystemVerilog repeat and forever loop - Verification Guide
https://verificationguide.com/systemverilog/systemverilog-repeat-and-forever-loop/#repeat_loop_syntax

* Condition Codes 1: Condition flags and codes - Processors blog - Processors - Arm Community
https://community.arm.com/developer/ip-products/processors/b/processors-ip-blog/posts/condition-codes-1-condition-flags-and-codes

* Pipelined MIPS Processor in Verilog (Part-1) - FPGA4student.com
https://www.fpga4student.com/2017/06/32-bit-pipelined-mips-processor-in-verilog-1.html

* Pipelined MIPS Processor in Verilog (Part-2) - FPGA4student.com
https://www.fpga4student.com/2017/06/32-bit-pipelined-mips-processor-in-verilog-2.html

* Pipelined MIPS Processor in Verilog (Part-3) - FPGA4student.com
https://www.fpga4student.com/2017/06/32-bit-pipelined-mips-processor-in-verilog-3.html


### 5.2. PPTs/PDFs

* Reed_Kotler.pdf
https://llvm.org/devmtg/2012-04-12/Slides/Reed_Kotler.pdf

* Building an LLVM backend.pdf
https://llvm.org/devmtg/2014-04/PDFs/Talks/Building%20an%20LLVM%20backend.pdf

* MSP430 Architecture pdf
http://www.electronics.teipir.gr/menu_el/personalpages/papageorgas/download/mcu_embedded/ti_india/vtu_lecture2.pdf

* llvm 소개
https://www.slideshare.net/minhyuks/llvm-49666289

* Tutorial: Building a backend in 24 hours : Anton_Korobeynikov.pdf
http://llvm.org/devmtg/2009-10/Korobeynikov_BackendTutorial.pdf

* LLVM Backend の紹介
https://www.slideshare.net/AkiraMaruoka/llvm-backend

* Microsoft PowerPoint - 09-pipelined-cpu-i
https://www.cs.cornell.edu/courses/cs3410/2012sp/lecture/09-pipelined-cpu-i-g.pdf

* project1.pdf
https://web.ece.ucsb.edu/~strukov/ece154b/labs/project1.pdf

### 5.3. Github

* Cpu0 LLVM Backend Github
https://github.com/Jonathan2251/lbd/tree/master/lbdex/Cpu0

* GitHub - llvm-mirror
https://github.com/llvm-mirror/llvm

* GitHub - llvm-mirror/llvm/lib/Target/MSP430
https://github.com/llvm-mirror/llvm/tree/master/lib/Target/MSP430

* GitHub - cpu0 llvm backend
https://github.com/Jonathan2251/lbd

* ValueTypes td
https://github.com/llvm-mirror/llvm/blob/master/include/llvm/CodeGen/ValueTypes.td

* Target td
https://github.com/llvm-mirror/llvm/blob/master/include/llvm/Target/Target.td

* TargetSelectionDAG td
https://github.com/llvm-mirror/llvm/blob/master/include/llvm/Target/TargetSelectionDAG.td

* TargetRegistry.h
https://github.com/llvm-mirror/llvm/blob/master/include/llvm/Support/TargetRegistry.h

* GitHub - frasercrmck/llvm-leg
https://github.com/frasercrmck/llvm-leg

* llvm-openrisc/lib/Target/OpenRISC at openrisc · asl/llvm-openrisc
https://github.com/asl/llvm-openrisc/tree/openrisc/lib/Target/OpenRISC

* style-guides/VerilogCodingStyle.md at master · lowRISC/style-guides
https://github.com/lowRISC/style-guides/blob/master/VerilogCodingStyle.md

* chsasank/ARM7: Implemetation of pipelined ARM7TDMI processor in Verilog
https://github.com/chsasank/ARM7

* takah29/arm-cpu: Armv5 single-cycle processor
https://github.com/takah29/arm-cpu

* connorjan/llvm-cjg: An LLVM backend for my custom 32-bit RISC CPU The Design of a https://scholarworks.rit.edu/theses/9550/
https://github.com/connorjan/llvm-cjg

* connorjan/cjg_risc: The final version of my custom 32-bit RISC CPU
https://github.com/connorjan/cjg_risc

* neelkshah/MIPS-Processor: 5-stage pipelined 32-bit MIPS microprocessor in Verilog
https://github.com/neelkshah/MIPS-Processor

* kavinr/5-Stage-MIPS: 5 Stage Pipelined MIPS Processor Implementation in Verilog
https://github.com/kavinr/5-Stage-MIPS

* MIPS-pipeline-processor/modules/pipeRegisters at master · mhyousefi/MIPS-pipeline-processor · GitHub
https://github.com/mhyousefi/MIPS-pipeline-processor/tree/master/modules/pipeRegisters

* GitHub - maze1377/pipeline-mips-verilog: A classic 5-stage pipeline MIPS 32-bit processor. solve every hazard with stall
https://github.com/maze1377/pipeline-mips-verilog

