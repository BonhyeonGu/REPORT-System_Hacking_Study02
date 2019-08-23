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
<li>chunk는 prev_size ,size, fd, bk, ,fd_nextsize, bk_nextsize, flag로 이루어져있다.</li>
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
<h2 id="풀이">2019-08-23 풀이</h2>
<p>here is stack address leak: 0xff939b94<br>
here is heap address leak: 0x9070570</p>

