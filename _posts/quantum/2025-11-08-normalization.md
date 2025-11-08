---
title: "Born's interpretation and Normalization"
last_modified_at: 2025-11-08T18:53:12+09:00
categories:
    - quantum
tags:


author_profile: true

---
# Born의 통계적 해석
파동함수 $\Psi$ 는 무엇인가? 입자의 파동함수는 슈뢰딩거의 방정식

$$i\hbar\frac{\partial}{\partial t} \Psi (r,t) = \hat{H}\Psi (r,t)$$

을 풀어서 구하며, $\Psi(x,0)$ 이 주어지면 모든 미래 시간 t에 대해 파동함수를 구할 수 있다. 즉 이는 어떤 시간 $t$ 에서 물체의 질량, 속도, 가해지는 힘을 알면 모든 다른 시간 $t'$에서 상태를 계산할 수 있는 것과 동일하다.

Born의 통계적 해석에 의하면 파동함수의 제곱은 해당 시간 해당 위치에서 입자를 발견할 확률이다. 이때 파동함수의 제곱이란 $|\Psi(x,t)|^2$ 이며, 이는 $\Psi ^* \Psi$ 를 의미하고 $\Psi^*$ 는 $\Psi$ 의 복소켤레이다.

$(a+bi)*(a-bi) = a^2+b^2$ 이므로 $|\Psi(x,t)|^2 >0$ 이다.

# Normalization
일반적으로 확률의 합은 1이다.
$$\int_{-\inf}^{\inf} |\Psi(x,t)|^2 dx =1$$
슈뢰딩거 방정식에서 $\Psi$ 가 슈뢰딩거 방정식을 만족하면 $A\Psi$ 도 슈뢰딩거 방정식을 만족한다. 이때 확률의 합이 되어야하므로 적절한 $A$를 찾아야한다. 이를 규격화라고 한다. $t=0$에서 규격화되었을 때 모든 $t$에서 규격화된다는 증명은 생략한다(Griffiths 참조).

# 문제
시간 $t=0$ 에서 파동함수가 $A(x/a) (0 \leq x \leq a), A(b-x)/(b-a) (a\leq x \leq b), 0 (other)$ 일 때

규격화는 다음과 같이 할 수 있다.

$$\int_{0}^{a} (A(x/a))^2 dx + \int_{a}^{b} (A(b-x)/(b-a))^2 dx = 1 $$

$$ \therefore \frac{A^2a}{3} + \frac{A^2(b-a)}{3} = 1 $$

$$\therefore A = \sqrt{\frac{3}{b}} $$

$x$ 의 기댓값은 다음과 같다.

$$\int_{0}^{a} (A(x/a))^2 x dx + \int_{a}^{b} (A(b-x)/(b-a))^2 xdx = A^2 \frac{b(2a+b)}{12} = \frac{2a+b}{4} $$