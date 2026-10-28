---
title: "[AI] 트랜스포머 깊게 보기 #3 — Seq2Seq와 최초의 어텐션: 문장을 벡터 하나에 담으면 생기는 일"
date: 2026-10-13 20:23:39 +0900
categories: ["AI", "논문 리뷰"]
tags: ["Transformer", "Seq2Seq", "Attention", "Bahdanau", "NLP"]
---

트랜스포머 깊게 보기 시리즈의 1부 마지막 편이다. 지난 편([#2 RNN과 LSTM](/posts/transformer-02-rnn-lstm/))에서 RNN이 문장을 한 단어씩 읽으며 은닉 상태를 갱신한다는 걸 봤다. 이번에는 그 RNN 두 개를 이어 붙여 번역기를 만들 때 무엇이 막혔고, 2014년의 어텐션이 그걸 어떻게 풀었는지를 다룬다. 전체 지도는 [Attention Is All You Need](/posts/attention-is-all-you-need/) 글에 있다.

트랜스포머의 Q, K, V가 갑자기 하늘에서 떨어진 개념이 아니라는 걸 보여주는 게 이 글의 목표다. 나는 이 연결을 알고 나서야 트랜스포머 논문의 "encoder-decoder attention"이라는 말이 왜 그렇게 당연하게 쓰였는지 이해가 됐다.

## 한 문장 정의와 요약

어텐션은 디코더가 단어를 하나 만들 때마다 인코더의 모든 은닉 상태에 점수를 매기고, 그 점수로 가중합한 벡터를 그때그때 새로 만들어 쓰는 방법이다.

기억할 건 세 가지다. 첫째, Seq2Seq는 문장 전체를 고정 길이 벡터 하나에 담아 넘겼고, 문장이 길어질수록 성능이 떨어졌다. 둘째, Bahdanau 어텐션은 벡터 하나 대신 인코더의 은닉 상태를 전부 남겨두고 디코더가 매 단계 다시 고르게 했다. 셋째, 이 "디코더가 묻고 인코더가 답하는" 구조가 트랜스포머에서 Query와 Key·Value로 이름만 바뀌어 그대로 살아남았다.

## 왜 필요했나: 벡터 하나에 문장 전체를

### Seq2Seq의 구조

2014년 구글 연구진이 발표한 Seq2Seq[1]는 구조가 단순하다. RNN(정확히는 LSTM) 하나가 원문을 끝까지 읽고, 마지막 은닉 상태를 다른 RNN에게 넘긴다. 두 번째 RNN은 그 벡터 하나만 들고 번역문을 한 단어씩 만든다.

```
인코더 (원문을 읽는 RNN)                         디코더 (번역문을 만드는 RNN)

 I ─▶ [h1] ─▶ love ─▶ [h2] ─▶ you ─▶ [h3] ═══▶ c ═══▶ [s1] ─▶ [s2] ─▶ [s3]
                                                        │       │       │
                                    벡터 하나로 전부 전달  나는     너를    사랑해
```

여기서 `c`를 context vector라고 부른다. 길이가 고정이다. 원문이 3단어든 50단어든 같은 크기의 벡터 하나에 담겨야 한다.

### 길어지면 무너진다

같은 해 나온 인코더-디코더 번역 실험 논문[2]은 이 구조의 성능이 입력 문장이 길어질수록 빠르게 떨어진다는 걸 실험으로 보였다. 이유는 지난 편에서 본 RNN의 성질과 같다. 은닉 상태는 새 단어가 들어올 때마다 덮어쓰이니, 앞쪽 단어의 흔적은 갈수록 옅어진다. 그 옅어진 벡터 하나가 디코더가 가진 유일한 정보다.

얼마나 옅어지는지 직접 확인해봤다. 학습하지 않은 작은 RNN(무작위 가중치, 은닉 8차원)에 첫 단어만 다른 두 문장을 넣고, 마지막 은닉 상태가 얼마나 다른지 쟀다.

```python
import math, random

random.seed(7)
D = 8
W = [[random.gauss(0, 0.35) for _ in range(D)] for _ in range(D)]   # 은닉→은닉
emb = {}
def vec(w):
    if w not in emb:
        emb[w] = [random.gauss(0, 1) for _ in range(D)]
    return emb[w]

def encode(sentence):
    h = [0.0] * D
    for w in sentence:
        x = vec(w)
        h = [math.tanh(sum(W[i][j] * h[j] for j in range(D)) + x[i]) for i in range(D)]
    return h

def dist(a, b):
    return math.sqrt(sum((x - y) ** 2 for x, y in zip(a, b)))

filler = [f"w{i}" for i in range(60)]
print("문장 길이 | 첫 단어만 다른 두 문장의 최종 벡터 거리")
for n in (1, 3, 5, 10, 20, 40):
    a = ["고양이"] + filler[:n - 1]
    b = ["강아지"] + filler[:n - 1]
    print(f"{n:>8} | {dist(encode(a), encode(b)):.6f}")
```

실행 결과는 이렇다.

```
문장 길이 | 첫 단어만 다른 두 문장의 최종 벡터 거리
       1 | 1.045655
       3 | 0.247891
       5 | 0.054583
      10 | 0.001526
      20 | 0.000000
      40 | 0.000000
```

한 단어짜리 문장에서는 "고양이"와 "강아지"가 거리 1.05만큼 떨어진 벡터가 된다. 열 단어가 되면 0.0015로 줄고, 스무 단어부터는 소수점 여섯째 자리까지 0이다. 마지막 벡터만 봐서는 첫 단어가 고양이였는지 강아지였는지 구분할 수 없다는 뜻이다.

물론 이건 학습하지 않은 RNN이라 과장된 결과다. 학습된 LSTM은 게이트로 정보를 훨씬 오래 붙잡는다. 그래도 "새 입력이 옛 흔적을 덮는다"는 방향은 같고, 그게 [2]에서 본 성능 저하의 정체다. Seq2Seq 논문[1]이 원문을 거꾸로 뒤집어 넣는 요령을 썼던 것도 이 때문이다. 원문 첫 단어를 디코더 첫 단어 가까이 두면 거리가 줄어든다. 흥미로운 건 [1]은 이 요령과 4층짜리 깊은 LSTM으로 긴 문장도 잘 번역했다고 보고했다는 점이다. 반면 [2]의 실험에서는 길이가 늘수록 성능이 빠르게 떨어졌다. 모델을 크게 키우고 요령을 써야 겨우 버틸 수 있는 구조였다는 뜻이고, Bahdanau 어텐션 논문[3]은 그 원인을 고정 길이 벡터라는 병목으로 짚었다.

## 작동 원리: Bahdanau 어텐션

2014년 어텐션 논문[3]의 아이디어는 한 줄로 요약된다. 벡터 하나에 담지 말고, 인코더 은닉 상태를 전부 남겨두자. 그리고 디코더가 단어를 만들 때마다 그중 필요한 걸 골라 쓰게 하자.

![문장 전체를 벡터 하나에 담는 Seq2Seq와 매 단계 다시 보는 어텐션](/assets/img/posts/transformer-03-seq2seq-attention/01.gif)
_문장 전체를 벡터 하나에 담는 Seq2Seq와 매 단계 다시 보는 어텐션_

### 세 단계로 쪼개기

디코더가 i번째 단어를 만들려는 순간을 보자. 직전 디코더 상태를 `s(i-1)`, 인코더의 j번째 은닉 상태를 `h(j)`라고 하면 계산은 세 단계다.

```
① 점수      e(i,j) = vᵀ · tanh( W·s(i-1) + U·h(j) )     모든 j에 대해
② 가중치    α(i,j) = exp(e(i,j)) / Σₖ exp(e(i,k))        softmax
③ context   c(i)   = Σⱼ α(i,j) · h(j)                    가중합
```

①은 "지금 디코더 상태와 j번째 원문 단어가 얼마나 관련 있나"를 숫자 하나로 만든다. 두 벡터를 각각 행렬로 변환해 더하고, tanh를 거쳐 벡터 `v`와 내적한다. 작은 신경망 하나를 통과시키는 셈이라 additive(덧셈형) 어텐션이라고 부른다. ②는 점수를 합이 1인 비율로 바꾼다. ③은 그 비율대로 은닉 상태를 섞어 이번 단계 전용 context를 만든다.

Seq2Seq의 `c`는 문장마다 하나였다. 어텐션의 `c(i)`는 디코더 단계마다 하나씩, 매번 새로 만들어진다. 이게 핵심 차이다.

참고로 Bahdanau 논문의 인코더는 양방향 RNN이다. 앞에서 읽은 은닉 상태와 뒤에서 읽은 은닉 상태를 이어 붙여 `h(j)`로 쓴다. 그래서 `h(j)`는 j번째 단어 주변의 앞뒤 문맥을 함께 담는다.

### 왜 하나를 고르지 않고 섞나

자연스러운 질문이 하나 있다. 어차피 "사랑해"는 "love"에서 나오는데, 가장 점수가 높은 은닉 상태 하나만 고르면 되지 않나?

고르는 연산(argmax)은 미분이 안 된다. 점수가 조금 바뀌어도 선택 결과는 그대로이다가 어느 순간 뚝 바뀐다. 기울기가 0이거나 정의되지 않으니 역전파로 W, U, v를 학습할 수 없다.

softmax 가중합은 다르다. 점수 e가 조금 오르면 α가 조금 오르고, c가 그 방향으로 조금 움직인다. 모든 단계가 매끄럽게 이어져 있으니 번역 오차를 거슬러 올라가 "점수를 어떻게 매겨야 했나"까지 학습할 수 있다. 그래서 정렬(어떤 원문 단어가 어떤 번역 단어에 대응하는지)을 사람이 알려주지 않아도, 번역만 잘하도록 학습시키면 정렬이 저절로 생긴다. 논문 제목이 "Jointly Learning to Align and Translate"인 이유다.

### 정렬 행렬

각 디코더 단계의 가중치 α를 한 행씩 쌓으면 정렬 행렬이 된다.

![번역하며 집중 위치가 옮겨가는 정렬 행렬](/assets/img/posts/transformer-03-seq2seq-attention/02.gif)
_번역하며 집중 위치가 옮겨가는 정렬 행렬 (설명용 예시값)_

| 만드는 단어 \ 원문 | I | love | you |
|---|---|---|---|
| 나는 | 0.85 | 0.05 | 0.10 |
| 너를 | 0.08 | 0.12 | 0.80 |
| 사랑해 | 0.05 | 0.85 | 0.10 |

(설명용 예시값이다. 각 행의 합은 1이다.)

영어와 한국어는 어순이 달라서 정렬이 대각선을 따라가지 않는다. "너를"은 세 번째 원문 단어 you를, "사랑해"는 두 번째 love를 본다. Bahdanau 논문에도 영어-프랑스어 번역의 정렬 행렬 그림이 실려 있는데, 형용사와 명사 순서가 뒤바뀌는 구간에서 대각선이 꺾이는 모습이 보인다. 개인적으로 이 논문에서 가장 설득력 있는 건 수식이 아니라 이 그림이라고 생각한다. 모델이 "무엇을 보고 이 단어를 만들었는지"를 처음으로 눈으로 확인할 수 있게 됐기 때문이다.

## 실제로 계산해보기: additive vs dot

1년 뒤 나온 후속 논문[4]은 점수를 매기는 더 간단한 방법들을 비교했다. 그중 가장 단순한 게 디코더 상태와 인코더 상태를 그냥 내적하는 dot 방식이다. 두 방식을 같은 입력으로 계산해봤다.

상황은 "I love you"를 번역하다가 "나는 너를"까지 만들고 "사랑해"를 고르려는 순간이다. 은닉 상태는 2차원으로 줄였고 값은 직접 정했다.

```python
import math

def softmax(xs):
    m = max(xs)
    e = [math.exp(x - m) for x in xs]
    s = sum(e)
    return [x / s for x in e]

def matvec(W, x):
    return [sum(w * v for w, v in zip(row, x)) for row in W]

def dot(a, b):
    return sum(x * y for x, y in zip(a, b))

# 인코더 은닉 상태 (원문 "I love you")
words = ["I", "love", "you"]
H = [[1.0, 0.2],   # h_I
     [0.1, 1.0],   # h_love
     [0.6, 0.5]]   # h_you

# 디코더 상태: "나는 너를"까지 만들고 "사랑해"를 고르려는 순간
s = [0.2, 1.2]

# 1) Bahdanau(additive): score = v · tanh(W1·h + W2·s)
W1 = [[0.5, -0.4], [-0.6, 1.2]]
W2 = [[0.3,  0.2], [-0.5, 0.4]]
v  = [0.4, 1.5]
add = [dot(v, [math.tanh(a + b) for a, b in zip(matvec(W1, h), matvec(W2, s))]) for h in H]

# 2) Luong(dot): score = s · h
dt = [dot(s, h) for h in H]

for name, scores in (("additive", add), ("dot", dt)):
    a = softmax(scores)
    c = [sum(a[i] * H[i][j] for i in range(3)) for j in range(2)]
    print(f"[{name}]")
    print("  score  :", [round(x, 3) for x in scores])
    print("  weight :", dict(zip(words, [round(x, 3) for x in a])))
    print("  context:", [round(x, 3) for x in c])

# 고정 길이 벡터 하나(마지막 은닉 상태)만 쓸 때의 context
print("[seq2seq] context = h_you =", H[-1])
```

실행 결과다.

```
[additive]
  score  : [0.277, 1.343, 0.979]
  weight : {'I': 0.169, 'love': 0.49, 'you': 0.341}
  context: [0.422, 0.695]
[dot]
  score  : [0.44, 1.22, 0.72]
  weight : {'I': 0.222, 'love': 0.484, 'you': 0.294}
  context: [0.447, 0.676]
[seq2seq] context = h_you = [0.6, 0.5]
```

두 방식 모두 love에 가장 큰 가중치(0.49, 0.484)를 줬고, context는 `h_love = [0.1, 1.0]` 쪽으로 끌려갔다. 반면 Seq2Seq는 어느 단계에서든 마지막 은닉 상태 `[0.6, 0.5]` 하나뿐이다. "사랑해"를 만들 때도, "나는"을 만들 때도 같은 벡터다.

additive 쪽 W1, W2, v는 내가 임의로 정한 값이다. 실제로는 학습으로 정해진다. 그래서 이 예제의 숫자보다는 "dot은 학습할 파라미터 없이 내적 한 번으로 비슷한 일을 한다"는 점을 보면 된다. 두 방식의 차이를 정리하면 이렇다.

| 항목 | Bahdanau (2014) | Luong (2015) |
|---|---|---|
| 점수 함수 | additive: `vᵀ tanh(W·s + U·h)` | dot `sᵀh`, general `sᵀWh`, concat 비교 |
| 디코더 상태 | 직전 상태 `s(i-1)` | 현재 상태 `h(t)` |
| 계산 비용 | 작은 신경망 통과 | dot은 내적 한 번 |
| 범위 | 원문 전체 | 전체(global)와 일부 창(local) 둘 다 제안 |

dot 방식은 행렬 곱 한 번으로 모든 점수를 동시에 계산할 수 있어서 빠르다. 트랜스포머가 고른 것도 이쪽이다. 거기에 √d_k로 나누는 스케일링을 더한 게 Scaled Dot-Product Attention이다.

## 트랜스포머로 이어지는 다리

이제 트랜스포머의 용어와 1:1로 맞춰볼 수 있다.

| Bahdanau / Luong 어텐션 | 트랜스포머 |
|---|---|
| 디코더 상태 `s` ("지금 무엇이 필요한가") | Query |
| 인코더 은닉 상태 `h(j)` (점수를 매길 대상) | Key |
| 인코더 은닉 상태 `h(j)` (섞어서 가져올 내용) | Value |
| 점수 → softmax → 가중합 | `softmax(QKᵀ/√d_k)V` |

차이는 두 가지다. 첫째, 예전에는 같은 `h(j)`를 Key와 Value로 함께 썼다. 트랜스포머는 `W_K`, `W_V`로 각각 변환해서 "점수 매기는 용도"와 "가져올 내용"을 분리했다. 둘째, 예전 어텐션은 디코더가 인코더를 볼 때만 썼다. 트랜스포머는 같은 문장 안에서 단어들이 서로를 보는 데도 썼다. 이게 셀프 어텐션이고, 그 덕분에 RNN 자체를 버릴 수 있었다.

트랜스포머 디코더의 "encoder-decoder attention"(Q는 디코더, K·V는 인코더)은 Bahdanau 어텐션을 거의 그대로 옮긴 것이다. 처음 트랜스포머 그림을 보면 낯설지만, 이 다리를 건너고 나면 바뀐 건 RNN이 빠진 것과 점수 함수뿐이다.

## 흔한 오해

어텐션이 트랜스포머에서 처음 나왔다는 오해가 가장 흔하다. 이 글의 주인공인 Bahdanau 어텐션이 2014년이고, 트랜스포머는 그로부터 3년 뒤다. 트랜스포머의 기여는 어텐션의 발명이 아니라 RNN을 빼고 어텐션만으로 모델을 만든 데 있다.

어텐션이 붙은 Seq2Seq가 RNN의 문제를 다 풀었다고 보는 것도 정확하지 않다. 정보 병목은 풀었지만, 인코더와 디코더는 여전히 RNN이다. 단어를 하나씩 순서대로 처리해야 한다는 속도 문제는 그대로 남았다. 이걸 푼 게 트랜스포머다.

정렬 가중치를 "정답 단어 대응표"로 읽는 것도 조심해야 한다. 가중치는 사람이 표시한 정답이 아니라 번역을 잘하도록 학습된 부산물이다. 한 단어가 원문 두세 단어에 나눠 집중하는 경우도 흔하고, 관사나 조사처럼 대응하는 단어가 없는 경우에는 엉뚱한 곳에 가중치가 퍼지기도 한다. 해석의 힌트로는 좋지만 근거로 쓰기는 어렵다.

## 요약

1부 세 편을 한 줄로 이으면 이렇다. 단어를 벡터로 바꿨고(#1), 그 벡터를 RNN으로 순서대로 읽었고(#2), 읽은 결과를 벡터 하나에 담다가 막혀서 어텐션으로 필요한 걸 매번 다시 골라 쓰게 됐다(#3).

이번 편의 핵심은 두 가지다. Seq2Seq의 고정 길이 context는 문장이 길어질수록 앞쪽 정보를 잃는다. 어텐션은 인코더 은닉 상태를 전부 남겨두고, 디코더 단계마다 점수 → softmax → 가중합으로 전용 context를 만든다. 그리고 이 구조가 트랜스포머에서 Query, Key, Value가 됐다.

2부에서는 트랜스포머 안으로 들어간다. 다음 편은 #4 셀프 어텐션과 Q·K·V 행렬 계산이다. 이번 편에서 단어 하나씩 계산했던 점수를 문장 전체에 대해 행렬 곱 한 번으로 계산하는 과정을 다룬다.

## 학습 확인 질문

1. Seq2Seq의 context vector가 긴 문장에서 문제가 되는 이유는 무엇인가?

   답: 원문 길이와 상관없이 고정 길이 벡터 하나에 문장 전체를 담아야 하기 때문이다. 인코더 RNN은 새 단어가 들어올 때마다 은닉 상태를 덮어쓰므로 앞쪽 단어의 정보가 점점 옅어지고, 디코더는 그 벡터 하나만 볼 수 있다.

2. Bahdanau 어텐션이 가장 점수가 높은 은닉 상태 하나를 고르지 않고 softmax 가중합을 쓰는 이유는?

   답: 하나를 고르는 연산은 미분할 수 없어 역전파로 점수 함수를 학습할 수 없다. softmax 가중합은 점수 변화가 context 변화로 매끄럽게 이어지므로, 정렬 정답 없이 번역 오차만으로 어디에 집중할지를 학습할 수 있다.

3. 트랜스포머의 encoder-decoder attention에서 Query, Key, Value는 각각 어디서 오고, Bahdanau 어텐션의 무엇에 해당하나?

   답: Query는 디코더에서 오고 Bahdanau의 디코더 상태 s에 해당한다. Key와 Value는 인코더 출력에서 오고 Bahdanau의 인코더 은닉 상태 h(j)에 해당한다. 다만 트랜스포머는 같은 인코더 출력을 W_K, W_V로 따로 변환해 Key와 Value를 분리한다.

## 참고 자료

- [1] [Sequence to Sequence Learning with Neural Networks (Sutskever et al., 2014)](https://arxiv.org/abs/1409.3215)
- [2] [On the Properties of Neural Machine Translation: Encoder-Decoder Approaches (Cho et al., 2014)](https://arxiv.org/abs/1409.1259)
- [3] [Neural Machine Translation by Jointly Learning to Align and Translate (Bahdanau et al., 2014)](https://arxiv.org/abs/1409.0473)
- [4] [Effective Approaches to Attention-based Neural Machine Translation (Luong et al., 2015)](https://arxiv.org/abs/1508.04025)
- [5] [Visualizing A Neural Machine Translation Model (Jay Alammar)](https://jalammar.github.io/visualizing-neural-machine-translation-mechanics-of-seq2seq-models-with-attention/)
