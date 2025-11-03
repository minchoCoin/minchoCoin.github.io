---
title: "Schrödinger equation"
last_modified_at: 2025-11-03T18:53:12+09:00
categories:
    - quantum
tags:


author_profile: true

---
# Schrödinger equation
$$i\hbar\frac{\partial}{\partial t} \Psi (r,t) = \hat{H}\Psi (r,t)$$

이때 $\hat{H}$ 는 해밀토니안 연산자(Hamiltonian operator)로서 전체 에너지(즉 운동에너지와 퍼텐셜 에너지의 합)를 나타냄

## Heuristic derivation
고전역학에서 에너지는 운동에너지와 위치에너지의 합이다.

$$ E=\frac{p^2}{2m} + V$$

운동량 p는 양자역학에서 파동수(공간 변화에 따른 위상 변화율) $k$와 관련있다. $p=\hbar k$

에너지 E는 양자역학에서 진동수(시간 변화에 따른 위상 변화율) $\omega$와 관련 있다. $E=\hbar \omega$

파동함수의 기본 형태는 다음과 같다

$$\Psi (x,t) = e^{i(kx-\omega t)}$$

위상이 일정한 점은 $kx-\omega t=constant$에서 해당 위치 x는 $x=\frac{\omega}{k}t + constant$ 로서 오른쪽으로 이동한다.

이제 파동함수를 t와 x에 대해 각각 미분해보면

$$ \frac{\partial}{\partial t}\Psi = -i\omega e^{i(kx-\omega t)} = -\frac{iE}{\hbar}\Psi(\because E=\hbar \omega)$$

$$ \therefore i\hbar \frac{\partial}{\partial t} \Psi = E\Psi$$

$$ \frac{\partial}{\partial x}\Psi = -ik e^{i(kx-\omega t)} = -\frac{ip}{\hbar}\Psi(\because p=\hbar k)$$

$$ \therefore -i\hbar \frac{\partial}{\partial x} \Psi = p\Psi$$

따라서 E와 p를 파동함수로 치환하면

$$ i\hbar \frac{\partial}{\partial t}\Psi = (-\frac{\hbar ^2}{2m}\frac{\partial^2}{\partial x^2} + V)\Psi$$

따라서 슈뢰딩거 방정식 $$i\hbar\frac{\partial}{\partial t} \Psi (r,t) = \hat{H}\Psi (r,t)$$ 이 성립한다.

운동량 $p=\hbar k$를 가지는 입자의 파동 $\Psi(x) = e^{ikx}$ 에서 운동 에너지는 $$-\frac{\hbar ^2}{2m}\frac{\partial^2}{\partial x^2} e^{ikx} = -\frac{\hbar ^2}{2m} (-k^2) e^{ikx} = \frac{\hbar ^2 k^2}{2m}e^{ikx}$$ 로서, 고전 운동에너지 식 $E=\frac{p^2}{2m}$ 과 일치한다.