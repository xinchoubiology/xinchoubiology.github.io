---
layout: post
title: Y-combinator：lambda function's recursion
description: How to create a automate iterator via lambda calculation
keywords: "lambda calculus, combinator, recursion, closet, Lisp"
category: lambda_calculus
tags: [lambda_calculus, recursion, combinator]
---
{% include JB/setup %}

> 緣起 - 還是要從最近玩的一個比較有意思的Toy說起，[『DrRacket』](http://docs.racket-lang.org/drracket/), 涉及到兩個叠代的繪圖 `procedure`,
	而我希望做的工作就是對這兩個叠代split `procedure` 函數做一個 high-order abstraction。其實對於瞭解Y-combinator的來說，一拉到底爺就可以了，最後有個關於lambda recursion基於Y-combinator的實現。

<!-- more -->

首先，我得到四種遞歸畫圖的`procudure`, 他們分別被命名為：`right-split`,	 `up-split`, 	`left-split`,	`down-split`, 代表圖像叠代的方向是identity picture的上，下，左，右四個方位。下面我寫出具體的代碼和示意圖來作為演示。[後面我們會明白之所以使用這種繪圖函數來作為這個問題的引子，是因為其本身具有的閉包(closure)屬性]

```lisp
(require (planet "sicp.ss" ("soegaard" "sicp.plt" 2 1)))
(define (right-split painter n)
	(if (= n 0)
	     painter
	     (let ((smaller (right-split painter (- n 1))))
	     	 (beside painter (below smaller smaller)))))
```

結果如下圖：

 ![right-split einstein](/assets/images/2014/07/right-split.png) 

 原諒我使用偉大的 Einstein 來做這個例子的說明，只是因為好像Library作者們似乎比較喜歡這位？ 

 同理，我們也可以用類似的叠代思想來表示 `up-split` 過程：

```lisp
 (define (up-split painter n)
 	(if (= n 0)
 	     painter
 	     (let ((smaller (up-split painter (- n 1))))
 	     	(below painter (beside smaller smaller)))))
```

 Usage:

```lisp
 (paint (up-split einstein 4))
```

 結果是這樣的：

 ![up-split einstein](/assets/images/2014/07/up-split.png) 

 從Einstein分裂的走時來看，我們已經可以發現 `below`, `beside` 的組合先後順序似乎決定了painter的組合方式的所有。

 從上面兩個例子我們就可以看出來，實際上，這個 `[direction]-split` procedure 本質上都是split 叠代的向 identity picture 上面組合疊加的函數，所以，我們受到上面的 `right-split`,	 `up-split` 的影響，進一步希望製作抽象procedure歸納出所有方向上的`split`函數。

 正如上面的[up-split einstein] 圖所說的，我們實際上這兩個函數只是 `below` 和 `beside` 這兩個過程的嵌套位置顛倒而已，如此的話，實際上就是說我們也應該可以抽象出過程滿足如下方式生成`up-split` 和 `right-split`

```lisp
	(define left-split (split beside below))
	(define up-split (split below beside))
```

顯然，split是這四個方位split函數的函數抽象，所以，實際上，從上面這個定義抽象上來看，至少還有  painter 和 n 兩個參數實際上是被構建的 Abstraction Barrier 給個離開了。

如此的話，我們至少可以猜測出 `split` 過程返回的應該是一個lambda 函數，他的形式如下  *(lambda (painter n) (...))* 的形式。 於是，我們的問題就此出現了。

* This question is related to the form of Procedure  `split` 's output -- *(lambda (painter n) (...))* ：

	+ 首先，我瞭解到，這個返回的 `(lambda (painter n)`  實際上肯定是等價於 `up-split` 和 `right-split` 的。

	+ 所以，我們確定  `(lambda (painter n)`  對應的是一個遞歸函數，類似於 `factorial`,  `(up-split painter n)` 的值來自於 `(up-split painter (- n 1))` 的定義。

	+ 所以這個問題就進一步可以被泛華做，我們應該如何是使用 lambda 函數來表示一個關於滋生的recursion 過程？

	+ 這裏顯然不同於普通定義法的地方就在於，實際上我們不可能像普通定義中使用 (up-split painter (- n 1)) 的方法來顯式的調用函數本身來完成遞歸了。因為這裏的lambda函數是匿名的，他們有一個名稱來給我進行引用。

所以，今天我寫下這篇post的目的，就是希望能基於lambda 函數的由來 lambda calculus 來分析究竟該如何在 匿名函數的情況下，來完成遞歸函數的定義。lambda函數具有的強大的表達能力，所以我們希望在其自身找到如何基於其定義完成遞歸的方法。

>  基石 - $\lambda \ Calculus$: 

*  我們首先從函數開始說起，函數是一個數學中非常基本的概念，例如，如果我們希望能夠使用從變量到值的mapping的時候，我們一般把他寫成 $f：x \mapsto x - y$ 的形式，但是，這裏我們給函數賦予的「函數名」f 是必須的嗎？ 顯然，過多的特殊標註會導致我們在使用高階函數的時候帶來混亂，這也就是Church的標註發的好處了。我們ignore每一個函數的函數名稱，取而代之的是輔助符號$\lambda$, 因為本身函數表達的從變量到值的映射按理來說其實本身就代表了自己，根本不需要使用函數名來做annotation。這也可以說我們是把 $x \mapsto x - y$ 整體當作是函數名字了。如此，我們就可以吧上面的這個f 和 g分別定義做：

$$
	f = \lambda x. x-y    \qquad \qquad g = \lambda y. x - y \\\
	\equiv  \\\
	f(0) = 0-y   \qquad \qquad  g(0) = x-0 \\\
	我們再lambda \ notation 裏面寫作：\\\
	((\lambda x.x-y) 0)   \qquad \qquad  ((\lambda y.x-y) 0) \\\
$$

使用lambda表達最大的好處主要在於描述高階函數，例如上面的 f , g 實際上可以看作是來自於 $h(x, y) = x - y$ 函數的marginal function，所以，如果我們使用lambda notation 的話，一切都會變得簡單起來。

* 首先， 映射被表示做了 $(x,y) \mapsto x - y$, 我們使用lambda symbol之後，寫作: $$\lambda xy. x - y$$，那麽再問，單參數函數和多參數函數的區分是有必要的嗎？ 如果我們使用 $\lambda xy$ 的形式的話，我們要輸入的參數就不是單變量了而是pairs，這顯然不是必要的，我們完全可以把二元（多元）函數看作是一個參數是函數的函數，於是我們把上面的函數改寫做：

$$
	x \mapsto (y \mapsto x - y)   \Rightarrow \lambda x.(\lambda y. x - y) 
$$

如果我們給這個匿名函數一個名字叫$$h^*$$, 的話，我們發現原來的 $h(x, y) \rightarrow (h^*(x))(y) $。 換句話說，我們把一個以數的函數映射成一個函數的函數，也就是我們常常說的「高階函數」（high-order function）。

我們把這種將多參數函數轉化成單參數的高階函數的過程，叫做Currying(柯裏化)。寫作 $$((\lambda xy. x - y) x y) \equiv (((\lambda x. (\lambda y. x - y)) a) b)$$.

* &lambda; Calculus 的形式化

> ($\lambda$ Terms): 

* 我們有一個無窮的字符串集合，我們叫他『變量』，同時還有一個字符串結合叫做『原子常量』，原子常量可能是有限集合，無限集合或者空集合。 但是Atomic constant只針對applied lambda calculus討論，我們目前主要考慮pure lambda calculus的話，$$Atomic\ Constant \equiv \oslash $$。So，$\lambda \ term$ 定義如下：

	+ 所有的變量 $v\_0, v\_{000}$, ... 都是lambda 項；
	+ 如果M和N是lambda 項的話，那麽（MN）也是lambda 項 （Called Application）；
	+ 如果M是lambda 項，x是任意變量，那麽 $\lambda x. M$ 也是一個lambda 項。

* 符號約定：

	+ Lambda 項是左結合的，f, x, y 都是 lambda term的話，那麽 f x y = （f x）y
	+ 如 $\lambda x. M$ 我們稱之為函數抽象，因為這個形式如果M是一個實際上對應函數的函數體，例如x+2
	+ 我們使用 小寫字母來表示lambda term中的`變量`，同時使用大寫字母（M，N，P，Q ...）來表示對應的lambda term。
	+ lambda 項都是左結合的：例如，$((((MN))P)Q)$ 我們就可以寫作 MNPQ, 此時parentheses 可以被忽略。同時，$\lambda x.$ 作為一個聲明，是不能單獨存在的。
	+ 幾個例子：

	$$
		\lambda x.\lambda y. yxab = (\lambda x. (\lambda y. (((yx)a)b))) \\\
		(\lambda x.\lambda y.yx)ab = (((\lambda x.(\lambda y.(yx)))a)b) \\\
		\lambda x.\lambda y.ab\lambda z.z = (\lambda x. (\lambda y. ((ab)\lambda z.z))) \\\
	$$

	+ 再來個Currying 定義： $\lambda x\_1x\_2...x\_n. M = (\lambda x\_1. (\lambda x\_2. (... (\lambda x\_n. M))))$。

* 語法全等：
	+ lambda terms的語法全等寫作： $M \equiv N$ 
	+ 表示的是M和N有完全相同的`結構`.  $\Rightarrow$
	+ MN  $\equiv$ PQ $\Rightarrow$ M $\equiv$ P 且 N $\equiv Q$ 
	+ 如果是function abstraction的話，那麽  $\lambda x. M \equiv \lambda y. N \Rightarrow x \equiv y  \\& \ M \equiv N  $

* 替換 &bull; preparation：
	+ lgh(M) := 所有原子在M中出現的次數。
		- lgh(a) = 1; 
		- lgh(MN) = lgh(M)+lgh(N); 
		- lgh($\lambda x. M$) = 1+lgh(M).
		- E.g $M \equiv x(\lambda y. yux) \Rightarrow lgh(M)=1+(1+3)=5$
	+ 出現(occurance):
		- P 出現於P中；
		- P 出現在M或者N中，則P出現在(MN)中；
		- P 出現在M中或者P $\equiv$ x, 那麽P出現在$\lambda x. M$中。
	+ Scope: 對於 function abstraction 來說： $\lambda x. M$ 中 $\lambda x$ 的scope 是 M
	+ 自由變量：對一個lambda term P來說，我們可以定義
		- 如果 $\lambda x. M$ 上， x出現在M中，那麽我們就說實際上M中的x是不自由的，bound。x要被從M的自由變量（FV）集合中排除。
		我麽可以對Free Variable 做如下定義：
		- FV(x) = {x}；
		- FV(MN)= FV{M} + FV{N}
		- FV($\lambda x. M$) = {M\x}.
		-  也就是說，和$\lambda$ 結合的x是binding and bound的，M中出現的x也是binding的，所以free的要把這兩者排除。
	+ E.g. 在$(((xv)(\lambda y. (\lambda z. (yv))))w)$ 在這個裏面，我們從左向右看，x:=free → v:=free → y， z:=bound and binding → y:=binding, v:=free, w:=free; So, 這個表達式中的 $FV(P)={x,v}+{y,v}-{y}+{w} = {x, v, z, w}$ 。

	+ 如果我們存在這樣的 lambda term：$P \equiv \lambda y. \lambda x. x$,  那麽我們把這個$FV(P) \equiv \oslash$ 的P叫做combinator，他是封閉的。 One more thing，一個lambda term中的free variable 和 bound variable 集合的交集不一定是空集。例如: 
	$P \equiv (\lambda y.yx(\lambda x. y(\lambda y. z) x))vw$ 中， x就即是free的，也是bound的。

* 替換 ：

有了前面的準備工作之後，我們就可以來定義subtitution了，對任意的M，N，x，我們定義$[N/x]M$ 是把M中所有的free 的x都替換成N的結果，具體的替換規則如下：

+ $ [N/x]x \equiv N $
+ $ [N/x]\alpha \equiv \alpha $
+ $ [N/x]PQ \equiv ([N/x]P)([N/x]Q) \qquad $   分配律
+ $ [N/x]\lambda x. P \equiv \lambda x. P \qquad $  因為這裏的x是bound的，而不是free的
+ $ [N/x]\lambda y. P \equiv \lambda y. P \qquad $  若 $ x \notin FV(P) $
+ $ [N/x]\lambda y. P \equiv \lambda y. [N/x]P \qquad $ 若 $ y \notin FV(N) $
+ $ [N/x]\lambda y. P \equiv \lambda z. [N/x] [z/y] P \qquad $ 若 $y \in FV(N) $, 因為這樣的話我們要防止被替換的N自己的free 變量發生變化。也就是我們不希望使用了替換之後 lambda term的控制區域發生任何變化。

試舉一例： 

$$
	[(\lambda y. xy) / x] [\lambda y. x(\lambda x. x)] \\\
	FV(N) = \left\\{x\right\\}, FV(P) = \left\\{x\right\\} + \oslash = \left\\{x\right\\},  \Rightarrow \\\
	\lambda y. [(\lambda y. xy) / x]  x(\lambda x. x) =  \\\ 
	\lambda y. ([(\lambda y. xy) / x] x)([(\lambda y. xy) / x] \lambda x. x) = \\\
	\lambda y. (\lambda y. xy)(\lambda x. x) \\\
$$

* $\alpha$ 變換和等價：

之前的subtitution 都是針對於 free variable而作的，我們還希望能夠改變bound variable， such as $x\ in\ \lambda x. M$, 設一個y，$y \notin FV(M)$
如此，我們可以得到下面這個$\alpha 全等$表達：

$$
\lambda x. M \equiv_\alpha \lambda y. [y/x]M \\
$$

這是一個「自反」，「對稱」，「傳遞」的過程(這個證明還是容易的)，自然了，這就是我們說的等價關係了。

順便，從$\alpha$ 等價關係中我們可以得到一個關於替換裏面的解釋：

$$
if  \ M \equiv\_{\alpha} M', N \equiv\_{\alpha} N', \\\
我們可以得到：[N/x]M \equiv\_{\alpha} [N'/x]M'
$$ 

從上面這個推論，證明了我們在之前尋找z替換overlapped FV的lambda term的原因了。

* $\beta$ 規約：
對於形如：$(\lambda x. M)N$的 lambda term，我們把他稱之為 $\beta$ 可約式，對應的term是：$[N/x]M$ 其實也就是說，我們可以把

$$
	(\lambda x. M)N \vartriangleright\_{1\beta} [N/x]M
$$

如果P經過有限步被規約到了Q，我們就說，$P \vartriangleright\_{\beta} Q$.

如此一來，我們就有一個很有意思的例子在這裏會出現了。。大家注意：

$$
	(\lambda x. x x)(\lambda x. x x) \vartriangleright\_{1\beta} [\lambda x. x x/x](x x) \equiv (\lambda x. x x)(\lambda x. x x)  ... \\\
$$

 似乎規約的結果和沒規約一樣, 甚至會出現了越reduce越長的效果。這是好事還是壞事呢？ 至少，我們可以瞭解到，不同的lambda term規約的結果不一定是朝著simplify的方向前進。

 1. 如果一個lambda term不含有&beta; 可規約想的話, 稱為&beta; 範式。

 2. 如果 $P \vartriangleright\_{\beta} Q$， 拿我們說Q是P的&beta; 範式。

 3. 顯然，當我們的lambda term中存在不止一個可規約項的時候，我們可能就存在多重不同的規約鏈(先後順序)，如此的花，不同的規約是否達到的是等價的結果呢？
 
 Church Rosser 定理告訴我們實際上如果P確定有範式的話，那麽我們不同規約順序得到的範式$M \equiv\_{\alpha} N$。
 
 但是我們如何確定我們使用的規約chain一定能收斂到存在的範式上呢？ 這裏我們又要做出一個定義，就是我們的P總是規約最左，最外側的&beta; 可規約式開始，我們把這種規約方式叫做`正則序`。這種方式一定能達到如果存在的&beta; 範式。

 綜上，我們現在得到了這麽兩句話：

 1. 如果我們的&lambda; term 存在一個 &beta; 範式的話，那麽這個範式是唯一的。

 2. 如果我們遵循一定的規約順序去做這件事的話(正則序)，那麽我們可以保證得到這個範式。

 從之前那兩個例子也可以看出，實際上&beta; 範式不一定存在。但是這又關係到&beta; 範式存在性的不可判斷性。這個問題暫時和我們今天討論的話題沒有太大聯係。在此不表。`：）` 主要是我覺得暫時用不到就沒細看。。。

 * $\eta $  變換(conversion):
 	+ if $x \notin FV(M)$, $\lambda x. M x = M$;
 	+ $\eta \ conversion $ 主要描述兩個lambda term之相等屬性。
 	+ 也就是說我們通過&beta; reduction 都可以得到相同(&alpha; 等價)的lambda term。   


上面就是我們介紹的所有關於Lambda Calculus 的東西了，Lambda Calculus 有很強的表達能力，按理說可以表達出所有的函數原型來。那麽我們就嘗試著一步一步來使用lambda calculus來進行各種常規的過程式計算吧。

前面我再Free Variable裏面已經提到過，FV=$\oslash$ 的項是子組合項。也就是說所有的M內的變量都和$\lambda x.$ 上的x（舉例）binding 起來。
總之，我們使用一些大寫字母表述出這些「封閉」的Lambda Term， 就叫做組合子了。

* 組合子邏輯項(combinator logic term):
	+ 首先先定義幾個算子：$I \equiv \lambda x. x $, So, $(I\ f) = f$;
	+ $K \equiv \lambda x. \lambda y. x$, So, $（K\ a）\ x = a $; 這代表constant。
	+ $S \equiv \lambda x. \lambda y. \lambda z. xz(yz)$, → $ S \ x\ y\ z = x z (y z) $
	+ 然後，我們就可以計算 (S K K) = $\lambda z. K z (K z)$, So, (S K K) x = x; 所以， (S K K) 組合出來 $I$, 但是TMD注意的是 $S K K$ 可不是 $I$ `：(` .. 顯然不可以通過 $\alpha \ transform$ 轉化嘛。。。不過他們的$\beta  \ reduction$ 的結果是一直的，這個是外延相等。


* 然後我們使用這些I, K, S, 加上變量們，組成了combinator logic term， 同時滿足 $X, Y \in \ CL-term \rightarrow (XY) \in \ CL-term$ 。
	+ CL-term 寫作  `[x].M` ：= $T[\lambda x. M]$;
	+ $[x].M = KM $ ,  If $x \notin FV(M)$;
	+ $[x].M = \lambda x. x = I $
	+ $[x].(UV) \rightarrow S([x].U)([x].V) $

* Exercise: 
$$
[x].u(vx) \rightarrow S([x].u)([x].vx) = S(Ku)(S([x].v)([x].x)) \\\ 
= S(Ku)(S(Kv)I)
$$

* Define ： $[x\_1, ... x\_n]. M = [x\_1].([x\_2].(...([x\_n].M)...)) $ 

所以，一個例子，例如我們希望能夠完成一個swap的定義：$\lambda x. \lambda y. y x = [x, y]. y x$, 所以我們來擴展這個swap 算子：

$$
	[x, y] y x \\\
	= [x]. ([y]. y x) \\\
	= [x]. (S([y]. y)([y]. x)) = [x]. (SI(Kx))  \\\
	= S([x].SI)([x].Kx) \\\
	= S(S([x].S)([x].I)) K \\\
	= S(S(KS)(KI))K
$$

我們來證明combinator 是不是具有swap之功效呢？

$$
	(S(S(KS)(KI))K)xy = \\\
	= (S(S(KS)(KI))K \ x) \ y  \\\
	= ((S(KS)(KI))x \ Kx)\ y  \\\
	= ((KS)x\ (KI)x\ Kx)\ y  \\\
	= (S I Kx) y = Iy \ Kxy = yx   
$$

所以這個swap， 成立。從上面這個例子我們也可以記憶不說嗎，CL-term只需要包含K，S，I就夠了。。其實連I都是不需要的。。。不過也無所謂了。。。`：）`。

* 基於Lambda Calculus 和 Combinator Logic Term，我們來監理我們自己的一套計算系統了：

首先就從Church Numerals開始吧，我們把一個high-order function 映射到自然數區上。對於自然數來說，我們完全可以使用 recursion definition:

1. 0 在集合N中；

2. 如果 n 在集合N中，那麽 n+1 在集合N中；

3. N是滿足1和2的最小集合。


* 按照這個模式，我們定義Church Numeral：
	+ 如果我們定義 0, ZERO CL-term的話，滿足 $ZERO\ f\ x = x$ => ZERO = $KI = [f].([x].x) = [f, x] x$。
	+ 定義 ADD-ONE, 按照自然數定義，有了0，有了 n+1 就可以了；
	+ $ADD-ONE\ ZERO\ f\ x = f(x)$; 所以是 $[n, f, x] f(n f x)$ ~ $\lambda n. \lambda f. \lambda x. f(n f x)$。
	+ 有了加一，進一步我們就可以定義 加法 PLUS：
	+ $PLUS\ m\ n\ f\ x= f(f f f ...(x))$; 我們寫成：$\lambda m. \lambda n. \lambda f. \lambda x. m f (n f x)$。
	+ $MULTI \ m\ n\ f\ x= f(f\ f\ f\ ...(x))\ {m \times n}$; 我們寫成：$\lambda m. \lambda n. \lambda f. \lambda x. m (n f) x$
	+ $POW\ m\ n 等價於\ n^m \ 次方階\ function $ ; $\lambda m. \lambda n. \lambda f. \lambda x. (m n) f x$
	+ ... 

```lisp
(define ZERO (lambda (f) (lambda (x) x)))
(define (ADD-1 n)
	(lambda (f) (lambda (x) (f ((n f) x)))))    
(define (MULTI m n)
	(lambda (f) (lambda (x) ((m (n f)) x))))

(define MULT (lambda (m) (lambda (n) (lambda (f) (lambda (x) ((m (n f)) x))))))

(define (POW m n)
	(lambda (f) (lambda (x) (((m n) f) x))))
```

Church number的存在給了我們定義recursion的方便定義了，我們只需要定義出不同的church number， N， `N f x` 就對應這 f 函數的n 次遞歸了。
如此一來的話：

我們可以這樣？：

```lisp

(define (split proc1 proc2)
            (lambda (n) ((church n) (lambda (painter) (proc2 painter painter)))))

(define zero (lambda (f) (lambda (x) x)))

(define (add-1 n)
        (lambda (f) (lambda (x) (f ((n f) x)))))
    
(define (church n)
        (cond ((= n 0) zero)
              (else (add-1 (church (- n 1))))))

(define right-split (split below beside))
(define up-split (split beside below))

```

Usage：

```lisp
	(paint ((right-split n) einstein)) ;;; n 是一個作者定義的叠代層數 這個調用發以及很接近於原來的等式了，括號僅僅是lambda的習慣而已。
```

藉助了church number，我們可以容易的定義出來每一次的` (let ((smaller (right-split painter (- n 1))))`, 但是如果我們希望把沒一層的split結果通過proc1
結合起來的話就不是那麽容易了。也就是說這種模式更適合描述$f(f .. (x))$ 這種類型的recursion，我們使用 Church Numerals的話

但是，實際上我們的這個東西是 $g(f(x), g(f(f(x)), ...))$, 如此一來的話如果我們希望引入g就不行了，因為這裏我們必須每一次都保留上一次的結果，才能不斷的recursion 下去。

那應該怎麼辦？

先解釋為什麼我們不能多f f嵌套式的遞歸，這裏我們先做幾個簡單的例子：例如，階乘的遞歸。

```lisp

(define fact (lambda (n) (if (= n 0) 1 (* n (fact (- n 1))))))

```

這個就是我們沒有辦法基於Church做遞歸的原因，前面的類似結果使用church必須對一個lambda term進行命名，這樣才能做，不過這樣的話，遇到像 我們這個split的例子就糾結了，因為我們要不斷的對內部的遞歸的f也要進行命名：

所以如果直接使用church numeral來完成 split的話，應該寫作：

```lisp
(require (planet "sicp.ss" ("soegaard" "sicp.plt" 2 1)))
(define zero (lambda (f) (lambda (x) x)))
(define (add-1 n)
        (lambda (f) (lambda (x) (f ((n f) x)))))
    
(define (church n)
        (cond ((= n 0) zero)
              (else (add-1 (church (- n 1))))))
;;; 這樣，我們就要首先定義Church numerals
;;; 然後，因為這個是一個關於recursion的高階函數，所以我們必須把兩個recursion分別寫出來
;;; 之前以及寫好了關於 proc2 的 第一個遞歸函數，所以

(define (embed-split proc)
    (lambda (n) ((church n) (lambda (painter) (proc painter painter)))))  ;;; g(x, f(g(x ..))) 內部的f 遞歸

(define split (lambda (proc1 proc2) (lambda (painter n) (if (= n 0) painter (proc1 ((split proc1 proc2) painter (- n 1)) (((embed-split proc2) n) painter)))))) ;;; 把第一個遞歸嵌套到第二個裏面

;;; 定義 各個方向的split
(define right-split (split below beside))
(define up-split (split beside below))

;;; Function Application
(paint (right-split einstein 3))
(paint (up-split einstein 3))

```

如此，就是我們只第一種完成不同方向-split的function abstraction了，但是，由於這種方法存在的問題和Factorial是一樣的，他總是要首先做一個 `(define func (lambda <??>))` 的修飾，其實這個修飾本質上就把anonymous function的意義給破壞了，雖然看上去也做到了遞歸的lambda term話，但是問題是，這個所謂的name，比如上面我定義的 『embed-split』，在lambda term的定義裏算什麽呢？這個一來不好看，而來我既然都已經新定義一個函數了，那我還何必這麽些。。我之間寫成再(define 裏面給個 define的高階函數算了。。)。所以，這個不優美的東西我還是希望把他替換掉，這樣才舒服。

於是：我們考慮到了之前做$\beta \ reduction$ 時候的幾個題目來

> 也就是不存在$\beta \ normalization$ 的項，無論如何歸於都是自身的幾個 Lambda Term：

* 首先我們先來看遞歸函數可以理解成的定義是：$f\_n = F(f\_{n-1})$，從這個地方看，正好和我們之前討論的Fix-point的$x = f(x)$ 是一致的，也就說說，我們這裏是要尋找一個函數F的不動點，正好就是我們要不斷叠代的f。 

* 關於數值函數的Fix-point:  $x = f(x)$:
	
	+ 我們如果要求一個函數的不動點的話，一般使用的就是不斷叠代的方法去逼近這個Fix-point，
	如此，我們 $|x-f(x)| \lt threshold$ 的時候，組成了一個不斷想 Fix-point 逼近的數列：
	$x, f(x), f(f(x)), ..., f(...(f(x))...)$, 顯然，上一個 $v_{n-1}$ 就是下一次計算之input，
	如此的話，這裏需找fix point的過程過程了一個不斷逼近 fix-point的遞歸函數。

	+ 當然了，實際上這種做法本身在數值函數上就是用一個recursion 過程來定義的：

```lisp
(define min-error 0.000001)

(define (fix-point func init)
	(let ((delta (abs (- init (func init)))))
	        (if (< delta min-error)
	        	  (func init)
	        	  (fix-point func (func init)))))

(fix-point (lambda (x) (/ (+ x (/ 4 x)) 2)) 1.) ;;; 求根公式例如
```

既然對於值型函數我們存在了關於x = f(x) 的不動點，那麽如果這裏的x變成是一個函數 f(x) 的話呢？顯然我們不應該再設置一個threshold了，這個東西對於函數層次似乎不合適。

先定義 $\beta \ reduction$，如果 $P \equiv\_{\beta} Q $, 那麽我們就可以說，當且僅當P可以在有限的步驟（也可以是0）內  $\vartriangleright\_{n\beta}$ 到 Q，或者是進行 $\alpha \ transform $  到Q。

如此，我們可以考慮一個Y combinator： $\lambda f. ((\lambda x. g(x\ x)) \lambda x. g(x\ x))$, 有了這個Y-combinator之後的話，我們可以發現：

$$
	YF = (\lambda f. ((\lambda x. g(x\ x)) \lambda x. g(x\ x)))F \\\
	=     (\lambda x. F(x\ x))\ (\lambda x. F(x\ x)) \\\
	= F((\lambda x. F(x\ x))\  (\lambda x. F(x\ x)) ) \\\
	= F(YF)
$$

有了上面的這個定義，就暗示了我們如果我們設計一個合理的F的話，YF 就要可以不斷的被F所嵌套產生 $F(F...(YF))$, 一旦再結合合適的終止條件分支的話，我們就可以藉助 Y combinator 完成  recursion。


$$
	g \equiv \lambda f. \lambda n. (if (= \ n\ 0) 1 (\* \ n\ (f \ (-\ n\ 1))))    \qquad \qquad 單一階乘定義 \\\
	所以我們發現 Y-combinator \ 的幫助下我們可以得到 : \\\
	Yg = g\ Yg  \\\
	所以，Yg\ n = g \ Yg \ n = (\* n\ Yg (- n \ 1)) \\\
$$



有了上面這個東西就好了， 那麽如果把上面這個過程轉換做Lisp的話：

```lisp
;;; g 單步階乘
(define g (lambda (f) (lambda (n) (if (= n 0) 1 (* n (f (- n 1)))))))

;;; Fix definition ~ (Y g) = (g (Y g)) 
(define (Fix g)
            (g (Fix g))) ;;; 結果陷入了無窮循環之中。

;;; 改成
(define (Fix g)
            (g (lambda (x) ((Fix g) x))))
    
;;; Factor
(define (factorize n)
            ((Fix g) n))

```

使用Y-combinator直接可以等價於下面這個。

```lisp

(define g (lambda (f) (lambda (n) (if (= n 0) 1 (* n (f (- n 1)))))))

(define Y
            (lambda (f) (f (lambda (x) ((Y f) x))))) ;;; 這個是對的。
```

如此我們使用了Y-combinator就可以直接定義出唔循環的遞歸了。我們把Y如果假設成是一個build-in 的procedure的話，

那麽我們可以直接定義 `factorize ` 為：

```lisp
((Y (lambda (f) (lambda (n) (if (= n 0) 1 (* n (f (- n 1))))))) 5) ;;; 計算5！

```

所以，如果我們設計一個combinator的話，我們就可以很容易的完成任何一個 二元函數的遞歸調用而不需要顯示的中間函數名了。factorize是一個例子，表現了Y-combinator對應單步函數g的遞歸能力。

基於這個想法，我們再來定義我們的split會如何呢？

```lisp
;;; 首先，我們定義Y-combinator
(define Y 
	(lambda (f) (f (lambda (x) ((Y f) x)))))

;;; 然後我們如果已有 einstein和below，beside的話
;;; 定義單步叠代
(define g
	(lambda (f) (lambda (n) (if (= n 0) einstein (beside einstein (below (f (- n 1)) (f (- n 1))))))))

;;; 遞歸調用是
(paint ((Y g) 5))

```

基於上面的實驗，我們在抽象出 split 這個高階過程。

```lisp
(require (planet "sicp.ss" ("soegaard" "sicp.plt" 2 1)))

;;; Y-combinator
(define Y 
	(lambda (f) (f (lambda (x) ((Y f) x)))))

;;;split procedure

(define (split proc1 proc2)
	(lambda (painter layer) ((Y  (lambda (f) (lambda (n) (if (= n 0) painter (proc1 painter (proc2 (f (- n 1)) (f (- n 1)))))))) layer)))
```

我們提取出painter and layer 以供不同的模式抽象。

最後，我們在Y-combinator的幫助下，完成了那層嵌套畫的遞歸

```lisp

;;; 定義 各個方向的split
(define right-split (split below beside))
(define up-split (split beside below))

;;; Function Application
(paint (right-split einstein 3))
(paint (up-split einstein 3))

```

這樣以來的話，我們就使用

```lisp
(define (split proc1 proc2)
	(lambda (painter layer) ((Y  (lambda (f) (lambda (n) (if (= n 0) painter (proc1 painter (proc2 (f (- n 1)) (f (- n 1)))))))) layer)))
```

這個告誡過程完成了遞歸函數的調用，並且如果Y-combinator可以被默認做一個build-in 的話，我就無需如之前使用church numerals似的還要給lambda函數命名了才能做一個`假的lambda 樣式的遞歸調用了`

在DrRacket中的結果：

<!--<div>
	<img src="/assets/images/2014/07/Y-combinator.png" width="600" height="400" style="center;">
</div>-->

 ![Y-combinator](/assets/images/2014/07/Y-combinator.png) 

-----------
> Conclusion:

從這個東西裏面，其他的我也說不出啥來了。。總是就是。。python 2.x 裏面的這個lambda 函數好差勁啊，完全沒有起到lambda 函數值真正作用。因為似乎，python 裏面的lambda 函數只允許嵌套一個表達式？因為在Dive into python他們是這麽說的：『总的来说，lambda 函数可以接收任意多个参数 (包括可选参数) 并且返回单个表达式的值。lambda 函数不能包含命令，包含的表达式不能超过一个。不要试图向 lambda 函数中塞入太多的东西；如果你需要更复杂的东西，应该定义一个普通函数，然后想让它多长就多长。』,如果這樣的話，python裏面的lambda就只是匿名函數而已了，這樣顯得能力遠遠不如真正的lambda 表達式。

其他的還有些問題沒想清楚。。。先這麽寫著吧暫時，這幾天再看看書，有幾個不明白的地方留得坑以後再來彌補。。。

Over

-------------
> Reference:

1. [Dive into python](http://woodpecker.org.cn/diveintopython/power_of_introspection/lambda_functions.html)

2. [recursive-lambda-expressions](http://blogs.msdn.com/b/madst/archive/2007/05/11/recursive-lambda-expressions.aspx)

3. Lambda-Calculus and Combinators, an Introduction . J. ROGER HINDLEY. Department of Mathematics, Swansea University, Wales, UK

4. [Fixed-point_combinator](http://en.wikipedia.org/wiki/Fixed-point_combinator)

5. [Lambda Calculus](http://en.wikipedia.org/wiki/Lambda_calculus)

6. [Pure Lambda Calculus](http://www.cse.iitd.ac.in/~saroj/LFP/LFP_2013/L17.pdf) 

7. [使用 Lambda 表达式编写递归](http://www.cnblogs.com/ldp615/archive/2013/04/09/recursive-lambda-expressions-1.html)

。。。 Ref 我也是瞎寫的，反正東看看西看看了。。。WTF，其實Y-combinator似乎還是最簡單的一種Fix function

