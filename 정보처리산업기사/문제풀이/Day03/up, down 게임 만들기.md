# up, down 게임 만들기

[목록으로](../../README.md)

[노션 원문](https://app.notion.com/p/261c2c49a250807799b3de8e7b6fb6e8)

조건 
// 1. f11을 눌러서 시작하면 사용자에게 숫자를 입력하세요. 라는 문구를 하나 보여주고 실제로 입력을 받는다.<br>// 2. 입력을 받으면 입력받은 값과 rs가 가진 값을 비교하여 up인지, down인지 사용자에게 출력으로 알려준다.<br>// 3. 다시 숫자를 입력하세요. 라는 문구를 보여주며 다시 입력을 받을 수 있게 해서 또 입력받고 2번의 과정을 반복<br>// 4. 정답을 맞추면 정답입니다. 출력해서 보여주고 프로그램 종료

최초에 만든 것 
import java.util.Scanner;
class Main \{<br>public static void main(String\[\] args) \{<br>//	  rs(1 \~ 100 사이의 랜덤한 숫자 생성 코드)<br>int rs = (int) (Math.random() \* 100) + 1;
//	  자바에서의 입력을 위한 준비<br>Scanner sc = new Scanner([System.in](http://system.in/));
```plain text
  System.out.println("숫자를 입력하세요. : " );

```
//	  자바에서 입력이라는 행위<br>int num = sc.nextInt();
```plain text
  while(rs != num) {
	  if(rs > num) {
		  System.out.println("UP");
		  System.out.println("다시 숫자를 입력하세요. : ");
		  num = sc.nextInt();
	  } else if(rs < num) {
		  System.out.println("DOWN");
		  System.out.println("다시 숫자를 입력하세요. : ");
		  num = sc.nextInt();
	  }
  }
  if(rs == num) {
	  System.out.println("정답입니다.");
  }

```
//	  scanner 자원 종료 코드<br>sc.close();<br>\}<br>\}

# **강사님 조언을 얻어서 조금 더 간결하게 만들었다.**
import java.util.Scanner;
class Main \{<br>public static void main(String\[\] args) \{
//	  rs(1 \~ 100 사이의 랜덤한 숫자 생성 코드)<br>int rs = (int) (Math.random() \* 100) + 1;
//	  자바에서의 입력을 위한 준비<br>Scanner sc = new Scanner([System.in](http://system.in/));
```plain text
  System.out.println("숫자를 입력하세요. : " );

```
//	  자바에서 입력이라는 행위<br>int num = sc.nextInt();
```plain text
  while(rs != num) {
	  if(rs > num) {
		  System.out.println("UP");
		  System.out.println("다시 숫자를 입력하세요. : ");
		  num = sc.nextInt();
	  } else if(rs < num) {
		  System.out.println("DOWN");
		  System.out.println("다시 숫자를 입력하세요. : ");
		  num = sc.nextInt();
	  }
  }
  if(rs == num) {
	  System.out.println("정답입니다.");
  }

```
//	  scanner 자원 종료 코드<br>sc.close();<br>\}<br>\}

- [첨부파일: 9.1_up_down_문제_만들기.txt](../../attachments/9.1_up_down_%EB%AC%B8%EC%A0%9C_%EB%A7%8C%EB%93%A4%EA%B8%B0.txt)
