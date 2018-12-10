---
title: Toddler's bottle - fd
date: 2018-12-03
tags: [pwnable, pwnable.kr]
author: arizona
type: post
---

### fd (1pt)

![fd](/static/fd1.PNG)

<br>
천리길도 한걸음부터야 1번문제 부터 ㄱㄱ 아자아자
ssh로 접속해보쟝 --->  Domain : "fd@pwnable.kr"   Port : 2222   pw : "guest"\
오 포너블 멋진문구가 두둥하고 등장해.

```sh
fd@ubuntu:~$ whoami
fd
fd@ubuntu:~$ id
uid=1004(fd) gid=1004(fd) groups=1004(fd)
```

오옹 나는 fd 라는 이름의 사용자구 uid gid 는 저따구구나!\
ls -al 한번 때려보쟈!

```sh
fd@ubuntu:~$ ls -al
total 40
drwxr-x---  5 root   fd   4096 Oct 26  2016 .
drwxr-xr-x 93 root   root 4096 Oct 10 22:56 ..
d---------  2 root   root 4096 Jun 12  2014 .bash_history
-r-sr-x---  1 fd_pwn fd   7322 Jun 11  2014 fd
-rw-r--r--  1 root   root  418 Jun 11  2014 fd.c
-r--r-----  1 fd_pwn root   50 Jun 11  2014 flag
-rw-------  1 root   root  128 Oct 26  2016 .gdb_history
dr-xr-xr-x  2 root   root 4096 Dec 19  2016 .irssi
drwxr-xr-x  2 root   root 4096 Oct 23  2016 .pwntools-cache
```

-r-sr-x---  1  fd_pwn  fd  7322 Jun 11 2014 fd  ??? ㅎㅎ  이것은 다음을 의미해<br><br>

- \-: 이것은 파일입니다! 디렉토리(d)가 아닙니다!\
- r-s: 사용자(fd) 가 이 파일을 읽기, 실행 권한이 있오. **setuid** 가 설정되어있오. ( 밑에 설명있어! )\
- r-x: 그룹이 파일을 읽기, 실행, 접근할 권한이 있오.\
- ---: 그룹 외에 다른사람은 접근을 몬해.ㅠ\
- 1: 링크수는 한개\
- fd_pwn: 이 파일은 소유자가 fd_pwn 이야!\
- fd:  이 파일은 fd 그룹에 속해있오. 리눅스에서는 사용자랑 동일한 이름의 그룹을 만드는거 알지?\
- 7322: 이 파일의 크기야.\
- Jun 11 2014: 이 파일 마지막 수정시간이야!\
- fd: 이 파일의 이름이야.\

<br>응 잠만 **setuid** 라는건 뭐지??  -> 이 파일에 맞게 설명해주자면,

한마디로,, **fd 파일을 실행할 때 만큼은** "fd 파일의 소유자" ,즉 **fd_pwn 의 권한을 얻는거야!!** 구체적으론,,  소문자s는 실행권한이 있는 상태야. fd 를 실행시켰다고 해보자.! 프로세스가 생성됐지? 그 프로세스 의 *EUID* 가 fd_pwn 으로 바뀌게 돼. *EUID* 는 프로세스가 갖는 ID 5개중하나야. "어떤 유저권한으로 프로세스를 실행하고 있니?" 를 적어놓은 거지.  이게 만약 취약한 실행파일인데,,소유자가 fd_pwn 가 아니라 root라면 ㅎㅎㅎ 이 컴퓨터는 끝난거야 ㅇㅋ?

아 그럼 *EUID* 말고는 또 무슨 ID 가 있으깡??<br><br>

1. PID : 프로세스 식별자야. 알지?? ㅎㅎ 프로세스에 번호표를 준거야.
2. RUID : 실제 사용자의 ID야. 저 파일을 실행시키면 1004(fd) 가 뜨겠지?
3. EUID : 유효 사용자 ID . 즉 fd 파일을 실행시키면 euid 값이 fd_pwn 의 uid 로 바뀌겠지!
4. RGID : 실제 사용자의 그룹 ID 야. 1004(fd) 가 뜨겠지.
5. EGID : 유효 사용자 그룹 ID 야. 여기서 감이잡히지? ㅎㅎ setgid 라는것도 있어.

<br>접근권한에 영향을 주는건, EUID, EGID 라구!! -> 매우 보안에 신경쓰야겠지?

자자.. fd.c 파일도 줬네 친절해라.. cat fd.c!

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
char buf[32];
int main(int argc, char* argv[], char* envp[]){
	if(argc<2){
		printf("pass argv[1] a number\n");
		return 0;
	}
	int fd = atoi( argv[1] ) - 0x1234;
	int len = 0;
	len = read(fd, buf, 32);
	if(!strcmp("LETMEWIN\n", buf)){
		printf("good job :)\n");
		system("/bin/cat flag");
		exit(0);
	}
	printf("learn about Linux file IO\n");
	return 0;

}
```

어디보ㅈr..  처음 if 문은 매개변수 넣어라 짜샤.. 이런뜻이고 ..
매개변수에 atoi 를 씌우네? 숫자로 넣어라 짜샤.. 이런뜻이고..
100 을 넣으면..  int  fd 는 100 - 0x1234 가 되네.. 흠 이게 *read(fd,buf,32)* 를 수행하네..
그다음 if 문에서 buf 랑 "LET~" 문자열이랑 같으면 플래그 떨구네..

**read(fd, buf, 32)** 이거 무슨함순지 다 알제??<br><br>

- fd:  파일 디스크립터라고 부르는 애야. 0 : 입력, 1 : 출력, 2 : 에러
- buf: 입력을 저장할곳 or 출력할것을 저장할곳 or 에러를 저장할곳 을 적어놓은 주소값.
- 32:  buf 라는 주소에 적힌 공간은 32 바이트 초과 건드리지마.. 라는 뜻이야.

즉, 이 문제는 fd 를 0 만들어주고, 입력창이 나오면 저 LET~ 문자열을 치면 되는거지!!  0x1234 = 4660 이니까..

**payload : ./fd 4660  ,  LETMEWIN**

```sh
fd@ubuntu:~$ ./fd 4660
LETMEWIN
good job :)
mommy! I think I know what a file descriptor is!!
```
