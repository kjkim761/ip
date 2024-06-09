Wiki

# Design
```
xv6는 physical memory 최대 2G를 사용하며 4G virtual memory에서 high address에 mapping됩니다. 이 부분은 page table entry에서 PTE_U bit가 0이라서 kernel mode에서만 사용할 수 있고 kernel virtual memory라고 부를 수 있습니다. 2G짜리 low address 부분은 user virtual memory라고 할 수 있습니다. user virtual memory 부분에는 user stack, user text, user heap 영역 등이 할당되게 되는데 physical memory 관점에서 볼 때는 kernel virtual memory와 user virtual memory 두 부분에 모두 mapping되는 경우가 생기게 됩니다.

mmu.h 파일에 나와 있는 것처럼 virtual address는 이렇게 구성됩니다.

// A virtual address 'la' has a three-part structure as follows:
//
// +--------10------+-------10-------+---------12----------+
// | Page Directory |   Page Table   | Offset within Page  |
// |      Index    	 |      Index     |                     |
// +----------------+----------------+---------------------+
//  \--- PDX(va) --/ \--- PTX(va) --/

page table이 2 level 구조입니다.

page directory entry와 page table entry들은 하위 12 bit를 활용해서 flag를 저장합니다. 주소값이 page size의 배수이기 때문에 하위 12bit는 0으로 생각하면 상위 20 bit만으로 page의 주소를 나타낼 수 있고 남는 bit들은 다른 목적으로 활용할 수 있는 것입니다.

copy on write를 구현하기 위해 physical page들에 reference counter를 저장할 수 있는 자료구조를 사용할 것입니다. CoW fork를 하여 추가 프로세스(예: 자식 프로세스)가 이미 다른 프로세스에 존재하는 페이지를 가리킬 때 counter값을 증가시킵니다. 프로세스가 더 이상 페이지를 가리키지 않으면 참조 횟수를 감소시킵니다. 이런 방식을 통해 페이지의 참조 횟수가 0일 때에만 페이지를 free할 수 있게 만들 수가 있습니다. 또한 on write에서 copy가 일어나야 하므로 pte의 flag부분을 적절하게 조절하여서 page fault trap이 발생했을 때 상황에 따라 copy가 일어날 수 있도록 만들 것입니다. 이제 구체적인 implementation을 서술하겠습니다.
```
# Implement
```
먼저 kalloc.c 파일 26번째 줄에 다음과 같이 reference count를 위한 배열인 refcount를 만들었습니다.

int refcount[PHYSTOP>>12] = {0,};

fork가 일어날 때 copyuvm이 호출되게 되는데 이 때 memory가 실제로 할당되는 대신에 copy해야 되는 페이지들을 page table이 가리키게 하고 refcount를 증가시키도록 다음과 같이 vm.c 파일에 있는 copyuvm 함수를 수정하였습니다.

pde_t*
copyuvm(pde_t *pgdir, uint sz)
{
  pde_t *d;
  pte_t *pte;
  uint pa, i, flags;

  if((d = setupkvm()) == 0)
    return 0;
  for(i = 0; i < sz; i += PGSIZE){
    if((pte = walkpgdir(pgdir, (void *) i, 0)) == 0)
      panic("copyuvm: pte should exist");
    if(!(*pte & PTE_P))
      panic("copyuvm: page not present");
    *pte &= (~PTE_W);
    pa = PTE_ADDR(*pte);
    flags = PTE_FLAGS(*pte);
    // if((mem = kalloc()) == 0)
    //   goto bad;
    // memmove(mem, (char*)P2V(pa), PGSIZE);
    // if(mappages(d, (void*)i, PGSIZE, V2P(mem), flags) < 0) {
    //   kfree(mem);
    //   goto bad;
    // }
    mappages(d, (void *)i, PGSIZE, pa, flags);
    incr_refc(pa);
  }
  lcr3(V2P(pgdir));
  return d;
}

여기서 PTE_ADDR가 바로 pte에서 상위 20 bit를 이용해 page address를 얻어내는 함수입니다. PTE_FLAGS는 반대로 page table entry의 하위 12bit를 활용하여 저장된 flags를 얻어내는 함수입니다.

kalloc.c 파일에서 refcount를 모두 0으로 초기화하는 부분을 freerange함수에 추가했습니다. 정의할 때 초기화를 해서 이 부분이 없어도 동작할 것으로 생각되지만 혹시 몰라서 추가했습니다.

또한 다음과 같이 kalloc 함수에 refcount를 1로 설정하는 코드를 추가했습니다. free page가 어떤 프로세스에 의해 할당될 때 1로 설정해야 하기 때문입니다.

char*
kalloc(void)
{
  struct run *r;

  if(kmem.use_lock)
    acquire(&kmem.lock);
  r = kmem.freelist;
  if(r){
    refcount[V2P(r)>>12] = 1;
    kmem.freelist = r->next;
  }
  if(kmem.use_lock)
    release(&kmem.lock);
  return (char*)r;
}

kfree 함수에서는 다음과 같이 refcount값이 0이하일 때만 free하여 freelist에 추가될 수 있도록 수정했습니다.

void
kfree(char *v)
{
  struct run *r;

  if((uint)v % PGSIZE || v < end || V2P(v) >= PHYSTOP)
    panic("kfree");

  if(kmem.use_lock)
    acquire(&kmem.lock);
  refcount[V2P(v)>>12]--;
  if(refcount[V2P(v)>>12]<=0){
    // Fill with junk to catch dangling refs.
    memset(v, 1, PGSIZE);
    r = (struct run*)v;
    r->next = kmem.freelist;
    kmem.freelist = r;
  }

  if(kmem.use_lock)
    release(&kmem.lock);
}

이제까지 설명한 부분들은 이미 존재하는 함수를 조금 수정하는 정도였는데 지금부터 설명할 함수들은 새롭게 정의한 함수들입니다.

void incr_refc(uint p){
  acquire(&kmem.lock);
  refcount[p>>12]++;
  release(&kmem.lock);
}

void decr_refc(uint p){
  acquire(&kmem.lock);
  refcount[p>>12]--;
  release(&kmem.lock);
}

int get_refc(uint p){
  acquire(&kmem.lock);
  int refc = refcount[p>>12];
  release(&kmem.lock);
  return refc;
}

이렇게 kalloc.c 파일에 incr_refc, decr_refc, get_refc 함수를 정의했고 각각 해당되는 페이지의 refcount를 증가, 감소, return하는 함수입니다. 공유자원을 건드릴 때는 필요에 따라 적절히 locking을 해주는 것도 잊으면 안 됩니다.

page fault trap이 발생했을 때 적절히 handle할 수 있도록 다음과 같이 trap.c 파일 80번째 줄에 추가했습니다.

case T_PGFLT:
	CoW_handler();
	break;

여기서 CoW_handler는 vm.c 파일에 정의한 함수로 다음과 같습니다.

void CoW_handler(void){
  uint pgflt_addr = rcr2();
  uint va = PGROUNDDOWN(pgflt_addr);
  pte_t *pte = walkpgdir(myproc()->pgdir, (void*)va, 0);
  if(pte==0){
    panic("wrong range");
  }
  uint pa = PTE_ADDR(*pte);
  int refc = get_refc(pa);
  if(refc>1){
    char *mem = kalloc();
    memmove(mem, (char *)va, PGSIZE);
    *pte = V2P(mem)|PTE_P|PTE_W|PTE_U;
    decr_refc(pa);
  }else{
    *pte |= PTE_W;
  }
  lcr3(V2P(myproc()->pgdir));
}

write권한이 없는 페이지에 write를 시도하면 이 handler가 새로 페이지를 할당받고 pte가 해당 페이지를 가리키도록 수정해줍니다. page를 복사하고 나서는 원래 페이지의 refcount는 감소시켜줘야 하므로 decr_refc 함수를 이용하여 감소시켰습니다. 또한 아까 설명한 get_refc 함수를 통해 refcount값을 알아내고 해당 값이 1 이하인 경우에는 페이지를 가리키는 유일한 프로세스인 것이므로 copy를 할 필요가 없고 단순히 write권한을 부여하여 원래 페이지를 사용하도록 해줬습니다. 페이지 테이블 항목을 변경할 때는 TLB를 flush하여 동기화가 깨지지 않도록 해야합니다. 또한 가상 주소가 프로세스의 페이지 테이블에 매핑되지 않은 잘못된 범위에 속해 있다면 에러 메시지를 출력하고 프로세스를 종료하도록 패닉을 걸었습니다. 

countvp, countpp, countptp 함수는 vm.c 파일에 다음과 같이 정의했습니다. xv6에서는 적층식으로 user virtual memory를 관리하며 proc->sz는 할당된 메모리 중 가장 높은 메모리의 위치에 맞춰서 설정이 되므로 countvp에서 logical page개수를 다음과 같이 구할 수 있었습니다. countpp는 demand paging을 사용하지 않는 xv6에서 countvp와 동일한 값을 반환하지만 이번에는 직접적으로 프로세스의 페이지 테이블을 탐색하여 구하는 방식으로 구현했습니다. countptp는 pde들의 flag를 읽어서 유효한 페이지 테이블인지 파악하여 개수를 셌고 페이지 디렉토리에 사용된 페이지도 빠트리지 않았습니다.

int countvp(void){
  uint countvp = (myproc()->sz)>>12;
  return countvp;
}

int countpp(void){
  pde_t *pgdir = myproc()->pgdir;
  int countpp = 0;
  for(int i = 0; i<512; i++){

    if(pgdir[i]&PTE_P){
      pte_t *pte = (pde_t *)P2V(PTE_ADDR(pgdir[i]));
      for(int j = 0; j<1024; j++){
        if(pte[j]&PTE_P){
          countpp++;
        }
      }
    }
  }
  return countpp;
}

int countptp(void){
  int countptp = 0;
  pde_t *pgdir = myproc()->pgdir;
  for(int i = 0; i<1024; i++){
    if(pgdir[i]&PTE_P){
      countptp++;
    }
  }
  return countptp+1;
}

countfp는 다음과 같이 kalloc.c에 구현했습니다.

int countfp(void){
  struct run *r;
  int countfp = 0;
  if(kmem.use_lock)
    acquire(&kmem.lock);
  for(r = kmem.freelist; r!=0; r = r->next){
    countfp++;
  }
  if(kmem.use_lock)
    release(&kmem.lock);
  return countfp;
}

free page 개수를 저장하는 별도의 전역변수는 필요 없이 kmem.freelist를 이용해서 순회해서 카운트했습니다.

kalloc.c 파일과 vm.c 파일에 정의한 함수들을 다른 파일에서도 사용할 수 있도록 다음과 같은 코드를 defs.h 파일에 추가했습니다.

int             countfp(void);
void            incr_refc(uint);
void            decr_refc(uint);
int             get_refc(uint);

void            CoW_handler(void);
int             countvp(void);
int             countpp(void);
int             countptp(void);

system call 들을 구현하기 위한 코드 부분들을 설명하겠습니다. 다음과
같이 sysproc.c 파일에 4개의 wrapper function 을 만들었습니다. 

int
sys_countfp(void){
  return countfp();
}

int 
sys_countvp(void){
  return countvp();
}

int 
sys_countpp(void){
  return countpp();
}

int 
sys_countptp(void){
  return countptp();
}

system call 등록을 위해 syscall.h 파일 23 번쨰 줄부터 다음과 같이 추가하고,

#define SYS_countfp 22
#define SYS_countvp 23
#define SYS_countpp 24
#define SYS_countptp 25

syscall.c 파일 106 번째 줄부터 다음과 같이 추가하고,

extern int sys_countfp(void);
extern int sys_countvp(void);
extern int sys_countpp(void);
extern int sys_countptp(void);

133번째 줄에 다음과 같이 추가했습니다.

[SYS_countfp] sys_countfp,
[SYS_countvp] sys_countvp,
[SYS_countpp] sys_countpp,
[SYS_countptp] sys_countptp,

또한 user program에서 system call 을 사용할 수 있도록 usys.S 파일 32 번째 줄에 다음과 같이
추가해주고,

SYSCALL(countfp)
SYSCALL(countvp)
SYSCALL(countpp)
SYSCALL(countptp)

user.h 파일 26 번째 줄에 다음과 같이 추가해줬습니다.

int countfp(void);
int countvp(void);
int countpp(void);
int countptp(void);
```
# Result
```
컴파일 및 실행을 위해 Makefile 에 test0, test1, test2, test3를 추가해줬습니다.

이제 make clean, make, make fs.img 명령을 차례로 입력하여 간단하게 컴파일 할 수 있습니다.
bootxv6.sh 을 통해 부팅하고 명세에서 요구한 대로 정확히 동작하는지 확인해 보겠습니다.

SeaBIOS (version 1.15.0-1)


iPXE (https://ipxe.org) 00:03.0 CA00 PCI2.10 PnP PMM+1FF8B4A0+1FECB4A0 CA00
                                                                               


Booting from Hard Disk..xv6...
cpu0: starting 0
sb: size 1000 nblocks 941 ninodes 200 nlog 30 logstart 2 inodestart 32 bmap start 58
init: starting sh
$ test1    
[Test 1] initial sharing
[Test 1] pass

$ test2
[Test 2] Make a Copy
[Test 2] pass

$ test3
[Test 3] Make Copies
child [0]'s result: 1
child [1]'s result: 1
child [2]'s result: 1
child [3]'s result: 1
child [4]'s result: 1
child [5]'s result: 1
child [6]'s result: 1
child [7]'s result: 1
child [8]'s result: 1
child [9]'s result: 1
[Test 3] pass

$ test0
[Test 0] default
ptp: 66 66
[Test 0] pass

$

실행 결과들에서 정확히 동작하는 것을 확인할 수 있습니다. test0에서는 countfp, countvp, countpp, countptp 모두 잘 실행되고 sbrk도 정상적으로 수행되었습니다. test1에서는 thread_test 의 test 1 에서는 fork를 통해 프로세스를 생성하고 자식 프로세스와 부모 프로세스가 같은 물리 페이지를 가리키고 있는지를 확인할 수 있습니다. test2에서는 자식 프로세스가 부모 프로세스와 공유하고 있는 변수를 수정하여 새로운 물리 페이지를 할당 받는지 확인할 수 있습니다. test3에서는 10개의 자식 프로세스를 생성하고, 각각의 자식 프로세스는 test2와 유사하에 부모 프로세스와 공유하는 변수를 수정합니다. 자식 프로세스의 종료 이후, free page의 적절한 회수가 이루어졌는지까지 확인할 수 있습니다.

저는 이러한 과정을 통해서 어떤 오류도 없이 명세의 요구를 완벽히 충족할 수 있었습니다.
```
# Trouble shooting
```
P2V 적용하는 거 딱 한 군데를 빠트려서 주말을 모두 날렸습니다.
```
# Comments
```
정말 오랜 시간이 걸렸고 하면서 답답함과 스트레스도 극한으로 받았지만 다 완료하니까 뿌듯했고 과정도 재밌었던 것 같습니다! 심심할 때 CoW에 의한 성능 변화를 측정하는 개인적인 심화 프로젝트도 해보고 싶습니다.
```
