---


---

<h1 id="pwnable.krleg-문제-풀이"><a href="http://pwnable.kr">pwnable.kr</a>:leg 문제 풀이</h1>
<h2 id="관찰">2019-07-22 관찰</h2>
<p>문제는 arm식 아키텍처 구조로 되어있다.</p>
<p>*leg.c (main)</p>
<pre><code>int main(){
	int key=0;
	printf("Daddy has very strong arm! : ");
	scanf("%d", &amp;key);
	if( (key1()+key2()+key3()) == key ){
		printf("Congratz!\n");
		int fd = open("flag", O_RDONLY);
		char buf[100];
		int r = read(fd, buf, 100);
		write(0, buf, r);
	}
	else{
		printf("I have strong leg :P\n");
	}
	return 0;
}
</code></pre>
<p>key 1부터 3까지의 함수의 리턴값을 더한것과 입력한 값이 같으면 flag를 출력하게 되어있다. 즉 key 함수들의 리턴값을 알아내는 것이 목표다.</p>
<pre><code>   0x00008d68 &lt;+44&gt;:	bl	0x8cd4 &lt;key1&gt;
   0x00008d6c &lt;+48&gt;:	mov	r4, r0
   0x00008d70 &lt;+52&gt;:	bl	0x8cf0 &lt;key2&gt;
   0x00008d74 &lt;+56&gt;:	mov	r3, r0
   0x00008d78 &lt;+60&gt;:	add	r4, r4, r3
   0x00008d7c &lt;+64&gt;:	bl	0x8d20 &lt;key3&gt;
   0x00008d80 &lt;+68&gt;:	mov	r3, r0
   0x00008d84 &lt;+72&gt;:	add	r2, r4, r3
   0x00008d88 &lt;+76&gt;:	ldr	r3, [r11, #-16]
   0x00008d8c &lt;+80&gt;:	cmp	r2, r3
</code></pre>
<p>함수가 출력되고 나서 mov r?, r0를 한 뒤 add하며 내려가고 있다. add는 if( (key1()+key2()+key3()) == key )를 하기 위한 과정이며 r0에서 다른 레지스터로 복사되는 것으로 보아 r0를 추적하는 것이 효과적일 것이다.</p>
<h2 id="arm-레지스터명령어">2019-07-23 arm 레지스터&amp;명령어</h2>
<p>*레지스터<br>
<code>pc</code> : intel의 eip와 유사하며 그 다음 명령어의 주소가 들어간다.<br>
<code>r0~r12</code> : 범용 레지스터<br>
<code>r13(sp)</code> : intel의 esp와 유사하며 스택의 시작지점 주소가 들어간다.<br>
<code>r14(lr)</code> : 함수가 끝나고 되돌아갈 주소가 저장된다. 함수 시작 전에 저장한다.</p>
<p>*Pipelining(파이프 라인)<br>
cpu에서 계산을 할 때에 한 명령어를 실행 할 때마다. 그 명령어의 레지스터를 올리고, 해석하고, 실행하고, 메모리에 기록하는 과정을 반복하는 것 보다 빠른 방법이다.<br>
각 명령어마다 다음 명령어로 지나갈때 한 가지 과정 씩 순차적으로 넘어간다. 0x001에 call a가 0x002에 call b가<br>
0x003에 call c가 있다고 하면</p>
<ol>
<li>call a가 Instruction Fetch 한다.</li>
<li>call a가 Instruction Decode 하고 call b가 Instruction Fetch한다.</li>
<li>call a가 Execute 하고 call b가 Instruction Decode하고 call c가 Instruction Fetch 한다.</li>
<li>call a가 Write-Back 하고 call b가 Execute 하고 call c가 Instruction Decode하고 call d가 Instruction Fetch 한다.</li>
<li>call b가 Write-Back 하고 call c가 Execute 하고 call d가 Instruction Decode한다.<br>
.<br>
.<br>
.</li>
</ol>
<p>식으로 진행된다.</p>
<p>*명령어<br>
<code>ldr r1, r2</code> :  r2(주소)의 값을 읽어와 r1(주소)에 저장<br>
<code>ldr r1, [r2, #16]</code> : r2(주소)에 16byte를 더한 주소의 값을 읽어와 r1(주소)에 저장</p>
<p><code>str r1, r2</code> : r1(값)을 r2(주소)에 저장<br>
<code>str r1, [r2], #4</code> : r1(값)을 r2(주소)에 저장 후 r2(주소)에 4byte를 더함</p>
<h2 id="key1">2019-07-24 key1()</h2>
<pre><code>(gdb) disass key1
Dump of assembler code for function key1:
   0x00008cd4 &lt;+0&gt;:	push	{r11}		; (str r11, [sp, #-4]!)
   0x00008cd8 &lt;+4&gt;:	add	r11, sp, #0
   0x00008cdc &lt;+8&gt;:	mov	r3, pc
   0x00008ce0 &lt;+12&gt;:	mov	r0, r3
   0x00008ce4 &lt;+16&gt;:	sub	sp, r11, #0
   0x00008ce8 &lt;+20&gt;:	pop	{r11}		; (ldr r11, [sp], #4)
   0x00008cec &lt;+24&gt;:	bx	lr
</code></pre>
<p>pc가 r3로 r3가 r0로 향한다.<br>
이때 0x8cdc 기준 pc는 0x8ce0 가 아니다. 0x8ce0를 실행할 때는 아직 0x8cdc이 Decode하고 있기 때문이다.<br>
그 다음 명령 0x8ce4을 진행하고 있을 때가 0x8cdc의 Execute때이다.</p>
<p>즉 r0는 0x8ce4이다.</p>
<h2 id="key2">key2()</h2>
<pre><code>(gdb) disass key2
Dump of assembler code for function key2:
   0x00008cf0 &lt;+0&gt;:	push	{r11}		; (str r11, [sp, #-4]!)
   0x00008cf4 &lt;+4&gt;:	add	r11, sp, #0
   0x00008cf8 &lt;+8&gt;:	push	{r6}		; (str r6, [sp, #-4]!)
   0x00008cfc &lt;+12&gt;:	add	r6, pc, #1
   0x00008d00 &lt;+16&gt;:	bx	r6
   0x00008d04 &lt;+20&gt;:	mov	r3, pc
   0x00008d06 &lt;+22&gt;:	adds	r3, #4
   0x00008d08 &lt;+24&gt;:	push	{r3}
   0x00008d0a &lt;+26&gt;:	pop	{pc}
   0x00008d0c &lt;+28&gt;:	pop	{r6}		; (ldr r6, [sp], #4)
   0x00008d10 &lt;+32&gt;:	mov	r0, r3
   0x00008d14 &lt;+36&gt;:	sub	sp, r11, #0
   0x00008d18 &lt;+40&gt;:	pop	{r11}		; (ldr r11, [sp], #4)
   0x00008d1c &lt;+44&gt;:	bx	lr
</code></pre>
<p>pc가 r3로 r3에 4를 더한 후 값이 r0로 이동된다.<br>
0x8d04의 pc는 0x8d08이다. 즉  r0는 0x8d0c이다.</p>
<h2 id="key3">key3()</h2>
<pre><code> 0x00008d20 &lt;+0&gt;:	push	{r11}		; (str r11, [sp, #-4]!)
 0x00008d24 &lt;+4&gt;:	add	r11, sp, #0
 0x00008d28 &lt;+8&gt;:	mov	r3, lr
 0x00008d2c &lt;+12&gt;:	mov	r0, r3
 0x00008d30 &lt;+16&gt;:	sub	sp, r11, #0
 0x00008d34 &lt;+20&gt;:	pop	{r11}		; (ldr r11, [sp], #4)
 0x00008d38 &lt;+24&gt;:	bx	lr
</code></pre>
<p>lr이 r3으로 r3이 r0가 되었다.<br>
0x8d28의 lr은 특징에 따라 key3()가 끝나고 가야하는 지점이 담겨 있을 것이다.</p>
<p>*main+64</p>
<pre><code>0x00008d7c &lt;+64&gt;:	bl	0x8d20 &lt;key3&gt;
0x00008d80 &lt;+68&gt;:	mov	r3, r0
0x00008d84 &lt;+72&gt;:	add	r2, r4, r3
</code></pre>
<p>r0는 0x8d80이다.</p>
<h2 id="결과">결과</h2>
<p>각각의 r0를 더해본다. 0x8ce4 + 0x8d0c + 0x8d80 = 0x1a770(108400)</p>
<p>My daddy has a lot of ARMv5te muscle!</p>

