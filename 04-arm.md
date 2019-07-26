---


---

<h1 id="pwnable.krasm"><a href="http://pwnable.kr">pwnable.kr</a>:asm</h1>
<h2 id="해석">2019-07-25 해석</h2>
<p>c파일에는 평소에 접하지 않는 함수들이 있다. 해석부터 진행해보자.</p>
<pre><code>setvbuf(stdout, 0, _IONBF, 0);
setvbuf(stdin, 0, _IOLBF, 0);
</code></pre>
<p>input, output 버퍼를 초기화 한다.</p>
<pre><code>char* sh = (char*)mmap(0x41414000, 0x1000, 7, MAP_ANONYMOUS | MAP_FIXED | MAP_PRIVATE, 0, 0);
memset(sh, 0x90, 0x1000);
memcpy(sh, stub, strlen(stub));
</code></pre>
<p>mmap</p>

