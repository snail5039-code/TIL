# 인자, 매개변수, return 문제 풀이

[목록으로](../../README.md)

[노션 원문](https://app.notion.com/p/265c2c49a25080a8b3d1ef29328ca11e)

```javascript
class Main {
  public static void main(String[] args) {
    계산기.합(10, 20);
    // 출력 => 결과 : 30
   System.out.println("결과 : " + 계산기.합(10, 20));
    계산기.합(50, 20);
    // 출력 => 결과 : 70
    System.out.println("결과 : " + 계산기.합(50, 20));
    계산기.빼기(50, 20);
    // 출력 => 결과 : 30
    System.out.println("결과 : " + 계산기.빼기(50, 20));
    계산기.빼기(5, 2);
    // 출력 => 결과 : 3
    System.out.println("결과 : " + 계산기.빼기(5, 2));
    계산기.곱하기(5, 2);
    // 출력 => 결과 : 10
    System.out.println("결과 : " + 계산기.곱하기(5, 2));
  }
}
class 계산기{
	static int 합(int a, int b) {
		return a + b;
	}
	static int 빼기(int a, int b) {
		return a - b;
	}
	static int 곱하기(int a, int b) {
		return a * b;
	}
}


class Main {
	public static void main(String[] args) {
		int 결과;

		결과 = 계산기.합(10, 20);
		System.out.println("결과 : " + 계산기.합(10, 20));
		// 출력 => 결과 : 30

		결과 = 계산기.합(30, 20);
		System.out.println("결과 : " + 계산기.합(30, 20));
		// 출력 => 결과 : 50

		결과 = 계산기.합(30, 70);
		System.out.println("결과 : " + 계산기.합(30, 70));
		// 출력 => 결과 : 100

		결과 = 계산기.차(30, 70);
		System.out.println("결과 : " + 계산기.차(30, 70));
		// 출력 => 결과 : -40

		결과 = 계산기.곱(3, 7);
		System.out.println("결과 : " + 계산기.곱(3, 7));
		// 출력 => 결과 : 21
	}
}
class 계산기 {
	static int 합(int a, int b) {
		return a + b;
	}
	static int 차(int a, int b) {
		return a - b;
	}
	static int 곱(int a, int b) {
		return a * b;
	}
}

//// 문제 : 아래와 같이 출력 되도록 해주세요.

//class Main {
//  public static void main(String[] args) {
//    int 결과1 = Math.oneToSum(3);
//    System.out.println("결과1 : " + 결과1);
//    // 출력 : 결과1 : 6
//    
//    int 결과2 = Math.oneToSum(10);
//    System.out.println("결과2 : " + 결과2);
//    // 출력 : 결과2 : 55
//  }
//}
//
//class Math {
//  static int oneToSum(int n) {
//	  if(n == 1) {
//		  return 1;
//	  }
//	  return n + oneToSum(n - 1);
//  }
//}


// 문제 : 아래와 같이 출력 되도록 해주세요.

class Main {
  public static void main(String[] args) {
    int 결과1 = Math.nToMSum(2, 3);
    System.out.println("결과1 : " + 결과1);
    // 출력 : 결과1 : 5
    
    int 결과2 = 30 + Math.nToMSum(5, 10);
    System.out.println("결과2 : " + 결과2);
    // 출력 : 결과2 : 45
  }
}

class Math {
	static int nToMSum(int a, int b) {
		return a + b;
	}
}


마지막 문제는 재귀함수를 어떻게 하는지 몰라서 그냥 해봤다.

강사님 문제 풀이 

1.
class Main {
  public static void main(String[] args) {
    계산기.합(10, 20);
    // 출력 => 결과 : 30
    계산기.합(50, 20);
    // 출력 => 결과 : 70
    계산기.빼기(50, 20);
    // 출력 => 결과 : 30
    계산기.빼기(5, 2);
    // 출력 => 결과 : 3
    계산기.곱하기(5, 2);
    // 출력 => 결과 : 10
  }
}
class 계산기{
	static void 합(int a, int b) {
		System.out.println("결과 : " + (a + b));
	}
	static void 빼기(int a, int b) {
		System.out.println("결과 : " + (a - b));
	}
	static void 곱하기(int a, int b) {
		System.out.println("결과 : " + (a * b));
	}
}



2.
class Main {
	public static void main(String[] args) {
		int 결과;

		결과 = 계산기.합(10, 20);
		System.out.println("결과 : " + 결과);
		// 출력 => 결과 : 30

		결과 = 계산기.합(30, 20);
		System.out.println("결과 : " + 결과);
		// 출력 => 결과 : 50

		결과 = 계산기.합(30, 70);
		System.out.println("결과 : " + 결과);
		// 출력 => 결과 : 100

		결과 = 계산기.차(30, 70);
		System.out.println("결과 : " + 결과);
		// 출력 => 결과 : -40

		결과 = 계산기.곱(3, 7);
		System.out.println("결과 : " + 결과);
		// 출력 => 결과 : 21
	}
}
class 계산기 {
	static int 합(int a, int b) {
		return a + b;
	}
	static int 차(int a, int b) {
		return a - b;
	}
	static int 곱(int a, int b) {
		return a * b;
	}
}


3.
//// 문제 : 아래와 같이 출력 되도록 해주세요.

class Main {
  public static void main(String[] args) {
    int 결과1 = Math.oneToSum(3);
    System.out.println("결과1 : " + 결과1);
    // 출력 : 결과1 : 6
    
    int 결과2 = Math.oneToSum(10);
    System.out.println("결과2 : " + 결과2);
    // 출력 : 결과2 : 55
  }
}

class Math {
	static int oneToSum(int n) {
		int sum = 0;
		for (int i = 1; i <= n; i++) {
			sum += i;
		}
		return sum;
	}
}

4.
// 문제 : 아래와 같이 출력 되도록 해주세요.

class Main {
  public static void main(String[] args) {
    int 결과1 = Math.nToMSum(2, 3); // a, b
    System.out.println("결과1 : " + 결과1);
    // 출력 : 결과1 : 5
    
    int 결과2 = Math.nToMSum(5, 10);
    System.out.println("결과2 : " + 결과2);
    // 출력 : 결과2 : 45
  }
}

class Math {
	static int nToMSum(int a, int b) {
		
		int sum = 0;
		
		for(int i = a; i <= b; i++) {
			sum += i;
		}
		return sum;
	}
}

요렇게도 할 수 있다.
for( ; a <= b; a++) {
			sum += a;
		}
왜냐 변수는 이미 선언되있으니깐.
```
