# analyze_mysql_source_code
mysql 8.0 source code analysis


1. storage/innobase/buf/buf0buf.cc


버퍼풀의 메인 구현 파일
버퍼풀 초기화, 메모리 할당, 페이지 관리 등 핵심 기능 구현


2. storage/innobase/include/buf0buf.h


버퍼풀 관련 구조체와 함수 선언
버퍼 페이지, 버퍼풀 인스턴스 등의 정의


3. storage/innobase/buf/buf0flu.cc


4. 버퍼풀 플러시(flush) 관련 구현
더티 페이지 쓰기 처리


5. storage/innobase/buf/buf0lru.cc


LRU 리스트 관리


--------------------------------------------------------------------------------------------------------


## 주요 구조체와 함수들:

// buf0buf.h
struct buf_pool_t {    // 버퍼풀 인스턴스
  // ...
  buf_chunk_t *chunks; // 메모리 청크 배열
  hash_table_t *page_hash;  // 페이지 해시 테이블
  UT_LIST_BASE_NODE_T(buf_page_t) free;  // 프리 리스트
  UT_LIST_BASE_NODE_T(buf_page_t) LRU;   // LRU 리스트
  // ...
};

// buf0buf.cc
buf_pool_t *buf_pool_init(
  ulint pool_size,     // 버퍼풀 크기
  ulint n_instances)   // 인스턴스 개수
{
  // 버퍼풀 초기화 구현
}

위 파일들에서 실제 메모리 할당은 주로 buf_chunk_t 단위로 이루어지며, 이는 운영체제의 메모리 할당 함수를 통해 구현됩니다.




## MySQL InnoDB의 버퍼 풀 관련 구현이 담긴 buf0buf.cc 파일

주요 내용을 요약하면:

buf_pool_create() 함수는 버퍼 풀 인스턴스를 초기화하는 핵심 함수입니다.


이 함수에서는:


CPU affinity 설정 (Linux 환경)
버퍼 풀 청크(chunk) 할당
해시 테이블 초기화
LRU 리스트, Free 리스트 등 초기화
플러시(flush) 관련 mutex 초기화
압축 페이지 관리를 위한 자료구조 초기화


buf_pool_ptr은 전역 버퍼 풀 인스턴스 배열을 가리키는 포인터입니다.
버퍼 풀의 크기나 청크 크기 등은 서버 파라미터로 설정 가능합니다.


---------------------------------------------------------------------------------------------

## InnoDB 버퍼 풀 소스코드를 분석하면 다음과 같은 중요한 지식과 인사이트를 얻을 수 있습니다:

# 메모리 관리 기법


대규모 메모리를 효율적으로 관리하는 방법 학습
버퍼 풀을 청크 단위로 나누어 관리하는 기법
LRU(Least Recently Used) 알고리즘 구현 방법
메모리 할당/해제 최적화 기법


# 동시성 제어


Mutex, Lock을 사용한 멀티스레드 환경에서의 동시성 제어 방법
데이터베이스의 ACID 속성을 보장하기 위한 기법들
읽기/쓰기 락의 구현과 활용


# 성능 최적화 기법


CPU affinity를 통한 캐시 활용 최적화
메모리 압축을 통한 공간 효율성 향상
선제적 읽기(Read Ahead) 구현
더티 페이지 관리와 플러시 최적화


# 대규모 시스템 설계


모듈화된 코드 구조 설계 방법
에러 처리와 복구 메커니즘
메모리와 디스크 간 데이터 동기화 처리
설정 가능한 파라미터를 통한 유연성 확보


# 개발 프로세스


코드 주석 작성법
디버깅을 위한 로깅/모니터링 구현
단위 테스트 방법
성능 측정과 통계 수집


# 프로그래밍 기법


C++ 템플릿 활용
STL 컨테이너 활용
메모리 정렬과 캐시 라인 고려
원자적 연산 사용

이러한 지식들은 다음과 같은 상황에서 실제 활용 가능합니다:

대용량 데이터 처리 시스템 설계
캐시 시스템 구현
메모리 집약적 애플리케이션 개발
멀티스레드 프로그래밍
성능에 민감한 시스템 최적화

실제 프로덕션 레벨의 코드를 분석하면서 이론적 지식이 어떻게 실제 구현되는지 배울 수 있고, 이는 더 나은 소프트웨어를 만드는데 큰 도움이 됩니다.

---------------------------------------------------------------------------------------------------------

## 메모리 관리 관련 코드

 // 버퍼 풀의 메모리를 청크 단위로 할당
buf_chunk_t *new_chunks = reinterpret_cast<buf_chunk_t *>(
    ut::zalloc_withkey(UT_NEW_THIS_FILE_PSI_KEY, new_chunks_size));

// LRU 리스트 관리
UT_LIST_INIT(buf_pool->LRU);
UT_LIST_INIT(buf_pool->free);

## 동시성 제어 관련

 // 다양한 뮤텍스 초기화
mutex_create(LATCH_ID_BUF_POOL_LRU_LIST, &buf_pool->LRU_list_mutex);
mutex_create(LATCH_ID_BUF_POOL_FREE_LIST, &buf_pool->free_list_mutex);
mutex_create(LATCH_ID_BUF_POOL_ZIP_FREE, &buf_pool->zip_free_mutex);

## CPU 최적화 관련

 #ifdef UNIV_LINUX
  cpu_set_t cpuset;
  CPU_ZERO(&cpuset);
  const long n_cores = sysconf(_SC_NPROCESSORS_ONLN);
  CPU_SET(instance_no % n_cores, &cpuset);

## 모니터링/통계 관련

 buf_pool->stat.reset();
buf_pool->last_printout_time = std::chrono::steady_clock::now();
이처럼 한 파일이지만 매우 많은 기법들이 적용되어 있습니다. 그 이유는:

버퍼 풀이 MySQL의 핵심 컴포넌트이기 때문에 성능에 매우 중요
메모리 관리, 동시성 제어, 성능 최적화 등 여러 측면을 모두 고려해야 함
오랜 시간에 걸쳐 많은 개발자들이 지속적으로 개선해온 결과
데이터베이스의 ACID 속성을 보장하기 위해 복잡한 매커니즘이 필요

이런 코드를 이해하기 위해서는:

점진적으로 접근하기
핵심 자료구조와 알고리즘 먼저 이해하기
디버깅을 통해 실제 동작 확인하기
관련 문서와 주석 참고하기


----------------------------------------------------------------------------------------------
## 간단 코드 분석

#ifdef UNIV_LINUX
  cpu_set_t cpuset;

  // cpu_set_t 구조체를 0으로 초기화
  CPU_ZERO(&cpuset);

  // 현재 시스템에서 사용가능한 논리적 core 수를 구한다.
  // 이 값은 버퍼풀 인스턴스를 생성할때 스레드를 할당하는데 사용된다. 
  // 예를들어 총 8개 논리 코어가 있는 시스템에서 버퍼풀 인스턴스를 4개로 지정했을 경우
  // 4개 인스턴스를 생성할 때 스레드를 2개씩 할당할 수 있다.
  const long n_cores = sysconf(_SC_NPROCESSORS_ONLN);

  // instance_no를 core 수로 나눈 나머지를 통해 각 버퍼풀 인스턴스가 사용할 cpu core를 지정한다.
  // 예: instance 0 -> core 0
  //     instance 1 -> core 1
  //     instance 4 -> core 0 (인스턴스가 코어 수보다 많은 경우 다시 처음부터)
  CPU_SET(instance_no % n_cores, &cpuset);

  // 버퍼풀의 통계정보 초기화 
  buf_pool->stat.reset();

  // 현재 스레드를 cpu_set_t 구조체에서 지정된 코어에 바인딩한다.
  // 이렇게 CPU affinity를 설정하면:
  // 1. 스레드가 여러 cpu core를 사용할때 발생할 수 있는 cache miss를 줄일 수 있다.
  // 2. 다른 프로세스나 스레드와 cpu 코어를 공유하지 않아 cpu 사용률이 올라갈 수 있다.
  // 3. 스레드가 다른 코어로 이동하면서 발생하는 context switching 비용을 줄일 수 있다.
  if (pthread_setaffinity_np(pthread_self(), sizeof(cpuset), &cpuset) == -1) {
    ib::error(ER_IB_ERR_SCHED_SETAFFNINITY_FAILED)
        << "sched_setaffinity() failed!";
  }

  /* Linux might be able to set different setting for each thread
  worth to try to set high priority for this thread. */
  // 리눅스에서 가장 높은 우선순위인 -20으로 설정하여 cpu 시간을 더 많이 할당한다.
  // 첫번째 인자: 우선순위를 조정할 대상의 종류(프로세스)
  // 두번째 인자: 대상의 id 
  // 세번째 인자: 우선순위 값(-20 ~ 19, -20이 가장 높은 우선순위)
  setpriority(PRIO_PROCESS, (pid_t)syscall(SYS_gettid), -20);
#endif /* UNIV_LINUX */



이 코드의 최적화 전략을 요약하면:

CPU Affinity 설정


각 버퍼풀 인스턴스를 특정 CPU 코어에 고정
캐시 지역성(Cache Locality) 향상
불필요한 context switching 감소


프로세스 우선순위 최적화


버퍼풀 관련 스레드에 최고 우선순위 부여
CPU 자원을 우선적으로 할당받음

이러한 최적화는 특히 대규모 다중 사용자 환경에서 데이터베이스 성능 향상에 크게 기여합니다.

------------------------------------------------------------------------------------------

## CPU Affinity가 뭘까?

CPU Affinity는 프로세스나 스레드가 특정 CPU 코어에서만 실행되도록 지정하는 기능입니다.
이해를 돕기 위해 예시를 들어보겠습니다:

CPU Affinity가 없는 경우:

Copy시간 순서대로 살펴보면:
t1: Thread1이 CPU1에서 실행 -> cache1에 데이터 로드
t2: Thread1이 CPU2로 이동 -> cache1의 데이터를 cache2로 복사 필요
t3: Thread1이 다시 CPU1로 이동 -> 다시 cache1로 데이터 복사 필요

문제점:
- Cache Miss 발생 증가
- 데이터를 캐시간 복사하는 오버헤드
- Context Switching 비용 증가

CPU Affinity를 설정한 경우:

CopyThread1은 항상 CPU1에서만 실행
- cache1에 한번 로드된 데이터 계속 재사용
- 다른 CPU 캐시로 데이터 복사 불필요
- Context Switching 최소화
MySQL InnoDB에서 CPU Affinity를 사용하는 이유:

버퍼 풀 성능 최적화

 // 8개 CPU 코어가 있고 버퍼풀 인스턴스가 4개인 경우:
instance 0 -> CPU core 0
instance 1 -> CPU core 1
instance 2 -> CPU core 2
instance 3 -> CPU core 3

각 인스턴스는 지정된 코어에서만 실행되어 캐시 효율성 증가

캐시 활용도 향상

 // 버퍼 풀 인스턴스의 데이터가 특정 CPU 캐시에 유지됨
CPU core 0의 L1/L2 cache: instance 0의 데이터
CPU core 1의 L1/L2 cache: instance 1의 데이터
...

NUMA(Non-Uniform Memory Access) 아키텍처 고려

CopyCPU 0 --- Memory 0
CPU 1 --- Memory 1

각 CPU는 자신의 로컬 메모리에 더 빠르게 접근
-> Affinity 설정으로 로컬 메모리 접근 비율 증가
효과:

캐시 히트율 증가
메모리 접근 지연시간 감소
Context Switching 오버헤드 감소
전반적인 시스템 성능 향상


--------------------------------------------------------------------------------------

## 실제 코드분석

if (pthread_setaffinity_np(pthread_self(), sizeof(cpuset), &cpuset) == -1) {
    ib::error(ER_IB_ERR_SCHED_SETAFFNINITY_FAILED)
        << "sched_setaffinity() failed!";
  }
  
  
  
 pthread_setaffinity_np 함수의 파라미터 설명:

 int pthread_setaffinity_np(
    pthread_t thread,           // 대상 스레드
    size_t cpusetsize,         // cpu_set_t 구조체의 크기
    const cpu_set_t *cpuset    // CPU 마스크가 설정된 구조체
);

각 인자 분석:

 pthread_self()       // 현재 실행 중인 스레드의 ID를 반환
sizeof(cpuset)      // cpu_set_t 구조체의 크기
&cpuset            // 앞서 CPU_SET()으로 설정한 CPU 마스크

실제 동작 예시:

 // 8개의 CPU 코어가 있는 시스템에서
cpu_set_t cpuset;
CPU_ZERO(&cpuset);          // cpuset을 0으로 초기화
                           // 00000000

instance_no = 2;           // 2번 인스턴스라고 가정
n_cores = 8;              // 시스템의 총 코어 수

CPU_SET(2 % 8, &cpuset);  // 2번 코어 설정
                          // 00000100
                          
// pthread_setaffinity_np 호출
// -> 이 스레드는 이제 2번 코어에서만 실행됨

에러 처리:

 if (pthread_setaffinity_np(...) == -1) {
    // 설정 실패시 에러 로그 기록
    ib::error(ER_IB_ERR_SCHED_SETAFFNINITY_FAILED)
        << "sched_setaffinity() failed!";
}

함수 실패 가능한 경우들:

 - 권한 부족 (root 권한 필요할 수 있음)
- 잘못된 CPU 번호 지정
- 시스템 리소스 부족
- OS가 CPU affinity를 지원하지 않는 경우

성능상 이점:

 Before:
Thread -> CPU0 -> Cache0
Thread -> CPU1 -> Cache1 (캐시 미스)
Thread -> CPU2 -> Cache2 (캐시 미스)

After (CPU2에 고정):
Thread -> CPU2 -> Cache2
Thread -> CPU2 -> Cache2 (캐시 히트)
Thread -> CPU2 -> Cache2 (캐시 히트)

실제 사용 예시:

 // 버퍼풀 인스턴스별 CPU 할당 예시 (8코어 시스템)
Instance 0 -> CPU0
Instance 1 -> CPU1
Instance 2 -> CPU2
Instance 3 -> CPU3
Instance 4 -> CPU4
Instance 5 -> CPU5
Instance 6 -> CPU6
Instance 7 -> CPU7

// 9번째 인스턴스는 다시 첫 번째 CPU부터
Instance 8 -> CPU0
이렇게 CPU Affinity를 설정함으로써 각 버퍼풀 인스턴스는 지정된 CPU에서만 실행되어 캐시 효율성을 극대화할 수 있습니다.




------------------------------------------------------------------------------

## 락 관련 코드 분석

// 다양한 뮤텍스 초기화
mutex_create(LATCH_ID_BUF_POOL_LRU_LIST, &buf_pool->LRU_list_mutex);
mutex_create(LATCH_ID_BUF_POOL_FREE_LIST, &buf_pool->free_list_mutex);
mutex_create(LATCH_ID_BUF_POOL_ZIP_FREE, &buf_pool->zip_free_mutex);


이 뮤텍스들이 중요한 이유는:

데이터 일관성 보장

여러 스레드가 동시에 버퍼 풀 자료구조를 수정하는 것을 방지
메모리 corruption 방지


성능 최적화

세분화된 뮤텍스로 경합(contention) 감소
각 작업에 필요한 최소한의 lock만 획득


버퍼 풀 관리

페이지 교체
메모리 할당/해제
압축 페이지 관리



이러한 뮤텍스 설계는 InnoDB의 ACID 속성을 보장하면서도 높은 동시성을 달성하는데 핵심적인 역할을 합니다.



## 버퍼 풀의 주요 뮤텍스들과 그 역할

// 1. LRU 리스트 뮤텍스
mutex_create(LATCH_ID_BUF_POOL_LRU_LIST, &buf_pool->LRU_list_mutex);
/*
LRU_list_mutex의 역할:
- LRU(Least Recently Used) 리스트의 접근을 직렬화
- 페이지 교체(replacement) 작업 보호
- young/old 페이지 이동 작업 보호

사용 예시:
mutex_enter(&buf_pool->LRU_list_mutex);
buf_LRU_add_block(block, TRUE);     // LRU 리스트에 새 블록 추가
mutex_exit(&buf_pool->LRU_list_mutex);
*/

// 2. FREE 리스트 뮤텍스
mutex_create(LATCH_ID_BUF_POOL_FREE_LIST, &buf_pool->free_list_mutex);
/*
free_list_mutex의 역할:
- 버퍼 풀의 빈 페이지 리스트 보호
- 새 페이지 할당 작업 직렬화
- 페이지 해제 작업 보호

사용 예시:
mutex_enter(&buf_pool->free_list_mutex);
block = buf_LRU_get_free_block();   // 빈 블록 할당
mutex_exit(&buf_pool->free_list_mutex);
*/

// 3. ZIP FREE 뮤텍스
mutex_create(LATCH_ID_BUF_POOL_ZIP_FREE, &buf_pool->zip_free_mutex);
/*
zip_free_mutex의 역할:
- 압축된 페이지의 메모리 관리 보호
- zip_free[] 배열 접근 직렬화
- 압축 페이지 할당/해제 작업 보호

사용 예시:
mutex_enter(&buf_pool->zip_free_mutex);
buf_buddy_free_compressed();         // 압축된 페이지 메모리 해제
mutex_exit(&buf_pool->zip_free_mutex);
*/

// 뮤텍스 사용시 주의사항:
// 1. 데드락 방지를 위한 Lock 순서 준수
mutex_enter(&buf_pool->LRU_list_mutex);    // 항상 LRU 먼저
mutex_enter(&buf_pool->free_list_mutex);   // 그 다음 free list
// ... 작업 수행
mutex_exit(&buf_pool->free_list_mutex);    // 역순으로 해제
mutex_exit(&buf_pool->LRU_list_mutex);

// 2. 임계 영역 최소화
mutex_enter(&mutex);
// 최소한의 필수 작업만 수행
mutex_exit(&mutex);

// 3. 중첩 뮤텍스 획득 최소화
// 가능한 한 번에 하나의 뮤텍스만 보유



------------------------------------------------------------------------------------------------------

## 청크 할당 코드 분석 


// 청크 할당 코드
buf_chunk_t *new_chunks = reinterpret_cast<buf_chunk_t *>(
    ut::zalloc_withkey(UT_NEW_THIS_FILE_PSI_KEY, new_chunks_size));
	
	
	
버퍼 풀의 청크 할당 부분:

 // 청크 할당 코드
buf_chunk_t *new_chunks = reinterpret_cast<buf_chunk_t *>(
    ut::zalloc_withkey(UT_NEW_THIS_FILE_PSI_KEY, new_chunks_size));
이 코드는:

zalloc_withkey: 0으로 초기화된 메모리를 할당하는 함수
UT_NEW_THIS_FILE_PSI_KEY: 메모리 추적을 위한 식별자
new_chunks_size: 할당할 청크의 전체 크기
reinterpret_cast: 할당된 메모리를 buf_chunk_t 포인터 타입으로 변환

청크 구조는 다음과 같습니다:
 struct buf_chunk_t {
    buf_block_t* blocks;      // 실제 버퍼 페이지들
    size_t size;             // 청크 내 블록 수
    size_t mem_size;         // 청크의 메모리 크기
    ulint offset;            // 버퍼 풀 내에서의 오프셋
};

LRU 리스트 초기화 부분:

 // LRU 리스트와 FREE 리스트 초기화
UT_LIST_INIT(buf_pool->LRU);    // 최근 사용된 페이지 리스트
UT_LIST_INIT(buf_pool->free);   // 사용 가능한 빈 페이지 리스트
이 리스트들의 구조와 용도:
 // LRU(Least Recently Used) 리스트
- 가장 최근에 사용된 페이지가 앞쪽에 위치
- 오래된 페이지는 뒤쪽으로 이동
- 메모리가 부족할 때 뒤쪽의 페이지부터 제거

// FREE 리스트
- 현재 사용되지 않는 빈 페이지들의 리스트
- 새로운 페이지가 필요할 때 이 리스트에서 가져옴
- 페이지가 해제되면 이 리스트로 반환
사용 예시:
 // 새 페이지가 필요할 때
if (!UT_LIST_GET_FIRST(buf_pool->free)) {
    // free 리스트가 비어있으면
    // LRU 리스트 끝에서 페이지를 제거하고 재사용
    block = buf_LRU_get_free_block();
} else {
    // free 리스트에서 블록 가져오기
    block = UT_LIST_GET_FIRST(buf_pool->free);
    UT_LIST_REMOVE(buf_pool->free, block);
}

// 페이지 접근시
UT_LIST_REMOVE(buf_pool->LRU, block);  // 현재 위치에서 제거
UT_LIST_ADD_FIRST(buf_pool->LRU, block); // LRU 리스트 앞으로 이동
이러한 구조를 통해:

메모리를 효율적으로 관리
자주 사용되는 페이지는 메모리에 유지
필요없는 페이지는 자동으로 제거됨
메모리 할당/해제 작업 최소화

이는 데이터베이스의 성능에 매우 중요한 역할을 합니다.
