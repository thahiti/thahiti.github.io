---
layout: post
title: "python app.py를 실행하면 벌어지는 일 — CPython 부팅부터 바이트코드 실행까지"
date: 2026-06-25 14:00:00 +0900
categories:
  - Computer Science
tags:
  - python
  - cpython
  - bytecode
  - interpreter
  - compiler
media_subpath: /assets/img/posts/cpython-boot-to-execution/
---
# python app.py를 실행하면 벌어지는 일 — CPython 부팅부터 바이트코드 실행까지

터미널에 `python app.py`를 치면 잠깐의 정적 뒤에 결과가 출력됩니다. 그 짧은 사이에 인터프리터 안에서는 꽤 많은 일이 순서대로 벌어집니다. 런타임이 부팅되고, 소스가 실행 가능한 형태로 변환되고, 그것이 한 줄씩 돌아갑니다. 평소에는 들여다볼 일이 없지만, "함수 안의 변수가 왜 local이냐 global이냐는 실행이 아니라 컴파일 시점에 정해지는가", "왜 `print`는 import 없이 어디서나 쓰이는가", "venv는 대체 무엇을 바꾸는가" 같은 질문은 결국 이 과정 어딘가에 답이 있습니다.

이 글에서는 그 전체 과정을 **직접 구현해 볼 수 있을 만큼** 잘게 분해합니다. 한 줄로 요약하면 흐름은 이렇습니다. 프로세스 부팅 → 런타임 자료구조 구축 → 소스를 code 객체로 컴파일 → frame을 만들어 stack machine으로 실행. 아래에서 각 단계를 차례로 뜯어보겠습니다. (CPython 기준이며, 버전에 따라 세부가 다른 부분은 표시합니다.)

> **한 줄 요약**
> 파이썬 실행은 네 단계입니다. ① 런타임을 두 단계(core/main init)로 부팅해 상태 객체와 빌트인 타입, `sys.path`를 세우고, ② 소스를 tokenize→parse→symtable→compile하여 정적 정보 덩어리인 **code 객체**로 만든 뒤, ③ 그 code 객체에 네임스페이스를 묶은 **frame**을 만들어, ④ **stack machine** eval 루프가 바이트코드를 한 opcode씩 실행합니다. "최상위 변수는 모듈 `__dict__`에 들어가는 global"이라는 흔한 설명도, "함수 지역변수가 빠르다"는 통념도 모두 이 과정의 산물입니다.

---

## 핵심 자료구조 3개

부팅을 이해하려면 먼저 인터프리터가 들고 다니는 상태 객체 3계층을 알아야 합니다.

| 구조체 | 범위 | 담고 있는 것 |
|--------|------|-------------|
| `_PyRuntimeState` | 프로세스당 1개 | 전역 런타임, 메모리 allocator, GIL 등 |
| `PyInterpreterState` | 인터프리터당 1개(보통 1개) | `sys.modules`(모듈 캐시), `builtins`, import 시스템 |
| `PyThreadState` | 스레드당 1개 | 현재 frame, 예외 상태, recursion depth |

특히 `PyInterpreterState.modules`가 곧 우리가 아는 `sys.modules`입니다. 그리고 "같은 모듈을 몇 번 import해도 본문은 딱 한 번만 실행된다"는, 처음엔 신기하게 느껴지는 규칙의 정체가 바로 이 dict입니다.

원리는 의외로 단순합니다. `import foo`를 만나면 인터프리터는 곧장 디스크를 뒤지지 않고, **먼저 `sys.modules`에 `'foo'` 키가 있는지부터 확인합니다.**

- **있으면 (캐시 히트):** 그 자리에서 캐시된 모듈 객체를 그대로 돌려주고 끝냅니다. 모듈 본문(top-level 코드)은 다시 실행하지 않습니다.
- **없으면 (캐시 미스):** ① 비어 있는 새 모듈 객체를 만들어 **본문을 실행하기 *전에* 먼저 `sys.modules['foo']`에 등록**하고, ② 그제서야 모듈 본문을 실행해 그 네임스페이스를 채웁니다.

그래서 두 번째 import부터는 늘 ①을 건너뛴 캐시 히트로 끝나고, 본문은 처음 한 번만 돕니다. "한 번만 실행"이라는 규칙이 무슨 특별한 장치가 아니라, 그저 이 조회 순서가 만들어 내는 결과인 셈입니다. 등록을 실행보다 *먼저* 한다는 점도 중요합니다. A가 B를 import하고 B가 다시 A를 import하는 순환 상황에서, A가 아직 덜 채워졌더라도 이미 `sys.modules`에 올라가 있으므로 같은 모듈을 다시 로드하러 들어가지 않아 무한 재귀를 면합니다. [공식 문서](https://docs.python.org/3/reference/import.html#loading)도 "모듈은 로더가 코드를 실행하기 *전에* `sys.modules`에 존재한다"고 못 박고 있습니다.

## 부팅 1단계 — 런타임 초기화 (core init)

OS가 `python` 바이너리를 exec하면 C `main()` → `Py_BytesMain` → `Py_RunMain`으로 들어갑니다. 초기화는 PEP 587(3.8+) 이후 두 단계로 나뉩니다.

**Core init**에서는 가장 원초적인 것들을 세웁니다. 메모리 allocator, 위의 상태 객체 3개, 그리고 **빌트인 타입**들(`type`, `int`, `str`, `tuple`...)입니다. 타입은 다른 모든 것보다 먼저 존재해야 합니다. `int` 객체를 만들려면 `int` 타입이 이미 있어야 하고, `int` 타입 자신은 `type`의 인스턴스여야 하기 때문입니다. 그럼 이 닭과 달걀 문제를 어떻게 풀까요. 핵심은, 이 타입들이 실행 중에 동적으로 *만들어지는* 게 아니라 C 소스에 **정적으로 박혀(statically allocated)** 있다는 데 있습니다. 즉 객체 자체는 컴파일 시점에 이미 존재하고, 초기화 때 `PyType_Ready`가 이들 사이의 관계(누가 누구의 타입인지, 어떤 메서드 슬롯을 갖는지)를 채워 넣어 고리를 완성합니다. "`int`의 타입은 `type`, `type`의 타입은 자기 자신"이라는 순환을 런타임 생성이 아니라 *미리 박아 둔 객체에 포인터를 이어 주는* 방식으로 끊는 것입니다. 이어서 `sys` 모듈의 뼈대와 `builtins` 모듈을 만듭니다.

이 시점의 난제가 하나 더 있습니다. **import 시스템 자체를 import할 수 없다**는 점입니다. 왜 그럴까요. CPython의 import 머신은 사실 C가 아니라 파이썬으로 작성돼 있는데(`importlib._bootstrap`, `importlib._bootstrap_external`), 이 파일들을 디스크에서 읽어 오려면 바로 그 import 시스템이 이미 돌고 있어야 합니다 — 자기 자신을 import해야 하는 또 다른 순환입니다. 해결책은 이 부트스트랩 모듈들의 바이트코드를 **미리 컴파일해 인터프리터 바이너리 안에 동결(freeze)**해 두는 것입니다. 그러면 디스크 탐색이나 경로 기반 import 없이, 메모리에 박힌 frozen 바이트코드(`_frozen_importlib`)에서 곧장 로드할 수 있습니다. 이렇게 손으로 부팅시킨 import 시스템이 비로소 나머지 평범한 모듈들을 디스크에서 정상적으로 불러오기 시작합니다.

## 부팅 2단계 — main init과 sys.path의 의미

**Main init**에서는 완전한 import 시스템(path 기반 finder들)을 켜고, encoding을 설정하고, `__main__` 모듈을 준비하고, `sys.path`를 계산합니다.

**`sys.path`의 의미**는 "모듈을 찾을 때 순서대로 뒤지는 디렉토리 문자열 리스트"입니다. import 시스템의 `PathFinder`가 이 리스트를 위에서부터 훑습니다. 이 리스트는 다음으로 채워집니다.

- `sys.path[0]`: 실행 방식에 따라 다릅니다. `python app.py`면 **app.py가 있는 디렉토리**, `python -m`이면 cwd입니다.
- `PYTHONPATH` 환경변수 항목들
- 표준 라이브러리 위치 — 인터프리터가 자기 실행 파일이 놓인 자리에서 출발해, 표준 라이브러리에 항상 존재하는 표지 파일(landmark, 예컨대 `os.py`)이 보이는 디렉토리를 위로 거슬러 올라가며 찾아 설치 루트(`sys.prefix`)를 역산하는 방식입니다. 표지 파일을 발견한 위치가 곧 "여기가 표준 라이브러리다"라는 단서가 됩니다. 이 "getpath" 로직은 3.11부터 frozen된 `getpath.py`가 담당합니다.
- site-packages — `site` 모듈이 시작 시(`-S`가 없으면) 추가합니다.

즉 venv가 작동하는 원리도 결국 이 단계에서 `sys.prefix`가 venv를 가리키게 되어 site-packages 경로가 바뀌는 것뿐입니다. 한 꺼풀 더 들여다보면 기계 장치는 이렇습니다. venv 디렉토리에는 `pyvenv.cfg`라는 작은 파일이 있고, 그 안의 `home` 키가 venv를 만들어 낸 원본 파이썬 설치를 가리킵니다. 시작 시 `site` 모듈이 실행 파일 근처에서 이 파일을 발견하면 "지금 돌고 있는 건 venv다"라고 판단하고 `sys.prefix`를 venv 쪽으로 잡습니다(표준 라이브러리 본체는 `home`을 통해 원본에서 그대로 빌려 옵니다). site-packages 경로는 이 `sys.prefix`에서 파생되므로, 결과적으로 `pip install`은 venv 안 site-packages에 설치되고 import도 거기서 패키지를 찾게 됩니다. 원본 설치 위치는 `sys.base_prefix`에 따로 남아 둘을 구분합니다 — 그래서 venv 안인지 아닌지는 `sys.prefix != sys.base_prefix`로 판별할 수 있습니다. `uv run`이 해 주는 일도 정확히 이 지점, 올바른 인터프리터를 골라 `sys.path`가 맞는 site-packages를 보게 하는 것입니다.

## 모듈, 모듈 객체, `__dict__`의 정체

이제 개념 셋을 한 번에 정리하겠습니다. C 레벨에서 모듈은 `PyModuleObject`이며, 구조를 단순화하면 이렇습니다.

```c
typedef struct {
    PyObject_HEAD
    PyObject *md_dict;   // 이것이 바로 __dict__ (네임스페이스)
    PyObject *md_name;
    // ...
} PyModuleObject;
```

**모듈 객체**는 그냥 이 구조체의 인스턴스입니다(`types.ModuleType`). **`__dict__`**는 그 안의 `md_dict` 필드, 즉 이름을 객체에 매핑하는 dict입니다. `module.foo`로 속성에 접근하면 내부적으로 `md_dict['foo']`를 찾습니다. 결정적으로, **모듈의 top-level 변수는 전부 이 `md_dict`의 항목**입니다. 흔히 "최상위 변수는 모듈 `__dict__`에 들어가는 global"이라고 설명하는데, 그 C 레벨 실체가 바로 이 필드입니다.

## 소스에서 code 객체까지 — 컴파일 파이프라인

**컴파일**은 소스 텍스트를 실행 가능한 `code 객체`로 바꾸는 과정이며 네 단계입니다.

1. **Tokenizer** (lexer): 바이트 스트림을 토큰으로 자릅니다(`NAME`, `NUMBER`, `OP`, `NEWLINE`, 그리고 들여쓰기를 추적하는 `INDENT` `DEDENT`). 인코딩 선언도 여기서 처리합니다.
2. **Parser**: 토큰을 **AST**(추상 구문 트리)로 만듭니다. Python 3.9부터 기존 LL(1) 대신 PEG parser가 기본이고 3.10에서 옛 parser는 제거됐습니다. 문법 오류(`SyntaxError`)는 이 단계까지에서 잡힙니다. 아직 `NameError` 같은 런타임 오류는 없습니다.
3. **Symbol table** (symtable): AST를 훑어 각 scope에서 모든 이름을 local/global/free(closure)/cell로 분류합니다. **이 단계가 LEGB 규칙을 바이트코드 생성 전에 미리 확정합니다.** 함수 안의 `x`가 그 함수에서 대입된 적 없으면 여기서 "global"로 못 박히고, 그 결과 나중에 `LOAD_GLOBAL`로 컴파일됩니다. "함수 안에서 변수가 local이냐 global이냐"가 실행 시점이 아니라 컴파일 시점에 결정되는 이유입니다.
4. **Compiler**: AST와 symbol table을 받아 바이트코드를 뽑고 이를 묶어 `code 객체`를 만듭니다(3.12+에는 중간 CFG와 optimizer 단계가 더 있습니다).

![소스 텍스트가 Tokenizer·Parser·Symtable·Compiler를 거쳐 code 객체가 되는 5단계 컴파일 파이프라인](diagram-01-compile-pipeline.svg)

**`code 객체`**(`PyCodeObject`)는 실행에 필요한 모든 정적 정보를 담습니다. 주요 필드는 다음과 같습니다.

- `co_code`: 바이트코드 자체(opcode와 operand의 바이트열)
- `co_consts`: 사용된 상수 튜플(리터럴, 중첩 code 객체, docstring)
- `co_names`: global/속성 접근에 쓰는 이름 튜플 (`LOAD_GLOBAL` `STORE_NAME`이 인덱스로 참조)
- `co_varnames`: 지역 변수 이름 (`LOAD_FAST` `STORE_FAST`가 인덱스로 참조)
- `co_freevars` `co_cellvars`: 클로저 변수
- `co_stacksize`: value stack의 최대 깊이 (frame이 미리 할당할 크기)
- `co_filename` `co_firstlineno` 및 줄 번호 테이블

여기서 결정적인 이분법이 보입니다. **모듈/클래스 본문은 `STORE_NAME`/`LOAD_NAME`**(dict 기반, `co_names` 인덱스)을 쓰고, **함수 본문은 `STORE_FAST`/`LOAD_FAST`**(배열 인덱스, `co_varnames`)을 씁니다. 함수 지역변수가 빠르고 모듈 global이 dict 조회인 근본 이유가 이 컴파일 결과의 차이입니다. 왜 한쪽이 빠를까요. 함수의 지역변수는 컴파일 시점에 이미 "몇 번째 칸"인지가 정해져 있어, 실행 때는 배열의 그 인덱스를 포인터 산술 한 번으로 곧장 집어내면 됩니다(이름이 무엇인지조차 볼 필요가 없습니다). 반면 global·속성 접근은 이름 문자열을 키로 dict를 조회해야 하고, 이는 해시 계산과 충돌 처리를 매번 거칩니다. 둘 다 평균적으로는 빠르지만, 단순 인덱싱과 해시 조회의 차이가 그대로 속도 차이로 나타나는 것입니다.

## code 객체를 실행하기 — frame과 stack machine

code 객체는 정적 데이터일 뿐 스스로 돌지 않습니다. 실행하려면 **frame**을 만듭니다. frame이 들고 있는 핵심은 이렇습니다.

- `f_globals`: global 네임스페이스 dict — **곧 그 모듈의 `__dict__`**
- `f_locals`: local 네임스페이스 (모듈 본문에서는 globals와 동일, 함수에서는 fast-locals 배열)
- `f_builtins`: builtins dict
- value stack: 연산용 피연산자 스택
- instruction pointer: 현재 실행 위치

CPython은 레지스터 머신이 아니라 **stack machine**입니다. 모든 opcode가 value stack을 push/pop합니다. 예를 들어 `BINARY_OP +`는 두 값을 pop해서 더한 뒤 결과를 push합니다. 실행 엔진은 `_PyEval_EvalFrameDefault`라는 거대한 루프로, opcode를 하나씩 디코드해 분기 실행합니다. 개념적 의사코드는 이렇습니다.

```python
def eval_frame(code, f_globals, f_builtins):
    stack = []                    # value stack
    consts, names = code.co_consts, code.co_names
    ip = 0
    while ip < len(code.co_code):
        op, arg = decode(code.co_code, ip); ip += 1
        if op == LOAD_CONST:  stack.append(consts[arg])
        elif op == STORE_NAME: f_globals[names[arg]] = stack.pop()  # 모듈 top-level 대입
        elif op == LOAD_NAME:  stack.append(lookup_name(names[arg], f_globals, f_builtins))
        elif op == LOAD_GLOBAL: stack.append(lookup_global(names[arg], f_globals, f_builtins))
        # ... 나머지 opcode
```

실제 구현체는 단순 `switch`가 아니라 computed goto로 dispatch해 분기 예측 효율을 높입니다.

> **버전 참고**
> 기본 모델은 위의 stack 기반 eval 루프 그대로지만, 최근 버전들은 그 위에 최적화를 얹어 왔습니다. 3.11부터는 실행 중 opcode를 특수화하는 adaptive interpreter(PEP 659)와 lazy하게만 생성되는 `_PyInterpreterFrame`이, 3.12부터는 DSL(`bytecodes.c`)로 생성된 인터프리터가, 3.13에는 실험적 JIT이 더해졌습니다. 그래도 토대가 되는 실행 모델은 변하지 않았습니다.

## "global을 lookup한다"의 정체와 빌트인

**"global을 lookup한다"**는 추상적 표현이 아니라 특정 opcode의 동작입니다. 이름 종류별로 컴파일된 opcode가 다릅니다.

- `LOAD_FAST n`: 함수 지역변수를 fast-locals 배열의 인덱스 `n`에서 읽습니다 (dict 조회 없음, 빠름).
- `LOAD_DEREF`: 클로저의 free/cell 변수입니다.
- `LOAD_GLOBAL n`: `co_names[n]` 이름을 **먼저 `f_globals`(모듈 `__dict__`)에서, 없으면 `f_builtins`에서** 찾습니다. 둘 다 없으면 `NameError`입니다.
- `LOAD_NAME n`: 모듈/클래스 본문에서 쓰이며 `f_locals` → `f_globals` → `f_builtins` 순으로 찾습니다 (동적 LEGB).

즉 함수가 module global을 "늦게(late binding)" 보는 이유가 이것입니다. 컴파일 시점에 값을 박는 게 아니라, **호출 시점에 이름으로 `f_globals` dict를 조회**하기 때문입니다. 그래서 함수 정의보다 아래에서 정의한 global도 그 함수가 나중에 호출되기만 하면 정상 참조됩니다.

**빌트인**은 이 fallback의 종착지입니다. `builtins`는 하나의 모듈이고 `print`, `len`, `int`, `Exception` 같은 이름을 담고 있습니다. 모든 frame의 `f_builtins`가 이 모듈의 dict를 가리키므로, 어떤 코드에서든 import 없이 `print`가 동작합니다. global에 없으면 자동으로 빌트인까지 떨어지는 이 2단계 조회가 "왜 `print`는 어디서나 쓰이는가"의 답입니다.

## app.py 전체 흐름 통합

이제 부팅부터 실행까지 한 줄기로 꿰면 이렇습니다.

1. OS가 `python`을 exec → 런타임 초기화(1단계 core, 2단계 main), `sys.path`와 site-packages 확정
2. 타깃이 파일이므로 `pymain_run_file`이 `app.py`를 엽니다.
3. `__main__` 모듈 객체를 만들고 그 `__dict__`를 준비합니다. `__name__='__main__'`, `__file__`, `__builtins__`를 세팅합니다 (그래서 `if __name__ == "__main__":`이 성립합니다).
4. 소스를 tokenize → parse → symtable → compile하여 **모듈용 code 객체**를 얻습니다.
5. `exec(code, __main__.__dict__)`에 해당하는 호출로 frame을 만듭니다. `f_globals = f_locals = __main__.__dict__`, `f_builtins = builtins dict`입니다.
6. eval 루프가 모듈 바이트코드를 위에서 아래로 실행합니다. 여기서 `x = 1`은 `LOAD_CONST 1; STORE_NAME x`로 컴파일되고, `STORE_NAME`이 `f_globals`, 즉 `__main__.__dict__`에 기록합니다. **이것이 "top-level 변수가 모듈 `__dict__`에 들어간다"의 바이트코드 레벨 실체입니다.** `def`/`class`도 객체를 만들어 `STORE_NAME`하는 실행문이고, `import`는 `IMPORT_NAME` opcode로 import 머신을 trigger합니다(찾은 모듈은 `sys.modules`에 캐시되고 `.pyc`로 컴파일 캐시됩니다).
7. 모듈 frame이 반환되면 프로그램이 끝나고 `Py_FinalizeEx`가 atexit과 GC를 돌리고 정리합니다.

![python app.py 실행부터 런타임 초기화, 컴파일, frame 생성, eval 루프, 정리까지 이어지는 부팅-실행 세로 흐름도](diagram-02-boot-to-execution.svg)

## 직접 들여다보기

이 모든 단계는 표준 라이브러리로 관찰할 수 있으므로, 구현을 시도하기 전에 실제 산출물을 확인하는 것이 가장 빠른 학습법입니다.

```python
import ast, symtable, dis
src = "x = 1\ndef f():\n    return x"

print(ast.dump(ast.parse(src)))          # 컴파일 2단계: AST 구조 확인
print(symtable.symtable(src, "<s>", "exec").get_symbols())  # symtable의 local/global 분류
code = compile(src, "<s>", "exec")        # 컴파일 4단계: code 객체 생성
print(code.co_names, code.co_consts)      # code 객체 내부 필드
dis.dis(code)                             # STORE_NAME / LOAD_GLOBAL 등 실제 바이트코드
```

`dis.dis`로 모듈 본문은 `STORE_NAME`, 함수 본문은 `LOAD_FAST`/`LOAD_GLOBAL`이 나오는 것을 눈으로 확인하면, 컴파일·실행 단계의 이분법이 한 번에 체감됩니다. 더 깊게는 `types.CodeType`으로 code 객체를 직접 합성하거나 `sys.modules`/`sys.meta_path`를 조작해 import 머신을 가지고 놀 수 있습니다.

## 마치며

`python app.py` 한 줄 뒤에는 네 단계가 순서대로 흐릅니다. 런타임을 두 단계로 부팅해 상태 객체·빌트인 타입·`sys.path`를 세우고, 소스를 code 객체로 컴파일하고, 거기에 네임스페이스를 묶어 frame을 만들고, stack machine eval 루프가 바이트코드를 한 opcode씩 실행합니다.

이 흐름을 한번 꿰고 나면 평소 헷갈리던 동작들이 같은 원리의 다른 얼굴로 보입니다. 변수가 local이냐 global이냐는 symtable이 컴파일 시점에 확정하고, 최상위 변수가 모듈 `__dict__`에 들어가는 것은 `STORE_NAME`이 `f_globals`에 기록하기 때문이며, `print`가 어디서나 동작하는 것은 모든 frame의 `f_builtins`가 같은 빌트인 dict를 가리키기 때문입니다. 추상적으로 외우던 규칙이 특정 opcode와 자료구조의 동작으로 환원되는 셈입니다.

같은 "런타임 객체와 네임스페이스가 무엇으로 이뤄져 있는가"라는 관점이 실전에서 어떻게 발목을 잡는지는, 파이썬 객체가 dict로 분해되는 또 다른 사례를 다룬 [LangGraph에서 Pydantic State를 invoke했더니 dict가 돌아왔다]({% post_url 2026-06-08-langgraph-pydantic-state-returns-dict %})에서도 이어집니다.

## 참고

- [PEP 587 — Python Initialization Configuration](https://peps.python.org/pep-0587/) (3.8+ 2단계 초기화)
- [PEP 659 — Specializing Adaptive Interpreter](https://peps.python.org/pep-0659/) (3.11 adaptive interpreter)
- [The Python Language Reference — Execution model](https://docs.python.org/3/reference/executionmodel.html) (이름 바인딩과 scope)
- [The Python Language Reference — The import system](https://docs.python.org/3/reference/import.html) (`sys.modules` 캐시와 로딩 순서)
- [PEP 405 — Python Virtual Environments](https://peps.python.org/pep-0405/) (`pyvenv.cfg`·`sys.prefix` 메커니즘)
- [`dis` — Disassembler for Python bytecode](https://docs.python.org/3/library/dis.html)
- [`symtable` — Access to the compiler's symbol tables](https://docs.python.org/3/library/symtable.html)
