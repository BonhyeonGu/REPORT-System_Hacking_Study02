---


---

<h1 id="pwnable.krunlink"><a href="http://14-pwnable.kr">14-pwnable.kr</a>:unlink</h1>
<h2 id="분석">2019-08-20 분석</h2>
<pre><code>#include &lt;stdio.h&gt;
#include &lt;stdlib.h&gt;
#include &lt;string.h&gt;
typedef struct tagOBJ{
        struct tagOBJ* fd;
        struct tagOBJ* bk;
        char buf[8];
}OBJ;

void shell(){
        system("/bin/sh");
}

void unlink(OBJ* P){
        OBJ* BK;
        OBJ* FD;
        BK=P-&gt;bk;
        FD=P-&gt;fd;
        FD-&gt;bk=BK;
        BK-&gt;fd=FD;
}
int main(int argc, char* argv[]){
        malloc(1024);
        OBJ* A = (OBJ*)malloc(sizeof(OBJ));
        OBJ* B = (OBJ*)malloc(sizeof(OBJ));
        OBJ* C = (OBJ*)malloc(sizeof(OBJ));

        // double linked list: A &lt;-&gt; B &lt;-&gt; C
        A-&gt;fd = B;
        B-&gt;bk = A;
        B-&gt;fd = C;
        C-&gt;bk = B;

        printf("here is stack address leak: %p\n", &amp;A);
        printf("here is heap address leak: %p\n", A);
        printf("now that you have leaks, get shell!\n");
        // heap overflow!
        gets(A-&gt;buf);

        // exploit this unlink!
        unlink(B);
        return 0;
}
</code></pre>
<p>힙에서 malloc과 free 과정에서 생기는 unlink라는 과정과 유사하게 소스를 구성했다.<br>
A, B, C라는 동적 구조체를 생성한 뒤 실제 힙 과정처럼 A와 B와 C를 링크했다.<br>
그 뒤에 스택 주소와 힙 주소를 출력 한 후 A-&gt;buf를 입력 받고 B를 unlink 한다.</p>
<p>소스 내부에 쉘을 호출하는 함수가 만들어져 있다. A-&gt;buf 를 gets()로 입력 받는 부분에서 BOF하여 DFB 시키면 될 것이다.</p>
<h2 id="dfbdouble-free-bug">2019-08-22 DFB(Double Free Bug)</h2>
<p><strong>malloc 시 특징</strong></p>
<ul>
<li>malloc(4) 로 4byte 를 할당받으면 4byte그 이상의 용량을 소비하게된다. 그 이유는 뒤에 chunk가 붇기 때문이다.</li>
<li>chunk는 prev_size ,size, fd, bk, fd_nextsize, bk_nextsize, flag로 이루어져있다.</li>
<li>prev_size는 이전 chunk가 free되면 flag를 제외한 이전 chunk의 크기가 기록된다.</li>
<li>flag는 PREV_INUSE(이전 chunk가 사용?), IS_MMAPPED(mmap()함수로 할당?), NON_MAIN_ARENA(멀티 쓰레드 환경에서 main이 아닌 쓰레드에서 생성?) 로 이루어져있다.</li>
</ul>
<p><strong>bin 구조</strong></p>
<ul>
<li>Haep의 효율적인 추가, 해제를 위해 bin 구조를 사용해 chunk list를 관리함</li>
<li>small bin &lt; 512 byte &lt;= large bin</li>
<li>fast bin &lt;= 72byte</li>
<li>각 chunk들은 free 후 bin list에 들어가기 전에 unsorted chunk list 를 거친다</li>
<li>메모리 할당 시 unsorted chunk list를 먼저 확인하여 같은 크기의 free된 chunk가 존재한다면 해당 chunk를 재사용 한다.</li>
</ul>
<p><strong>unlink 매크로 함수</strong></p>
<ul>
<li>free된 chunk가 다시 malloc 될 때 bin 리스트에서 그 chunk를 제거하는 작업을 하기 위해 unlink를 실행한다.</li>
<li>어떤 chunk 가 free 될 때 연속 된 위치에 free된 chunk가 있다면 두개를 합치는 작업을 거친다.</li>
<li>free된 chunk가 합쳐질 때 bin list에서 해당 chunk를 없에고 합친 후 넣기 위해 unlink를 실행한다.</li>
</ul>
<p><strong>unlink  취약점</strong></p>
<ul>
<li>Heap Overflow 같은 방법으로 fd와 bk가 조작된 별개의 chunk를 만든 후 연결 시킨후 free과정에 실행 할 수 있다.</li>
<li>glib 2.3.2 이하에서만 작동하는 취약점이다.</li>
</ul>
<h2 id="풀이1">2019-08-23 풀이1</h2>
<p>함수 gets() 때 "AAAAAAAA"를 넣어 관찰해봤다.</p>
<p>EAX  0x91fc578 ◂— ‘AAAAAAAA’</p>
<p>0x91fc578 에 들어감이 확인됬다. 추적해보자.</p>
<p><strong>unlink() 전</strong></p>
<pre><code>pwndbg&gt; x/50x 0x91fc560
0x91fc560:	0x00000000	0x00000000	0x00000000	0x00000021
0x91fc570:	0x091fc590	0x00000000	0x41414141	0x41414141
0x91fc580:	0x00000000	0x00000000	0x00000000	0x00000021
0x91fc590:	0x091fc5b0	0x091fc570	0x00000000	0x00000000
0x91fc5a0:	0x00000000	0x00000000	0x00000000	0x00000021
0x91fc5b0:	0x00000000	0x091fc590	0x00000000	0x00000000
0x91fc5c0:	0x00000000	0x00000000	0x00000000	0x00000411
0x91fc5d0:	0x20776f6e	0x74616874	0x756f7920	0x76616820
0x91fc5e0:	0x656c2065	0x2c736b61	0x74656720	0x65687320
0x91fc5f0:	0x0a216c6c	0x000a340a	0x00000000	0x00000000
</code></pre>
<p><strong>unlink() 후</strong></p>
<pre><code>pwndbg&gt; x/50x 0x91fc560
0x91fc560:	0x00000000	0x00000000	0x00000000	0x00000021
0x91fc570:	0x091fc5b0	0x00000000	0x41414141	0x41414141
0x91fc580:	0x00000000	0x00000000	0x00000000	0x00000021
0x91fc590:	0x091fc5b0	0x091fc570	0x00000000	0x00000000
0x91fc5a0:	0x00000000	0x00000000	0x00000000	0x00000021
0x91fc5b0:	0x00000000	0x091fc570	0x00000000	0x00000000
0x91fc5c0:	0x00000000	0x00000000	0x00000000	0x00000411
0x91fc5d0:	0x20776f6e	0x74616874	0x756f7920	0x76616820
0x91fc5e0:	0x656c2065	0x2c736b61	0x74656720	0x65687320
0x91fc5f0:	0x0a216c6c	0x000a340a	0x00000000	0x00000000
</code></pre>
<p>함수 unlink() 를 실행 후 A -&gt; buf와 연속된 위치의 값이 변함을 확인했다. fd와 bk 값이라고 추정 할 수 있다.<br>
unlink()를 확인해서 정확한 구조를 알 필요가 있다.</p>
<p><em>(테스트 케이스마다 A, B, C의 주소가 계속 변하고있다. 위의 규칙은 동일했기에 주소만 지속적으로 기록한다.)</em></p>
<h2 id="section">~2019-08-26</h2>
<ol>
<li>구조체의 경우 들어간 변수들은 연속적인 구조를 나타낸다.</li>
<li>shell을 불러오는 함수가</li>
</ol>
<h2 id="풀이2">2019-08-27 풀이2</h2>
<p>0x9e00578 ◂— ‘AAAAAAAA’</p>
<pre><code>pwndbg&gt; x/30x 0x9e00560
0x9e00560:	0x00000000	0x00000000	0x00000000	0x00000021
0x9e00570:	0x09e00590	0x00000000	0x41414141	0x41414141
0x9e00580:	0x00000000	0x00000000	0x00000000	0x00000021
0x9e00590:	0x09e005b0	0x09e00570	0x00000000	0x00000000
0x9e005a0:	0x00000000	0x00000000	0x00000000	0x00000021
0x9e005b0:	0x00000000	0x09e00590	0x00000000	0x00000000
0x9e005c0:	0x00000000	0x00000000	0x00000000	0x00000411
0x9e005d0:	0x20776f6e	0x74616874
</code></pre>
<p>현재 chunk 구조를 간소화 한 듯한 struct 구조는 size, fd, bk, data로 이루어져있다.<br>
A의 fd는 B를 나타내며 B는 C와 A를, C는 B를 연결하고 있다.</p>
<p><strong>main() unlink() 이후</strong></p>
<pre><code>  0x080485f2 &lt;+195&gt;:	call   0x8048504 &lt;unlink&gt;
  0x080485f7 &lt;+200&gt;:	add    esp,0x10
  0x080485fa &lt;+203&gt;:	mov    eax,0x0
  0x080485ff &lt;+208&gt;:	mov    ecx,DWORD PTR [ebp-0x4]
  0x08048602 &lt;+211&gt;:	leave  
  0x08048603 &lt;+212&gt;:	lea    esp,[ecx-0x4]
  0x08048606 &lt;+215&gt;:	ret    
</code></pre>
<p>함수 <code>unlink()</code> 이후에 eax를 초기화 시키고 ecx에 ebp-0x4속 값을 넣은 후<br>
ecx-0x4의 값을 esp로 넣고 있다.</p>
<p>함수 <code>unlink()</code> 디셈블</p>
<pre><code>pwndbg&gt; pdisas unlink
 ► 0x8048504 &lt;unlink&gt;       push   ebp
   0x8048505 &lt;unlink+1&gt;     mov    ebp, esp
   0x8048507 &lt;unlink+3&gt;     sub    esp, 0x10
   0x804850a &lt;unlink+6&gt;     mov    eax, dword ptr [ebp + 8]
   0x804850d &lt;unlink+9&gt;     mov    eax, dword ptr [eax + 4]
   0x8048510 &lt;unlink+12&gt;    mov    dword ptr [ebp - 4], eax
   0x8048513 &lt;unlink+15&gt;    mov    eax, dword ptr [ebp + 8]
   0x8048516 &lt;unlink+18&gt;    mov    eax, dword ptr [eax]
   0x8048518 &lt;unlink+20&gt;    mov    dword ptr [ebp - 8], eax
   0x804851b &lt;unlink+23&gt;    mov    eax, dword ptr [ebp - 8]
   0x804851e &lt;unlink+26&gt;    mov    edx, dword ptr [ebp - 4]
   0x8048521 &lt;unlink+29&gt;    mov    dword ptr [eax + 4], edx
   0x8048524 &lt;unlink+32&gt;    mov    eax, dword ptr [ebp - 4]
   0x8048527 &lt;unlink+35&gt;    mov    edx, dword ptr [ebp - 8]
   0x804852a &lt;unlink+38&gt;    mov    dword ptr [eax], edx
   0x804852c &lt;unlink+40&gt;    nop    
   0x804852d &lt;unlink+41&gt;    leave  
   0x804852e &lt;unlink+42&gt;    ret 
</code></pre>
<p>함수 unlink()는 레지스터를 통하여 ebp로 esp를 조작하고 있다.</p>
<p>esp까지의 변화 과정을 추적하면 아래와 같다.</p>
<ol>
<li>ecx = [ebp-4]</li>
<li>esp = [ecx-4]</li>
<li>esi = esp</li>
</ol>
<p>즉 ebp-4을 찾아야 한다.</p>
<h2 id="풀이3">2019-08-28 풀이3</h2>
<pre><code>EAX  0x9d9f578 ◂— 'AAAAAAAA'
here is stack address leak: 0xffa83bc4
here is heap address leak: 0x9d9f570
pwndbg&gt; x/30x 0xffa83bc4
0xffa83bc4:    !0x09d9f570!     0x09d9f5b0      0x09d9f590      0xf7ef0520
0xffa83bd4:    ?0xffa83bf0?     0x00000000      0xf7cf5b41      0xf7eb5000
0xffa83be4:     0xf7eb5000      0x00000000      0xf7cf5b41      0x00000001
0xffa83bf4:     0xffa83c84      0xffa83c8c      0xffa83c14      0x00000001
0xffa83c04:     0x00000000      0xf7eb5000      0xffffffff      0xf7f09000
pwndbg&gt; p $ebp-4
$1 = (void *) 0xffa83bd4

!  : A의 주소
? : ebp-4
</code></pre>
<p>A의 주소에서 ebp-4까지의 거리는 10byte이다.</p>
<h2 id="풀이4">2019 -08-29 풀이4</h2>
<pre><code>0x9884578 ◂— 'AAAAAAAAA'
Ahere is stack address leak: 0xffd87254
here is heap address leak: 0x9884570
pwndbg&gt; x/50x 0x9884550
0x9884550:	0x00000000	0x00000000	0x00000000	0x00000000
0x9884560:	0x00000000	0x00000000	0x00000000	0x00000021
0x9884570: !0x09884590!	0x00000000	0x41414141	0x41414141
0x9884580:	0x00000041	0x00000000	0x00000000	0x00000021
0x9884590:	0x098845b0	0x09884570	0x00000000	0x00000000
0x98845a0:	0x00000000	0x00000000	0x00000000	0x00000021
0x98845b0:	0x00000000	0x09884590	0x00000000	0x00000000
0x98845c0:	0x00000000	0x00000000	0x00000000	0x00000411
0x98845d0:	0x20776f6e	0x74616874	0x756f7920	0x76616820
0x98845e0:	0x656c2065	0x2c736b61	0x74656720	0x65687320
0x98845f0:	0x0a216c6c	0x000a340a	0x00000000	0x00000000
0x9884600:	0x00000000	0x00000000	0x00000000	0x00000000
</code></pre>

