---
title: "What I learned"
categories:
  - Blog
tags:
  - Post Formats
  - readability
  - standard
---

# 내가 알고리즘 문제풀면서 배운것들..

## 1.member function에서 private 변수의 &리턴하기
```c++
#include <iostream>
#include <vector>
using namespace std;

class Foo {
private:
    vector<int> node;
    vector<int> visit;
public:
    int& visited(const int& cur) { return visit[cur]; }
};

int main(void) {
    Foo f;
    f.visited(10) = 1;//10번 정점 방문체크
    
    return 0;
}
```

처음엔 이게 뭔소린지 싶었다. 아니.. private변수는 저렇게 직접 수정할 수 없도록 존재하는거 아니였나?.. private의 참조를 리턴하면 어떻게 하냐.. 이거 잘못된 코드 아니야?

**하지만 컴파일은 잘됨!!!** ...? 왜죠?

바로 구글링을 시전했습니다. ^^ 역시 stackoverflow.. 바로 발견!

> `private` does not mean "this memory may only be modified by member functions" -- it means "direct attempts to access this variable will result in a compile error". When you expose a reference to the object, you have effectively exposed the object.

> `private`는 "이 메모리는 멤버함수에 의해서만 수정될 수 있다"는 것을 의미하는 것이 아니다. "이 변수에 접근하려는 직접적인 시도는 컴파일 오류를 초래할 것이다." 를 의미한다. 객체에 참조를 노출시키면 객체를 효과적으로 노출시킨다.

--https://stackoverflow.com/questions/4706788/why-can-i-expose-private-members-when-i-return-a-reference-from-a-public-member

!!!... 직접적인 수정이 안된다는 것은 알고 있었다. 하지만 멤버함수에서만 수정할 수 있다는 고정관념에 빠져버린 저였습니다. 필요하다면, private 변수의 참조를 리턴할 수 있다는 사실.. 깨달았습니다. 혹여 이 변수가 수정되는 것이 싫다! 라면 `const int& visited(const int& cur);` 로 하면 된다는 것도 알게되었습니다. 참조를 리턴하면 값을 복사하지 않아도되고, const 값으로 참조를 리턴한다면 수정되는 우려도 사라집니다.

---

### 1.1 vector\<bool\>의 참조를 리턴하기
위에서 본 케이스의 연장선입니다. 사실 위의 visit배열(vector)는 bfs를 돌면서 방문체크 + 최단거리를 기록하기 위해 선언한 배열입니다. 하지만, 단순히 방문을 했다, 안했다만 체크하는 배열이 필요한 문제도 있었습니다.
```c++
#include <iostream>
#include <vector>
using namespace std;

class Foo {
private:
    vector<int> node;
    vector<bool> visit;//참 거짓만 필요
public:
    bool& visited(const int& cur) {
        return visit[cur];//error!!!
    }
};
```
오잉? 하실겁니다. `vector<bool>::operator[]`가 리턴하는 값이 분명 `bool&` 일텐데.. 왜 컴파일이 되지 않느냐.. 무엇이 문제야! -> `비 const 참조에 대한 초기 값은 lvalue여야 합니다.` ???... 아니.. vector의 `[]`는 참조를 리턴하는거 아니였나? `visit[cur]`가 참조값이 아닌가본데...

역시 바로 구글링...^^

> https://docs.microsoft.com/ko-kr/cpp/standard-library/vector-bool-class?view=vs-2019
>
> `operator[]` Returns a simulated reference to the vector\<bool\> element at a specified position.
>
> `operator[]` 는 벡터의 지정된 위치의 simulated된(영어 잘하시는 분..) 참조를 반환한다?

무엇인고... 하니 stl `vector<bool>`은 특수한 케이스의 stl이였던것... stl을 제작하신 위대한 개발자들이 bool을 저장하는데 있어서 1바이트씩(sizeof(bool) = 1) 쓰는 것이 맘에 안들었나보다.. true false를 저장하는데 있어서 단지 1비트만 있으면 되는데 1바이트=8비트 -> 7비트 낭비를 하기 때문 ㄷㄷ.. 극한의 메모리 관리다..

그러니까 `vector<bool>`은 bool값을 저장할 때 공간 하나로 8개의 값을 저장할 수 있는 것. `visit[0]` = vector의 첫번째 요소의 첫 비트. `visit[1]` = vector의 첫번째 요소의 두번째 비트... . . . `visit[7]` = vector의 첫번째 요소의 마지막 비트.(8비트 다 사용) 

그러니까 `vector[n]`을 하면 n위치를 보정한 값이 나오는 것!(참조 값이 아님..) 그럼 값변경이 안되잖아!... 아니야.. `visit[0] = true;`는 잘 되잖아?... 그럼 assign(=) 할 때는 어디다 넣는거야..? 요즘 IDE는 기능도 좋아서 `=` 에 마우스를 올리면,(F12를 눌러도 선언을 볼 수 있다.) 다 나온다. 오잉!!! 반환값이 `vector<bool>::reference`이다!!! `typedef std::_Bit_reference std::vector<bool>::reference`로 되어있다. -> 아하! 저함수는 `_Bit_reference`(비트에 대한 참조)를 리턴하는구나.. 그러니까 대입이 가능했던것.

함수를 `vector<bool>::reference visited(const int& cur);`로 바꾸면 컴파일이 잘된다.. **비트값에 대한 참조**를 리턴하니 잘된다. `vector<bool>`은 표준에 등록되어 있긴 하지만 몇 가지 점에서 문제가 있고 컴파일러마다 지원범위가 달라서 가급적 사용을 자제하는 것이 좋다고 한다..(난 좋은거 같은데..)

---

## 2. class 멤버 함수의 포인터
> http://www.devpia.com/MAEUL/Contents/Detail.aspx?BoardID=50&MAEULNo=20&no=495073&ref=495067 참고1***
>
> https://docs.microsoft.com/ko-kr/cpp/cpp/pointer-to-member-operators-dot-star-and-star?view=vs-2019 참고2

위의 참고사이트1에 정말 기가막히게 잘 답변이 달려있어서.. 꼭! 읽어보시는걸 추천드립니다.

---

### 2.1 class내의 멤버함수포인터
먼저 요약하자면 `C에서의 함수포인터와 C++의 멤버함수 포인터는 문법이 다르다!`라고 할 수 있습니다.
```c++
void foo(void);
int main(void) {
    void (*fp)(void); //함수포인터 선언 - 반환형이 void, 인수가 (void)인 함수포인터
    //1번 주장
    fp = foo; //#1맞음(배열의 이름은 배열의 시작주소->함수의 이름은 함수의 시작주소이다.)
    fp(); //#2맞음(사용할때도 fp()가 맞다!)

    //2번 주장
    fp = &foo; //#3맞음(포인터변수에는 주소가 들어가야한다.)
    (*fp)(); //#4맞음(따라서 사용할때도 (*fp)() 가 맞다!)
}
```

개발자들의 이 두가지 주장이 전부 일리가 있어서 전부 수용한다. 라고합니다... ~~우리만 힘들잖아..~~ 쨋든 C에서는 `#1,#2`와 `#3,#4`의 경우를 둘다 사용할수 있다! 입니다. 하지만 class의 멤버함수의 경우는 다릅니다.

바로, `클래스 내부의 멤버함수를 호출하는 경우`와 `클래스 외부의 함수를 호출하는 경우`를 구분해야 하는것 이었습니다.

```c++
#include <iostream>
using namespace std;

void outFunc(void) { cout << "outFunc called" << endl; }
class Foo {
private:
    //선언하는법
    void (*fp)(void); //외부함수의 포인터!(1번, C스타일 함수포인터)
  //void (*Foo::fp)(void); 와 같은선언임. 클래스내에서의 변수선언은 앞에 범위지정자(Foo::)가 생략되어있음

    void (Foo::*mfp)(void); //Foo 클래스의 멤버함수포인터!(2번, 새로 생긴 멤버함수포인터..)
  //void (*Foo::mfp)(void); 와 전혀다른! 선언임.
public:
    void func(void) { cout << "func called." << endl; }

    //대입법
    void init(void) {
        mfp = func; //@1맞음
        mfp = Foo::func; //@2맞음
        mfp = &Foo::func; //@3맞음 답변자가 읽었던 책에 이방법을 권장한다고 함.
        mfp = &func; /*@4 참고1 사이트의 답변에는 안된다고 나와있는데
                     C++17에선 잘 되는 것 같습니다?..*/
        mfp = outFunc; // 오류!!!->mfp는 Foo클래스의 "멤버함수"를 가리킬 수 있는 포인터야!!!

        fp = outFunc; //맞음 위의 #1번 적용 //c스타일(외부함수 저장)
        fp = &outFunc; //맞음 위의 #3번 적용
        //fp = 에 모든 @1, @2, @3, @4 적용불가능.에러.
    }

    //(클래스 내에서의)호출법
    void call(void) {
        (this->*mfp)();/*(처음보는 ->* 연산자..)(멤버포인터연산자)
        멤버함수이기 때문에 함수포인터로 호출할때
        어느객체(this)가 호출하는지 명시. --참고2*/
        (Foo::*mfp)(); // 문법오류

        fp(); //위의 #2번 적용 //c스타일
        (*fp)(); //위의 #4번 적용
    }
};
```
클래스에서 외부함수를 가리킬 포인터가 필요할 수 있다! 그때 필요한 것이, 기존에 우리가 알고있던 함수포인터이다. 하지만 클래스의 멤버함수를 가리킬 포인터도 필요할 수 있다. 그때 사용하는것이 멤버함수 포인터이다. 함수포인터에 멤버함수를 가리킨다는 조건이 붙은겁니다.

---

### 2.2 클래스 외부에서 클래스의 멤버함수포인터 호출
그렇다면 외부에서 클래스의 멤버함수를 호출할 때에는 어떻게 하느냐? 위에서 한 것을 그대로 바깥으로 가져오면 됩니다.

```c++
class Foo {
private:
public:
    void func(void) { cout << "func called" << endl; }
};

int main(void) {
    void (Foo::*fp)(void) = &Foo::func;//Foo::func 도 가능합니다.
    //선언 및 초기화 //클래스 밖이기 때문에 범위지정 필쑤!(Foo::)

    Foo foo1;
    Foo* foo2 = new Foo;

    //클래스 내부에서는 this가 사용되었죠.
    (foo1.*fp)(); //그냥 변수일때 호출법 .* 연산자가 사용된다.
    (foo2->*fo)(); //포인터 변수일때 호출법 ->* 연산자가 사용된다.
}
```

위에서 배운걸 그대로 적용하면 됩니다. `.*`연산자와 `->*`연산자가 생소할 수 있습니다.. 저도 처음 보게된 연산자입니다 ㅠㅠ 이에대한 설명은 **참고2**사이트에 잘 나와있으니 참고하시기 바랍니다.

---

### 2.3 그래서 위에껀 다 필요없습니다! ~~네?~~
물론 공부한 내용이 다 필요없는건 아닙니다. 지금까지 읽은게 아까워서가 아닙니다 ㅎㅎ.. C++에는 이러한 호출할 수 있는 함수들을 담을 수 있는 class를 제공합니다. ~~(역시 갓갓언어 C++)~~ 바로바로~ STL `function` 입니다. 일반적인 함수, 멤버함수, 람다함수까지 다 넣어버릴 수 있는 객체입니다.

> https://modoocode.com/254 참고3

```c++
#include <iostream>
#include <functional>//function 이용하기위해
using namespace std;

void outFunc(void) {
    cout << "outFunc called" << endl;
}
class Foo {
private:
    //클래스 내부에서 사용시
    function<void(Foo&)> f1 = func; //Foo::func, &Foo::func, &func 전부가능
    fnnction<void(void)> f2 = outFunc; //&outFunc
public:
    void func(void) {
        cout << "func called" << endl;
    }

    void call(void) {
        f1(*this); //$1
        f2(); //$2
    }
};

void func(Foo& foo) {
    cout << "Not Me!" << endl;
}

int main(void) {
    //클래스 외부에서 사용시
    function<void(Foo&)> f = Foo::func;//&Foo::func 도 가능
    //Foo::로 반드시 명시해주셔야 합니다. 왜냐하면 외부함수인 func이 호출될 수 있기 때문입니다.
    //f = func(가능)은 외부함수를 대입하는 겁니다. 물론 실제로 저렇게 헷갈리게 함수를 정의하진 않겠지만..

    Foo foo;
    f(foo); //$3 호출

    return 0;
}
```
function은 우리가 vector나 queue같은 컨테이너를 이용할때처럼 <>안에 타입을 넣어줍니다. `function<void(int)>` 같은 경우에는 반환형이 `void`이고 인자로 `int`하나를 받는다는 뜻입니다.

여기서 `$1`의 경우가 생소할 수 있을 거라 생각합니다. 사실 멤버함수의 경우에는 인자로 그 클래스를 받아옵니다. 그래서 안에서 `this`를 사용할 수 있는 겁니다. 또 어느 객체가 이 함수를 호출하는지 알필요가 있습니다. 예를들면 함수안에서 변수값을 변경했을때 어떤 객체의 변수가 변경되었는지 명확히 할 필요가 있습니다. 그래서 function을 선언할때도 `function<void(Foo&)>`로 선언하는 것입니다. 함수호출을 하는 객체를 알기 위해서말이죠. 호출할때도 인자를 받아서, `f1(*this)`를 해주는 겁니다. 반면에 `$2`경우에는 그저 외부에 정의된 함수이기 때문에 굳이 인자를 넣어주지 않아도 됩니다.

밖에서는 그저 함수호출에 필요한 객체를 function에 넘겨주기만하면 호출해줍니다. `$3`의 경우, foo를 넘겨주면 잘 호출됩니다. 저같은 경우에는 알고리즘문제 탐색부분에서 좌표를 계산할때 많이 이용하는 편입니다.

```C++
#include <iostream>
#include <vector>
#include <algorithm>
#include <functional>
using namespace std;
typedef pair<int, int> point;

namespace move {//좌표이동 함수들
    point up(const point& p) { return point(p.first - 1, p.second); }
    point down(const point& p) { return point(p.first + 1, p.second); }
    point left(const point& p) { return point(p.first, p.second - 1); }
    point right(const point& p) { return point(p.first, p.second + 1); }
}
class Foo {
    function<point(const point&)> fp[4] = { move::up, move::down, move::left, move::right };
    //함수들을 배열로 저장
    bool visited[100][100] = { 0, };

    void dfs(const point& cur) {
        visited[cur.first][cur.second] = true;
        for (auto& call : fp) {
            point next(call(cur));//함수호출(다음좌표 계산)
            if (condition(next)) {
                doSomething...
            }
        }
    }
}
```

---

## 3. std::unordered_map
c++에도 dictionary와 같은 기능을 하는 std::map 이라는 stl이 있습니다. 어떤 특정한 key값에 대응하는 value값을 저장할 수 있는 컨테이너입니다. map은 키값들을 정렬해서 저장해놓습니다. `std::map`은 키값 비교로 `std::less`를 기본적으로 사용하기 때문에 (`map<int, string, less<int>>`) 키값이 int이든 string이든 pair(less가 정의되어 있습니다.)든 어느 키값이든 다 사용할 수 있죠.

하지만 저같은 경우에는 문제풀이시 굳이 key값에 대한 정렬이 필요없는 경우도 있었습니다. `map`은 O(logN), `unordered_map`은 O(1)... 그럴땐 그저 key에대한 value를 딱딱 내주는 unordered_map을 사용하면 됬었습니다. 여기서 생기는 문제점이! 나는 key값으로 pair를 주고싶었는데, 사용할 수 없다고 나오는 것... 그 이유인즉슨 `unordered_map`은 3번째 인수로 `std::hash`를 사용하는 것이었습니다. 그러니까 key값을 hash값으로 변환해서 그에 따른 value값을 얻어내는 구조였습니다(hash table). `std::hash`에는 char, int, string 등등의 자료형은 hash값을 계산할 수 있지만 pair는 정의되어있지 않습니다. 따라서 사용할 수 있는 2가지 방법들을 소개합니다.

> https://www.techiedelight.com/use-std-pair-key-std-unordered_map-cpp/

### 3.1 나만의 hash함수
```c++
#include <unordered_map>
using namespace std;
typedef pair<int, int> point;

struct point_hash {
    size_t operator() (const point& p) const {
        return hash<int>()(p.first) ^ hash<int>()(p.second);
    }
};

int main(void) {
    unorderd_map<point, int, point_hash> um;//사용가능
}
```
간단하게 이용할 수 있는 hash함수이다. 구조체에 `operator()`를 정의해서 사용할 수 있다. 알고리즘 문제푸는 수준은 이정도면 괜찮지만! 실제로는 많은 해시값 충돌이 발생한다 ㅠㅠ.. 그래서 XOR 연산을 하기전에 해시값을 shift 하거나 rotate하면 어느정도 예방할 수 있다고 합니다.

또 vector에 대한 해쉬테이블이 필요한 경우도 있었습니다. 이런경우에도 `unordered_map`에 vector를 해싱하는 함수를 명시적으로 표시해야합니다. 제가 생각한건 배열안의 값을 하나씩 해싱해서 더한다거나, shift하고 rotate하는거? 근데 구글링하다 꽤 맘에들고 괜찮은 방법이 있었습니다. 

```c++
#include <vector>
#include <unordered_map>
using namespace std;

struct vector_hash {
    size_t operator() (const vector<int>& v) const {
        //v를 string으로 바꾸고 그 string을 해싱하기!!!
        return hash<string>()(string(v.begin(), v.end()));
    }
};

int main(void) {
    vector<int> v;
    unorderd_map<point, int, vector_hash> um;//사용가능
}
```



---

### 3.2 boost::hash
굉장히 엄청난 개발자들이 만든 boost 라이브러리에 있는 hash함수들을 이용하면된다.. pair는 물론 포인터나 배열까지 해싱할 수 있는 라이브러리입니다.  `#include <boost/functional/hash.hpp>`를 include하고 `boost::hash<point>`를 사용할 수 있다는 것.

`unordered_map<point, int, boost::hash<point>>`로 선언하여 사용할 수 있다!!!