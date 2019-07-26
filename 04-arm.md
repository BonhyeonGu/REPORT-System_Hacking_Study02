---


---

<h1 id="pwnable.krasm"><a href="http://pwnable.kr">pwnable.kr</a>:asm</h1>
<h2 id="해석">2019-07-26 해석</h2>
<p>readme에 들어가보면</p>
<pre><code>once you connect to port 9026, the "asm" binary will be executed under asm_pwn privilege.

make connection to challenge (nc 0 9026) then get the flag. (file name of the flag is same as the one in this directory)
</code></pre>
<p><code>nc 0 9026</code>을 통해 flag를 휙득해라며 파일 이름은 현제 디렉토리에 있는 것과 같다고 나와있다.</p>
<p>c파일에는 평소에 접하지 않는 함수들이 있다. 해석부터 진행해보자.</p>
<pre><code>setvbuf(stdout, 0, _IONBF, 0);
setvbuf(stdin, 0, _IOLBF, 0);
</code></pre>
<p>input, output 버퍼를 초기화 한다.</p>
<pre><code>char* sh = (char*)mmap(0x41414000, 0x1000, 7, MAP_ANONYMOUS | MAP_FIXED | MAP_PRIVATE, 0, 0);
memset(sh, 0x90, 0x1000);
memcpy(sh, stub, strlen(stub));
</code></pre>
<p><code>mmap()</code> 은 주소공간을 맵핑해주는 함수이다. 원하는 주소, 권한, 용량에 맵핑 할 수 있다.<br>
현재 소스에는 char 타입 포인터 sh를 0x41414000 부터 0x1000 만큼의 사이즈로 읽기, 쓰기, 실행 권한이 가능하며(추정), 파일과 연결되지 않은 체 0으로 초기화 된 영역, 고정된 주소로, 다른 프로세스는 접근하지 못하게끔하며 파일은 연결되지 않고, 스킵은 존재하지 않는 메모리 영역으로 맵핑 한다는 뜻이다.</p>
<p>다음 <code>memset()</code>으로 sh를 0x90(NOP)로 채운 후 <code>memcpy()</code>가 stub를 stub의 용량 만큼 sh에 복사한다.</p>

