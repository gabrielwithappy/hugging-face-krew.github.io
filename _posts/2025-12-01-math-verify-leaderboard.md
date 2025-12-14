---
layout: post
title: "Math-Verify로 Open LLM 리더보드 바로잡기"
author: Jimin
categories: [LLM, leaderboard]
image: assets/images/blog/posts/2025-12-01-math-verify-leaderboard/thumbnail.png
---

* TOC
{:toc}

_이 글은 Hugging Face 블로그의 [Fixing Open LLM Leaderboard with Math-Verify](https://huggingface.co/blog/math_verify_leaderboard)를 한국어로 번역한 글입니다._

---


# Math-Verify로 Open LLM 리더보드 바로잡기

3주 전, 우리는 LLM이 수학 문제를 제대로 풀었는지 정확하게 평가하는 일이 얼마나 어려운지를 보여주었고, 이를 해결할 더 나은 방법으로 [Math-Verify](https://github.com/huggingface/Math-Verify)를 소개했습니다 (자세한 내용은 [공지](https://x.com/HKydlicek/status/1881734376696041659)에서 확인할 수 있어요)!

그리고 오늘, Math-Verify를 활용해 지금까지 Open LLM Leaderboard에 제출된 **총 3,751개 모델을 전부 다시 평가**했고, 그 결과 이전보다 훨씬 더 **공정하고 신뢰할 수 있는 모델 비교가 가능해졌습니다!**



## 왜 Open LLM Leaderboard의 수학 평가는 제대로 작동하지 않았을까?

Open LLM Leaderboard는 Hugging Face Hub에서 가장 많이 활용되는 리더보드로, 다양한 태스크에서 오픈 LLM(대규모 언어 모델)의 성능을 비교합니다. 이 중 MATH-Hard는 수학 문제에 특화된 평가로, LLM이 고등학교 및 대학 수준의 수학 문제를 얼마나 정확하게 풀 수 있는지 측정합니다. 이 수학 문제들은 [Hendrycks MATH](https://github.com/hendrycks/math) 데이터셋의 최고 난이도(Level 5) 문제 중 1,324개를 뽑아, 선행 미적분(Precalculus), 기초 대수(Prealgebra), 대수(Algebra), 중급 대수(Intermediate Algebra), 조합·확률(Counting/Probability), 정수론(Number Theory) 등 7개의 주제로 구성되어 있습니다. 문제는 5-shot 방식으로 진행되며, 모델은 문제를 풀기 전에 예시 5개를 참고해 답변 형식을 학습합니다.

일반적인 문제 예시는 다음과 같습니다:

> 모든 실수 r,s에 대해 연산 # 를 다음 두 조건으로 정의한다: 
> 
> $r\ ≠ 0 = r, r\ ≠ s = s\ ≠ r$ and $(r + 1)\ ≠ s = (r\ ≠ s) + s + 1$
>   
> 이때 $11\ ≠ 5$의 값은? 

이에 대한 정답(ground truth)은 다음과 같습니다.
> 71

기존 리더보드에서는 모델이 다음과 같이 고정된 문자열로 답변을 마무리해야 합니다 ([Minerva-Math 논문](https://arxiv.org/abs/2206.14858)):

> “Final answer is [ANSWER]. I hope it is correct.”

이후 리더보드에서는 [SymPy](https://docs.sympy.org/latest/index.html)를 사용하여 `[ANSWER]`를 기호 표현으로 변환하고(필요한 경우 값을 단순화), 정답과 비교합니다.

하지만 이러한 기존 방식에는 여러 문제가 있다는 사용자들의 보고가 있었습니다.

가장 반복적으로 나타난 문제는 일부 모델이 예시에서 제시된 답변 형식을 제대로 따르지 못한다는 점입니다. 대신 답변을 소개하는 문장을 함께 출력하는 경우가 있었고, 이 때문에 실제로는 정답임에도 정해진 형식을 지키지 않았다는 이유만으로 오답 처리되는 경우가 발생했습니다.
(이는 “모델이 수학 문제를 얼마나 잘 푸는지”를 정확히 평가하고 싶을 때 큰 문제가 됩니다.)

| 📄 예시                                                                                                              | ❗️문제 유형 | ✅ Math-Verify 변환값 | 🛑 기존 리더보드 결과 |
| ------------------------------------------------------------------------------------------------------------------ | ------- | ----------------- | ------------- |
| Therefore, the perimeter of one of these triangles is $14 + 7\sqrt{2}$ inches, expressed in simplest radical form. | 추출 실패   | `7*sqrt(2) + 14`    | None          |
| Therefore, the sum of the infinite geometric series is (\frac{7}{9}).                                              | 추출 실패   | `7/9`               | None          |
| ( p(n) ) and ( p(n+1) ) share a common factor greater than 1 is (\boxed{41}).                                      | 추출 실패   | `4`                 | None          |
| So it’s \frac{1}{9}                                                                                                | 추출 실패   | `1/9`               | None          |
| Concluding he has \boxed{5} cars                                                                                   | 추출 실패   | `5`                 | None          |

다음 단계인 `[ANSWER]`를 기호 표현으로 변환하는 과정에서도 SymPy 파싱과 관련된 다양한 문제가 있었습니다: 

| 📄 예시                                                                           | ❗️문제 유형            | ✅ Math-Verify 변환값                                     | 🛑 기존 리더보드 |
| ------------------------------------------------------------------------------- | ------------------ | ----------------------------------------------------- | ---------- |
| The final answer is $2x + 4y + z - 19 = 0$. I hope it is correct.               | 매개변수 방정식 부분 파싱 실패  | Eq(2*x + 4*y + z - 19, 0)                             | 0          |
| (23)                                                                            | LaTeX 경계 문자로 추출 실패 | `23`                                                  | None       |
| ((- \infty, -14) \cup (-3, \infty)).                                            | 구간 표현 추출 실패        | Union(Interval.open(-oo, -14), Interval.open(-3, oo)) | None       |
| 100%                                                                            | 잘못된 기호로 인한 추출 실패   | `1`                                                   | None       |
| \begin{pmatrix}\frac{1}{50}&\frac{7}{50}\\frac{7}{50}&\frac{49}{50}\end{pmatrix} | 행렬 추출 실패           | Matrix([[1/50, 7/50], [7/50, 49/50]])                 | None       |


마지막으로, 추출된 답과 정답 표현을 비교하는 과정에서도 여러 문제가 발생했습니다:

| 📄 예시                      | ❗️문제 유형       | ✅ Math-Verify | 🛑 기존 리더보드 |
| -------------------------- | ------------- | ------------- | ---------- |
| 1/3 == 0.333333            | 반올림 비교 미지원    | True          | False      |
| sqrt(1/2)*7 == sqrt(0.5)*7 | 수치 평가 비교 미지원  | True          | False      |
| k = 1 == 1                 | 변수 할당 비교 미지원  | True          | False      |
| Matrix.ones == Matrix.ones | 행렬 동등성 비교 미지원 | True          | False      |
| {1} \union {1,4} == {1,4}  | 집합 비교 미지원     | True          | False      |

**새로운 Math-Verify 파서로 이 모든 문제를 해결 할 수 있습니다!**




## 어느 모델이 수학을 가장 잘할까? 공정한 평가를 통해 완전히 뒤바뀐 결과

이러한 문제들이 누적되면서, 일부 모델은 심각한 과소평가를 받았으며 실제보다 훨씬 낮은 성능으로 평가되었습니다. 그래서 우리는 기존 평가기를 제거하고 Math-Verify를 도입했으며, 이 작업은 단 3줄의 코드만 수정하면 될 정도로 간단했습니다! (여러분도 수학 평가에 쉽게 적용할 수 있습니다.)

그 결과, 6월 이후 제출된 모든 모델을 다시 평가하게 되었고, 이 과정에서 MATH 영역 상위 20위 모델들의 순위가 완전히 뒤바뀌었습니다.

### 변화의 영향

평균적으로 모델들은 기존보다 **61문제**를 더 맞혔고, 이에 따라 총점 기준으로 **4.66점** 상승이라는 큰 변화를 기록했습니다!

![](https://huggingface.co/datasets/huggingface/documentation-images/resolve/main/blog/math_verify_leaderboard/score-change.png)

가장 큰 개선을 보여준 두 개의 서브셋은 모두 대수 관련 영역(Algebra와 Prealgebra)으로, 각각 **8.27점**과 **6.93점**의 상승이 있었습니다. 극단적인 경우 일부 모델은 이 영역에서 거의 **90점** 가까이 향상되기도 했습니다. 우리는 이러한 서브셋이 여러 해답이 존재하는 문제(집합 형태)나 행렬 형식으로 답을 표현하는 문제가 많기 때문에 개선 폭이 컸다고 보고 있습니다. Math-Verify는 이러한 형식의 답변 처리 능력을 크게 향상시켰고, 그 결과 높은 점수 상승으로 이어졌습니다.

![](https://huggingface.co/datasets/huggingface/documentation-images/resolve/main/blog/math_verify_leaderboard/subset-change.png)

### 모델 패밀리별 변화

우리는 처음에 Qwen 모델을 점검하는 과정에서 수학 평가 방식의 문제를 발견했습니다. 기존 리더보드에서 Qwen 모델은 공식 성능(자체 보고 점수)보다 비정상적으로 낮은 점수를 받고 있었기 때문입니다. Math-Verify 적용 이후, Qwen 모델의 점수는 두 배 이상 증가하며 이전 평가가 얼마나 과소평가되었는지를 보여주었습니다.

하지만, 영향받은 것은 Qwen 모델만이 아닙니다. DeepSeek 모델들도 크게 개선되었습니다. Math-Verify 적용 후 DeepSeek 모델의 점수는 거의 세 배 가까이 상승했습니다! 이는 DeepSeek 모델이 정답을 일반적으로 `(\boxed{})` 표기 안에 넣어 출력하는데, 기존 평가 방식은 이 정답 형식을 제대로 추출하지 못해 실제로는 맞은 답을 오답으로 처리했기 때문입니다.

![](https://huggingface.co/datasets/huggingface/documentation-images/resolve/main/blog/math_verify_leaderboard/model-family-change.png)

### MATH-Hard 리더보드 변화

앞서 언급했듯, 상위 20개 모델 순위는 크게 변동되었으며, **Nvidia의 AceMath** 모델들이 이제 MATH-Hard 리더보드를 장악하고 있습니다. 특히 **Qwen** 파생 모델들은 거의 AceMath 바로 아래 대부분의 순위를 차지하고 있습니다. 아래는 기존 및 새로운 상위 20개 모델 리더보드 전체 비교 테이블입니다:

![](https://huggingface.co/datasets/huggingface/documentation-images/resolve/main/blog/math_verify_leaderboard/math-hard-change.png)

### 전체 리더보드 변화

마지막으로 전체 리더보드 결과가 어떻게 변했는지 살펴보았습니다. 상위 4위까지는 순위가 그대로 유지되었지만, 그 이후 순위는 상당히 큰 변동이 있었습니다. 특히 MATH 서브셋에서 Qwen 파생 모델들이 다수 상승하면서, 전체 리더보드 상위권에서도 파생 모델의 비중이 크게 늘어났습니다.

![](https://huggingface.co/datasets/huggingface/documentation-images/resolve/main/blog/math_verify_leaderboard/overal-change.png)

또한 순위가 200위 이상 상승한 모델들도 다수 존재합니다! 더 자세한 결과는 Open LLM Leaderboard에서 확인할 수 있습니다.


## 마무리

Math-Verify의 도입으로 Open LLM Leaderboard의 평가 정확성과 공정성이 크게 향상되었습니다. 덕분에 리더보드가 재편되었고, 많은 모델들이 이전보다 훨씬 정확한 성능을 보여주게 되었습니다.

저희는 모든 개발자와 연구자들에게 자신의 수학 평가에도 Math-Verify를 적용해 볼 것을 적극 권장합니다. 이를 통해 훨씬 신뢰할 수 있는 평가 결과를 얻을 수 있습니다.
또한, 업데이트된 리더보드를 확인해 관심 있는 모델이 어떻게 달라졌는지 직접 살펴보시길 바랍니다.