---
layout: post
title: Clearing Exclusive Monitor from Exception Return on ARM Linux
---

# Clearing Exclusive Monitor from Exception Return on ARM Linux

---

## Contents

1. [Introduction][section-introduction]  
2. [Synchronization Primitives][section-synchronization-primitives]
3. [Local Monitor][section-local-monitor]
4. [Global Monitor][section-global-monotor]
5. [Usage Restrictions][section-usage-restrictions]
6. [Clear Exclusive Monitor from Exception Returns][section-clearex]
7. [Simple Store for atomic_set()][section-atomic-set]

---

## 1. Introduction [section-introduction]

ARM 동기화 명령어인 ldrex/strex/clrex 중 clrex 를 명시적으로 수행해야 하는 이유와 clrex가 삽입된 코드 위치가 가지는 의미에 대해서 알아본다. LL/SC와 같은 기본적인 내용은 알고 있다고 가정한다.

>NOTE  
>ARMv7-AR Reference Manual 참고  
> A3.4 Synchronization and semaphores
> A3.4.1 Exclusive access instructions and Non-shareable memory regions  
> A3.4.2 Exclusive access instructions and Shareable memory regions  
> A3.4.4 Context switch support  
> A3.4.5 Load-Exclusive and Store-Exclusive usage restrictions  
>
> git commit 200b812d0084f800bc52465e273b118ff5f8141f
>
> http://comments.gmane.org/gmane.linux.ports.arm.kernel/64613

## 2. Synchronization Primitives [section-synchronization-primitives]

| Load Exclusive | Store Exclusive | Clear Exclusive |
|----------------|-----------------|-----------------|
| ldrex[b/d/h]   | strex[b/d/h]    | clrex           |

>NOTE  
>A3.4 Synchronization and semaphores

## 3. Local Monitor [section-local-monitor]

로컬 모니터는 특정 하나의 프로세서에만 관련된다. 단 예외/문맥전환에 의해 다른
스레드흐름이 끼어들 수 있다. (역시 하나의 프로세서라는 사실에는 변함이 없다)

| Initial   | Operation     | Effect                     | Final state       |
|-----------|---------------|----------------------------|-------------------|
| Open      | LoadExcl(x)   | Loads Memory               | Exclusive         |
|           |               | Tags Address x             |                   |
|           |               |                            |                   |
| Exclusive | CLREX         | Clears tagged address t    | Open              |
|           | StoreExcl(t)  | Updates memory             | Open              |
|           | StoreExcl(!t) | Updates or                 | Open              |
|           |               | Does not update memory     |                   |
|           | Store(t)      | Updates memory             | Open or Exclusive |
|           | LoadExcl(x')  | Loads memory and           | Exclusive         |
|           |               | Change tag to x'           |                   |

>NOTE  
> 주의할 경우만 언급되어 있으며 전체표는 다음 참고  
> A3.4.1 Exclusive access instructions and Non-shareable memory regions

* StoreExcl(!t) 는 메모리 업데이트 여부에 관계없이 monitor 의 상태를 Open 으로
  만든다. 이 성질로 인해 clrex 대신 사용될 수도 있다.  
* Store(t)는 메모리를 항상 업데이트하지만 모니터의 상태를 구현에 따라
  (implementation defined ) exclusive 상태로 둘 수 있기 때문에 ldrex/strex 쌍
  사이에 존재하면 문제가 될 수 있다.  

멀티스레드 흐름에서 문제가 될 수 있는 다음의 경우를 생각해 보자.  

    T1          T2
    ldrex(x)            <- 1. x를 태그하고 모니터의 상태를 exclusive 로 변환
    ------------------	<- 2. 예외 혹은 스케쥴링
                str(t)	<- 3. 태그된 주소 t의 내용을 업데이트하고 모니터의
                              상태는 implementation defined
    ------------------	<- 4. 복귀
    strex(t)            <- 5. 3에서 모니터의 결과 상태가 exclusive 이면 strex
                              성공.

다른 스레드 흐름 T2의 str(t)로 인해 메모리가 변경되었으므로 T1 스레드의
strex(t) 는 실패해야함에도 성공하는 결과가 되어버린다.  
이것은 아래와 같이 수정되어야 한다.

    T1          T2
    ldrex(x)            <- 1. x를 태그하고 모니터의 상태를 exclusive 로 변환
    ------------------	<- 2. 예외 혹은 스케쥴링
                str(t)	<- 3. 태그된 주소 t의 내용을 업데이트하고 모니터의
                              상태는 open 혹은 exclusive
                clrex	<- 4. 모니터의 상태를 open 상태로 확정함.
    ------------------	<- 5. 복귀
    strex(x)            <- 6. clrex에 의해 태그된 주소가 지워지고 모니터는
                              open 상태가 되어 strex가 반드시 실패한다.

strex(t) 를 반드시 실패하게 하기 위하여 복귀 직전에 clrex 를 호출하여야 한다.

물론 싱글 스레드 흐름에서 다음의 경우는 없다고 생각한다.
(프로그래밍 오류로 간주한다. 아래와 같은 코드가 정상은 아니다.)

    T1
    ldrex(x)
    str(t)
    strex(t)

## 4. Global Monitor [section-global-monotor]
Global 모니터는 Shareable memory regions에 대해서 CPU n의 Local 모니터에 의해 허가된 strex(t)이 실제 메모리를 업데이트할 것인지 여부를 결정하기 위해 추가적으로 검사된다.

| Initial state | Operation       | Effect                            | Final state       |
|---------------|-----------------|-----------------------------------|-------------------|
| Open          | LoadExcl(x,n)   | Loads memory, tags address x      | Exclusive         |
|               |                 |                                   |                   |
| Exclusive     | ClearExcl(n)    | None                              | Open or Exclusive |
|               | StoreExcl(t,n)  | Updates memory                    | Open or Exclusive |
|               | StoreExcl(t,!n) | Updates memory                    | Open              |
|               |                 | Does not update memory            | Exclusive         |
|               | Store(t,n)      | Updates memory                    | Open or Exclusive |
|               | Store(t,!n)     | Updates memory                    | Open              |

- `LoadExcl/Store[Excl](?,!n)` 은 어떤 cpu n에서 `Load/StoreExcl(?,n)` 이 실행될 때 해당 cpu를 제외한 다른 cpu !n 들의 global exclusive monitor 의 상태 변화를 나타낸다.
- StoreExcl(t,!n)은 위에서 설명된 바와 같이 CPU n에서 StoreExcl(t,n) 이 실행될때 해당 명령어가 메모리를 업데이트하는지의 여부에 따라 변하는 CPU !n의 글로벌 모니터의 상태를 나타낸다. 즉 CPU n 이 메모리를 업데이트 했다면 다른 모든 CPU들의 글로벌 모니터의 상태는 open 이 되고 메모리를 업데이트 하지 않았다면 exclusive 로 남는다.
- Store(t, n)은 메모리 내용을 항상 업데이트하지만 구현에 따라 모니터의 상태를 exclusive 로 그냥 둘수도 open 상태로 만들 수 도있다.(로컬 모니터의 경우와 같다.)
- Store(t, !n) 는 메모리 내용을 항상 업데이트하며 다른 CPU들의 글로벌 모니터 상태를 항상 open 으로 만든다. (즉 다른 모든 cpu !n 들의 global monitor 상태를 open으로 바꾼다.  

다음의 간단한 예를 한번 보자.

	CPU1		CPU2
	ldrex(x)			<- 1. x를 태그하고 CPU 1의 로컬 및 글로벌 모니터의 상태를 exclusive 로 변환
				str(t)	<- 2. CPU2가 태그된 주소 t의 내용을 업데이트하고 CPU1의 글로벌 모니터의 상태는 open
	strex(t)			<- 3. CPU1의 로컬모니터는 exclusive, 글로벌모니터는 2.에서 open이 되어 strex 실패

## 5. Usage Restrictions [section-usage-restrictions]

- 문맥전환이나 예외처리와 같은 싱글스레드흐름을 바꿀 수 있는 상황에서는 clrex를 사용하여 로컬 모니터가 open 상태임을 확실히 해야 한다. (결국 이거 하나 이해할려고 이 지랄을 하고 있는 거라고...) 즉 다음의 결과를 예측할 수 없다.

	T1			T2
	ldrex(x1)				1. 주소 x1을 t로 태그
				ldrex(x2)	2. 태그 t를 x2로 갱신.
	strex(!t)				3. 태그되 주소 t에 대한 strex가 아님. 결과를 예측할 수 없음.
				strex(t)	4. 알수가 없다고.

다음과 같이 처리하여야 한다.

	T1			T2
	ldrex(x1)				1. 주소 x1을 t로 태그, 모니터 exclusive
	clrex					2. 태그 t를 지움, 모니터 open.
				ldrex(x2)	3. 주소 x2를 t로 태그.
				clrex		4. 태그 t를 지움, 모니터 open.
	strex(!t)				5. 모니터가 open 상태이므로 쓰기 실패.
	clrex					6. 태그 t를 지움, 모니터 open.
				strex(t)	7. 모니터가 open 상태이므로 쓰기 실패.

- Taking Data Abort exception.

>NOTE  
>중요사항만 언급됨.  
>전문은 A3.4.5 Load-Exclusive and Store-Exclusive usage restrictions 참고

## 6. Clear Exclusive Monitor from Exception Returns [section-clearex]
data abort 를 제외하고 clrex가 호출되는 지점은 다음과 같다.

	arch/arm/kernel/entry-header.S:
	
	.macro  svc_exit, rpsr, irq = 0
	...
	clrex						@ 모니터 클리어
	ldmia   sp, {r0 - pc}^		@ 예외 복귀
	
	...
	
	.macro  restore_user_regs, fast = 0, offset = 0
	...
	clrex				@ 모니터 클리어.
	movs    pc, lr		@ 예외복귀

해당 매크로들은 각각 예외 발생 이전의 상태(kernel 혹은 user 모드)에 따라 예외 복귀시 이전 문맥을 복원하기 위해 호출되는데 `svc_exit` 과 `restore_user_regs` 심볼을 검색해보면 각 모드에 따라 예외 처리의 가장 마지막에 호출됨을 알 수 있다. 예외에서 원래 모드로의 복귀전에 스케쥴러가 호출되어 현재 프로세스가 선점될 수 있는데 커널모드에서 자발적인 스케쥴러호출을 제외하고는 다음의 경우만이 `_TIF_NEED_RESCHED` 의 설정여부에 따라 스케쥴러를 호출한다.

	1. irq 처리 후 커널모드로의 복귀 직전 (선점모드)
	2. und/dabt/pabt/swi/irq 처리 후 유저모드로의 복귀 직전

예외 복귀 직전의 시점에 clrex를 하는 것으로 `switch_to()` 에서 clrex를 빼도 되는 이유는 예외 복귀 직전 시점이 바로 스캐쥴러가 호출되어 이전 프로세스의 문맥이 끊긴 지점이며 실행중이던 다른 프로세스가 선점되어 이전의 프로세스가 실행 재개된다면 바로 이곳으로 부터 실행을 계속 하기 때문이다. 엄연히 따지면 지금까지 대충 표현한것과는 약간 달리 clrex가 호출되는 시점은 정확히 다음과 같다.

	T1			T2			T3
	ldrex(x)						     T1 continues.
	--------------------------------- <- T1 enters exception and schedule
				ldrex(x)					   T2 continues.
	--------------------------------- 		<- T2 enters exception and schedule
	clrex								 T1 restored, in exception now.
	--------------------------------- <- T1 returns from exception
	strex(!t)							 T1 continues.
	--------------------------------- <- T1 enters exception and schedule
							ldrex(x)				   T3 continues.
	--------------------------------- 				<- T3 enters exception and schedule 
				clrex						   T2 restored, in exception now.
	---------------------------------		<- T2 returns from exception
				strex(t)					   T2 continues.

다음과 같은 ldrex/strex 쌍 사이에 스케쥴러를 호출하는 경우는 프로그래밍 오류로 간주한다.

	ldrex(x)
	schedule()
	strex(t)

위의 흐름을 보다 자세히 나타내면 다음과 같다.

	1. T1 running.
	2. exception occurs, T1 enters an exception.
	3. saves T1's previous context.
	4. handles the exception.
	5. checks if `_TIF_NEED_RESCHED` is on and call `schedule()` conditionally.
	6. T2 process continues in an exception mode.
       (in case of calling `schedule()` in 5)
	7. T2 returns from exception mode.
	8. T2 continues to run.
	9. exception occurs, T2 enters an exception.
	10. saves T2's previous context.
	11. handles the exception.
	12. checks if `_TIF_NEED_RESCHED` is on and call `schedule()` conditionally.
	13. T1 process continues in an exception mode.
        (in case of calling `schedule()` in 12)
	14. T1 returns from exception mode.
	15. T1 continues to run.
	16. ......
	17. ......

## 7. Simple Store for atomic_set() [section-atomic-set]

- `atomic_set()` 을 단순 store로 변경한 커밋.  
	- git commit 200b812d0084f800bc52465e273b118ff5f8141f  
- discussion threads
	- http://comments.gmane.org/gmane.linux.ports.arm.kernel/64613  

위에 명시된 스레드에서 문제를 삼은 경우는 결국 다음과 같은데 이 경우 __switch_to 가 호출되지 않으므로(결과적으로 clrex가 호출되지 않으므로) 마지막 strex가 오류로 성공하게 되는 경우이며 이 때문에 irq 복귀에서 clrex가 필요하게 된다. 시그널 핸들러의 경우 또한 같은 문제가 발생할 수 있으며 이에 모든 예외복귀경로에서 clrex를 해야만 한다는 것이 결론이다. 예외복귀마다 clrex를 하게 되면 `__switch_to()` 에서 clrex 는 결국 필요없어지기 때문에 (예외복귀구간이 스케쥴러 호출가능 구간인점을 상기해라.) 빠지게 되었고 예외복귀마다 clrex하는 효과로 인해 `atomic_set()` 이 단순 store 명령으로 대체되었다.

	T1			I2 (interrupt)
	ldrex(x)
				ldrex(x)	<- 1. x를 태그하고 모니터의 상태를 exclusive로 변환
				strexeq(t)	<- 2. eq 조건만족하여 메모리를 업데이트, 모니터 상태를 open으로 변환
				ldrex(x)	<- 3. x를 태그하고 모니터의 상태를 exclusive로 변환
				strexeq(t)	<- 4. eq 조건을 불만족 메모리를 업데이트하지 않음, 모니터 상태는 exclusive.
								  여기서 모니터의 상태가 exclusive로 남게 됨으로써 2.의 메모리 업데이트가
								  5.번에게 가려지게 된다. 
	strex(t)				<- 5. strex가 메모리를 업데이트하게 되는 오류가 생겼다.

또한 다음의 문제 때문에

	T1			T2(예외)
	ldrex(x)			<- 1. x를 태그하고 모니터의 상태를 exclusive 로 변환
	------------------	<- 2. 예외 혹은 스케쥴링
				str(t)	<- 3. 태그된 주소 t의 내용을 업데이트하고 모니터의 상태는 implementation defined
	------------------	<- 4. 복귀
	strex(t)			<- 5. 실패해야함에도 3.에서 모니터의 결과 상태가 exclusive 이면 strex 성공.

ldrex/strex 쌍으로 구현되어 있었던 `atomic_set()` 은

	T1			E1(예외)
	ldrex(x)			<- 1. x를 태그하고 모니터의 상태를 exclusive 로 변환
	------------------	<- 2. 예외 혹은 스케쥴링
				ldrex(x)
				strex(t)<- 3. 메모리가 업데이트되고 로컬모니터의 상태는 open
	------------------	<- 4. 복귀
	strex(t)			<- 5. 3에서 모니터의 결과 상태가 open이므로 strex 실패 (문제없음).

예외복귀시의 clrex 호출로 인해 단순 store 명령으로 구현 가능해졌다.

	T1			T2(예외)
	ldrex(x)			<- 1. x를 태그하고 모니터의 상태를 exclusive 로 변환
	------------------	<- 2. 예외 혹은 스케쥴링
				str(t)	<- 3. 태그된 주소 t의 내용을 업데이트하고 모니터의 상태는 open 혹은 exclusive
	------------------	<- 4. 복귀
	clrex				<- 5. 모니터의 상태를 open 상태로 확정함.
	strex(x)			<- 6. 5.에 의해 모니터는 open 상태가 되어 strex가 반드시 실패한다 (문제없음).
