---
lang: ko
translation_key: casting-call-to-search-conditions
title: "캐스팅 공고가 검색 조건이 되기까지"
description: "Castar가 자유 형식의 캐스팅 공고를 LLM 호출 한 번으로 읽어 엄격한 필터와 우대 조건으로 나누는 방법, 그리고 맞지 않는 배우를 보여 주느니 차라리 아무도 보여 주지 않는 이유."
summary: "LLM 호출 한 번으로 자유 형식의 캐스팅 공고를 엄격한 필터와 우대 조건으로 나눕니다. 조건에 맞는 배우가 없으면 조건을 완화하지 않고, 없다고 말합니다."
hero: /assets/posts/from-a-casting-call-to-search-conditions/cover.png
hero_width: 1200
hero_height: 630
image: /assets/posts/from-a-casting-call-to-search-conditions/cover.png
---

캐스팅 공고는 사람이 읽으라고 쓴 글입니다. 흔히 이런 식입니다. “남자 조연, 40대 초반~50대 초반. 영어 능통자 필수. 키 178cm 이상. 수영 가능자 우대. 따뜻하지만 세월이 느껴지는 얼굴, 짧은 머리.”

배우 프로필은 데이터베이스가 읽으라고 만든 것입니다. Castar 프로필에는 성별, 출생 연도, 키, 몸무게, 그리고 정해진 목록에서 고르는 특기 같은 몇 가지 구조화된 필드가 있고, 사진이 있습니다.

검색은 그 둘 사이에 있습니다. 이 글에서는 Castar가 공고를 프로필 데이터에 바로 적용할 수 있는 검색 조건으로 바꾸는 과정과, 그 과정에서 내린 결정들을 설명합니다. 그중 가장 중요한 결정은 정직한 답이 "해당하는 사람이 없다"일 때 시스템이 어떻게 해야 하느냐에 관한 것입니다.

이 글에 나오는 캐스팅 공고, 배역, 출력은 모두 설명을 위해 만든 가상의 예시입니다.

## 두 종류의 조건

캐스팅 공고를 찬찬히 읽어 보면 조건이 두 부류로 나뉩니다.

**기호적 조건**은 프로필 필드에 대응합니다. "남자"는 성별 값이고, "40대 초반~50대 초반"은 나이의 범위이고, 나이는 출생 연도로 계산할 수 있습니다. "키 178cm 이상"은 키의 하한이고, "영어 능통"과 "수영"은 특기 목록의 항목입니다. 이런 조건은 프로필이 만족하거나 만족하지 않거나, 둘 중 하나입니다.

**의미적 조건**에는 대응하는 필드가 없습니다. "따뜻하지만 세월이 느껴지는 얼굴", "선이 날카로운 이목구비", "평생 바깥일을 해 온 것 같은 인상"은 외모와 인상에 대한 묘사이고, 이 정보는 사진에만 있습니다. 배우가 얼마나 잘 맞는지는 정도의 문제이고, 어느 정도는 취향의 문제이기도 합니다.

두 부류는 일부러 따로 다룹니다. 틀리는 방식이 다르기 때문입니다. 기호적 조건이 잘못 매칭되면 누가 봐도 틀린 결과입니다. "50대"로 쓴 배역에 21세 배우가 나오는 식이죠. 반면 의미적 조건의 매칭은 언제나 이견의 여지가 있습니다. 둘을 하나의 유사도 점수로 합치면 한쪽이 다른 쪽을 메울 수 있게 되고, 외모가 아주 잘 맞는 배우가 나이 조건을 만족하지 못하는데도 조건을 만족하는 배우보다 위로 올라갈 수 있습니다. 둘을 분리해 두면 기호적 조건은 정확하게, 의미적 조건은 정도에 따라 다룰 수 있습니다.

이 글은 기호적 조건을 다룹니다. 외모는 별도의 이미지–텍스트 채널로 처리하며, 따로 글을 쓸 예정입니다.

## 호출 한 번, 작은 JSON 하나

<figure class="figure figure--pipeline" role="group" aria-labelledby="fig1-cap">
  <div class="figure__scroll" tabindex="0" role="region" aria-label="그림 1. 작은 화면에서는 가로로 스크롤됩니다">
    <img src="{{ '/assets/posts/from-a-casting-call-to-search-conditions/pipeline.svg' | relative_url }}" width="700" height="640" alt="가상의 캐스팅 공고 “남자 조연, 40대 초반~50대 초반. 영어 능통자 필수. 키 178cm 이상. 수영 가능자 우대. 따뜻하지만 세월이 느껴지는 얼굴, 짧은 머리.”를 처리하는 파이프라인(그림 속 문구는 영어). LLM 호출 한 번이 JSON을 출력합니다. 기호적 갈래에서는 구조화된 필드(성별 = 남성, 나이 40–53, 키 ≥ 178, 영어 필수, 수영 우대)가 나와 엄격한 필터로 적용됩니다. 의미적 갈래에서는 외모 텍스트 “따뜻하고 세월이 느껴지는 얼굴의 남성, 짧은 머리”가 나와 공유 이미지–텍스트 공간에서 사진과 매칭되고 순위를 정하는 데 쓰입니다. 두 갈래가 합쳐져 후보 목록이 됩니다.">
  </div>
  <figcaption id="fig1-cap"><b>그림 1.</b> LLM 호출 한 번으로 캐스팅 공고를 두 갈래로 나눕니다. 기호적 조건은 구조화된 프로필 필드에 대한 엄격한 필터가 되고, 짧은 외모 묘사는 사진과 매칭되어 순위를 정하는 데 쓰입니다. <a href="{{ '/assets/posts/from-a-casting-call-to-search-conditions/pipeline.png' | relative_url }}">원본 크기로 보기</a>.</figcaption>
</figure>

캐스팅 공고 하나당 언어 모델을 한 번 호출합니다. 프롬프트에는 공고 본문과 함께 아래에서 설명할 규칙들, 그리고 플랫폼의 특기 어휘 목록이 들어가고, 모델에게 간결한 JSON 객체 하나를 요청합니다. 이 글에서는 필드 이름을 단순화했습니다. 객체는 다음과 같이 구성됩니다.

<div class="table-wrap" markdown="1">

| 필드 | 들어가는 내용 |
|---|---|
| `gender` | `"male"`, `"female"`, 공고에 언급이 없으면 `null` |
| `age` | 숫자 범위 `{min, max}` 또는 `null` |
| `height_cm` | 숫자 경계값. 공고에 숫자가 있을 때만 |
| `skills.required` | 플랫폼 특기 어휘에서 고른 라벨 |
| `skills.preferred` | 같은 어휘에서 고른 라벨 |
| `appearance` | 이미지 채널로 보낼 외모와 인상에 대한 짧은 묘사 |

</div>

공고에 적혀 있지 않은 것은 `null`이나 빈 값으로 둡니다. 모델에게는 본문이 뒷받침할 때만 필드를 채우라고 지시합니다. 필터가 빠지면 아무도 배제되지 않지만, 추측으로 채운 필터는 사람들을 조용히 배제합니다.

나이, 특기, 외모를 각각 따로 호출해 이어 붙이지 않고, 호출 한 번으로 처리합니다. 호출 횟수가 늘면 지연 시간과 비용이 늘고, 호출 하나하나가 타임아웃이나 깨진 응답이 생길 수 있는 지점이 됩니다. 각 부분이 서로 얽혀 있기도 합니다. "수영"이 필수인지 우대인지는 앞뒤 문장에 달려 있는데, 한 번에 처리하면 공고 전체를 한꺼번에 볼 수 있습니다. 대신 프롬프트가 길어지고, 그 프롬프트를 한 덩어리로 관리해야 한다는 비용이 있습니다.

이제 필드를 하나씩 살펴보겠습니다.

### 성별

성별은 공고에 명시되어 있거나, "아버지", "할머니"처럼 배역을 가리키는 단어 자체에 성별이 담겨 있을 때만 채웁니다. 그 외에는 `null`로 둡니다. "형사"처럼 직업만 있는 경우는 해당하지 않습니다.

### 나이: 연령대 표현을 숫자로

캐스팅 공고는 보통 나이를 연령대로 적고, 이를 초반, 중반, 후반으로 나눕니다. 각 구간은 끝자리 범위로 고정해 대응시킵니다. 초반은 X0–X3, 중반은 X4–X6, 후반은 X7–X9입니다.

<div class="table-wrap table-wrap--nums" markdown="1">

| 표현 | 범위 |
|---|---|
| 40대 | 40–49 |
| 40대 초반 | 40–43 |
| 40대 중반 | 44–46 |
| 40대 후반 | 47–49 |
| 40대 초반~50대 초반 | 40–53 |

</div>

마지막 행처럼 구간으로 적힌 경우에는 앞 표현의 하한과 뒤 표현의 상한을 씁니다.

<figure class="figure" role="group" aria-labelledby="fig2-cap">
  <div class="figure__frame">
    <svg class="fig-capped" viewBox="0 0 400 190" role="img" aria-labelledby="fig2-title fig2-desc">
      <title id="fig2-title">“40대 초반~50대 초반”을 40세부터 53세로 대응시키기</title>
      <desc id="fig2-desc">40부터 49, 50부터 59까지 각각 열 칸씩 두 줄. 각 연령대는 초반(0–3), 중반(4–6), 후반(7–9)으로 나뉩니다. 40세부터 53세까지, 즉 40대 전체와 50대 초반이 강조되어 있습니다.</desc>
      <g font-size="14" font-weight="600" fill="#f2f1ec">
        <text x="4" y="52">40대</text>
        <text x="4" y="132">50대</text>
      </g>
      <!-- 40s row: all in range -->
      <g fill="#2b2b29" stroke="#ffffff" stroke-width="1.1">
        <rect x="52" y="30" width="32" height="34" rx="4"/>
        <rect x="86" y="30" width="32" height="34" rx="4"/>
        <rect x="120" y="30" width="32" height="34" rx="4"/>
        <rect x="154" y="30" width="32" height="34" rx="4"/>
        <rect x="188" y="30" width="32" height="34" rx="4"/>
        <rect x="222" y="30" width="32" height="34" rx="4"/>
        <rect x="256" y="30" width="32" height="34" rx="4"/>
        <rect x="290" y="30" width="32" height="34" rx="4"/>
        <rect x="324" y="30" width="32" height="34" rx="4"/>
        <rect x="358" y="30" width="32" height="34" rx="4"/>
        <!-- 50s row: 50–53 in range -->
        <rect x="52" y="110" width="32" height="34" rx="4"/>
        <rect x="86" y="110" width="32" height="34" rx="4"/>
        <rect x="120" y="110" width="32" height="34" rx="4"/>
        <rect x="154" y="110" width="32" height="34" rx="4"/>
      </g>
      <g fill="#161616" stroke="#3d3d3a" stroke-width="1.1">
        <rect x="188" y="110" width="32" height="34" rx="4"/>
        <rect x="222" y="110" width="32" height="34" rx="4"/>
        <rect x="256" y="110" width="32" height="34" rx="4"/>
        <rect x="290" y="110" width="32" height="34" rx="4"/>
        <rect x="324" y="110" width="32" height="34" rx="4"/>
        <rect x="358" y="110" width="32" height="34" rx="4"/>
      </g>
      <g font-family="'IBM Plex Mono', ui-monospace, monospace" font-size="13" text-anchor="middle" fill="#ffffff">
        <text x="68" y="52">40</text><text x="102" y="52">41</text><text x="136" y="52">42</text><text x="170" y="52">43</text><text x="204" y="52">44</text>
        <text x="238" y="52">45</text><text x="272" y="52">46</text><text x="306" y="52">47</text><text x="340" y="52">48</text><text x="374" y="52">49</text>
        <text x="68" y="132">50</text><text x="102" y="132">51</text><text x="136" y="132">52</text><text x="170" y="132">53</text>
      </g>
      <g font-family="'IBM Plex Mono', ui-monospace, monospace" font-size="13" text-anchor="middle" fill="#8e8e88">
        <text x="204" y="132">54</text><text x="238" y="132">55</text><text x="272" y="132">56</text><text x="306" y="132">57</text><text x="340" y="132">58</text><text x="374" y="132">59</text>
      </g>
      <!-- brackets -->
      <g fill="none" stroke="#8e8e88" stroke-width="1.1">
        <path d="M54,70 V75 H184 V70"/><path d="M190,70 V75 H252 V70"/><path d="M258,70 V75 H388 V70"/>
        <path d="M54,150 V155 H184 V150"/><path d="M190,150 V155 H252 V150"/><path d="M258,150 V155 H388 V150"/>
      </g>
      <g font-size="12.5" text-anchor="middle" fill="#b4b3ac">
        <text x="119" y="92">초반</text><text x="221" y="92">중반</text><text x="323" y="92">후반</text>
        <text x="119" y="172">초반</text><text x="221" y="172">중반</text><text x="323" y="172">후반</text>
      </g>
    </svg>
  </div>
  <figcaption id="fig2-cap"><b>그림 2.</b> “40대 초반~50대 초반”은 <code>{"min": 40, "max": 53}</code>이 됩니다. 40대 초반의 시작부터 50대 초반의 끝까지입니다. 강조된 나이가 범위 안에 들어갑니다.</figcaption>
</figure>

모든 공고가 연령대로 나이를 적지는 않습니다. "청년"이나 "중년" 같은 단어만 있으면 프롬프트에 적어 둔 기본 범위로 대응시킵니다. 모델이 쿼리마다 범위를 지어내지 않습니다. 기본값도 추정이긴 하지만, 명시적이고 고칠 수 있는 추정입니다.

출력은 언제나 숫자 한 쌍이고, 라벨이 아닙니다. 초기 검색 프로토타입은 나이를 "젊은 층"과 "나이 든 층" 두 그룹으로만 나눴는데, 두 그룹의 경계가 어디에도 적혀 있지 않았습니다. 숫자 범위는 출생 연도와 바로 비교할 수 있고, 범위가 틀렸을 때 어떻게 틀렸는지 정확히 보입니다.

### 키: 숫자가 있을 때만

"키 178cm 이상"은 `{"min": 178}`이 됩니다. "키 큰 분"이나 "장신"은 아무 조건도 되지 않습니다. 상대적인 표현을 기준값으로 바꾸려면 제작진이 준 적 없는 숫자를 골라야 하고, 그 숫자는 기준에 조금 못 미치는 사람들을 전부 조용히 걸러 냅니다.

### 특기: 플랫폼 어휘 안에서만

Castar 프로필의 특기는 언어, 운동, 악기 같은 정해진 어휘에서 고릅니다. 모델은 이 목록을 받고, 목록에 있는 라벨로만 답할 수 있습니다. 그 과정에서 동의어도 정규화합니다. "말 탈 줄 아는 분"과 "승마 가능"은 모두 플랫폼 특기 '승마'가 됩니다. "스페인어 구사 가능"은 '스페인어', "영어 능통"은 '영어'가 됩니다.

이 제약이 중요한 이유는 필터가 쓸모 있으려면 프로필이 그 필터를 만족할 수 있어야 하기 때문입니다. 모델이 특기를 자유롭게 적으면 "승마 가능"은 특기에 '승마'가 있는 프로필과 끝내 매칭되지 않고, 겉보기에 멀쩡한 필터가 아무도 돌려주지 않게 됩니다.

<figure class="figure" role="group" aria-labelledby="fig3-cap">
  <div class="figure__frame">
    <svg class="fig-capped" viewBox="0 0 420 196" role="img" aria-labelledby="fig3-title fig3-desc">
      <title id="fig3-title">자유로운 특기 표현을 플랫폼 특기로 대응시키기</title>
      <desc id="fig3-desc">왼쪽은 캐스팅 공고에 적힌 표현들: “말 탈 줄 아는 분”, “승마 가능”, “스페인어 구사 가능”, “영어 능통”. 화살표가 두 말 관련 표현을 플랫폼 특기 '승마'로, “스페인어 구사 가능”을 '스페인어'로, “영어 능통”을 '영어'로 연결합니다.</desc>
      <defs>
        <marker id="f3a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto"><path d="M0,0 L10,5 L0,10 z" fill="#8e8e88"/></marker>
      </defs>
      <g font-size="11.5" font-weight="600" letter-spacing="0.5" fill="#8e8e88">
        <text x="0" y="14">캐스팅 공고의 표현</text>
        <text x="420" y="14" text-anchor="end">플랫폼 특기</text>
      </g>
      <g font-size="13.5" fill="#b4b3ac">
        <text x="0" y="50">“말 탈 줄 아는 분”</text>
        <text x="0" y="90">“승마 가능”</text>
        <text x="0" y="140">“스페인어 구사 가능”</text>
        <text x="0" y="180">“영어 능통”</text>
      </g>
      <g fill="none" stroke="#8e8e88" stroke-width="1.2">
        <path d="M146,46 C216,46 220,68 286,68" marker-end="url(#f3a)"/>
        <path d="M146,86 C216,86 220,68 286,68" marker-end="url(#f3a)"/>
        <path d="M146,136 L286,136" marker-end="url(#f3a)"/>
        <path d="M146,176 L286,176" marker-end="url(#f3a)"/>
      </g>
      <g fill="#2b2b29" stroke="#ffffff" stroke-width="1.1">
        <rect x="292" y="53" width="128" height="30" rx="15"/>
        <rect x="292" y="121" width="128" height="30" rx="15"/>
        <rect x="292" y="161" width="128" height="30" rx="15"/>
      </g>
      <g font-size="13.5" font-weight="600" text-anchor="middle" fill="#f2f1ec">
        <text x="356" y="73">승마</text>
        <text x="356" y="141">스페인어</text>
        <text x="356" y="181">영어</text>
      </g>
    </svg>
  </div>
  <figcaption id="fig3-cap"><b>그림 3.</b> 같은 특기를 가리키는 여러 표현이 플랫폼 어휘의 라벨 하나로 모입니다. 그래서 필터는 프로필에 실제로 들어 있을 수 있는 것만 요구합니다.</figcaption>
</figure>

### 필수인가, 우대인가

캐스팅 공고는 배역에 꼭 필요한 조건과 있으면 좋은 조건을 구분하고, 그 구분은 대개 표현에 드러납니다. "우대", "가능자 우대", "있으면 좋음", "~시 가산점" 같은 표현은 그 조건이 선택 사항이라는 뜻입니다. "수영 가능자 우대"는 수영을 `preferred`에 넣습니다. "영어 능통자 필수"처럼 요구 사항으로 적힌 특기는 `required`에 넣습니다.

### 외모 텍스트

외모와 인상을 묘사하는 부분은 짧은 `appearance` 문자열로 모아 이미지–텍스트 채널로 넘깁니다. 이 안의 어떤 내용도 필터가 되지 않습니다.

## 엄격하다는 건 정말 엄격하다는 뜻

조건을 추출하고 나면 필수 조건들을 필터로 적용합니다. 성별, 나이 범위(출생 연도로 계산한 나이와 비교), 키 조건(하한·상한), 필수 특기입니다. 하나라도 만족하지 못하는 배우는 제외됩니다.

**아무도 통과하지 못하면 검색은 거기서 멈추고, 아직 조건에 맞는 배우가 없다고 제작진에게 알립니다.** 나이 범위를 넓히거나 영어 조건을 빼고 다시 검색하지 않습니다.

처음에는 반대로 했습니다. 초기 프로토타입은 필터를 거친 결과가 비면 가장 약한 조건을 빼고, 무언가 나올 때까지 다시 검색했습니다. 친절해 보이지만, 아무에게도 알리지 않고 질문을 바꾸는 셈입니다. "50대"라고 쓴 제작진이 목록 맨 위에서 21세 배우를 보면, 애초에 맞을 수 없던 후보에게 시간을 쓰거나 나머지 목록까지 믿지 않게 됩니다. 결과가 조건에 맞는 것처럼 보인다는 점이 문제를 더 키웁니다. 화면 어디에도 "나이 조건은 무시했습니다"라는 말은 없으니까요.

<figure class="figure" role="group" aria-labelledby="fig4-cap">
  <div class="figure__frame">
    <svg class="fig-capped" viewBox="0 0 440 236" role="img" aria-labelledby="fig4-title fig4-desc">
      <title id="fig4-title">조건 완화 후 재검색과 엄격한 필터 비교</title>
      <desc id="fig4-desc">영어가 가능한 50대 남성을 찾는 가상의 공고에 대한 두 패널. 왼쪽, 완화 후 재검색: 나이 조건이 조용히 빠지고 21세, 26세, 34세 배우가 목록에 나옵니다. 오른쪽, 엄격한 필터: 세 조건이 모두 유지되고 결과는 “아직 조건에 맞는 배우가 없습니다”이며, 무엇을 완화할지는 제작진이 정합니다.</desc>
      <defs>
        <marker id="f4a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto"><path d="M0,0 L10,5 L0,10 z" fill="#8e8e88"/></marker>
      </defs>
      <g fill="#161616" stroke="rgba(255,255,255,0.12)">
        <rect x="0.5" y="0.5" width="211" height="235" rx="10"/>
        <rect x="228.5" y="0.5" width="211" height="235" rx="10"/>
      </g>
      <g font-size="11.5" font-weight="600" letter-spacing="0.5" fill="#8e8e88">
        <text x="16" y="26">완화 후 재검색</text>
        <text x="244" y="26">엄격한 필터</text>
      </g>
      <!-- condition chips -->
      <g font-size="12.5" text-anchor="middle">
        <rect x="16" y="40" width="50" height="26" rx="13" fill="#2b2b29" stroke="#ffffff" stroke-width="1"/>
        <text x="41" y="57" fill="#f2f1ec">남성</text>
        <rect x="72" y="40" width="46" height="26" rx="13" fill="none" stroke="#8e8e88" stroke-width="1" stroke-dasharray="3 3"/>
        <text x="95" y="57" fill="#8e8e88">50대</text>
        <line x1="80" y1="53" x2="110" y2="53" stroke="#b4b3ac" stroke-width="1.2"/>
        <rect x="124" y="40" width="68" height="26" rx="13" fill="#2b2b29" stroke="#ffffff" stroke-width="1"/>
        <text x="158" y="57" fill="#f2f1ec">영어</text>

        <rect x="244" y="40" width="50" height="26" rx="13" fill="#2b2b29" stroke="#ffffff" stroke-width="1"/>
        <text x="269" y="57" fill="#f2f1ec">남성</text>
        <rect x="300" y="40" width="46" height="26" rx="13" fill="#2b2b29" stroke="#ffffff" stroke-width="1"/>
        <text x="323" y="57" fill="#f2f1ec">50대</text>
        <rect x="352" y="40" width="68" height="26" rx="13" fill="#2b2b29" stroke="#ffffff" stroke-width="1"/>
        <text x="386" y="57" fill="#f2f1ec">영어</text>
      </g>
      <text x="95" y="82" font-size="11" text-anchor="middle" fill="#8e8e88">조용히 빠짐</text>
      <g stroke="#8e8e88" stroke-width="1.2">
        <line x1="106" y1="90" x2="106" y2="110" marker-end="url(#f4a)"/>
        <line x1="334" y1="90" x2="334" y2="110" marker-end="url(#f4a)"/>
      </g>
      <!-- relaxed results -->
      <g fill="#101010" stroke="rgba(255,255,255,0.12)">
        <rect x="16" y="118" width="180" height="30" rx="6"/>
        <rect x="16" y="154" width="180" height="30" rx="6"/>
        <rect x="16" y="190" width="180" height="30" rx="6"/>
      </g>
      <g fill="#3d3d3a">
        <circle cx="34" cy="133" r="8"/><circle cx="34" cy="169" r="8"/><circle cx="34" cy="205" r="8"/>
      </g>
      <g font-size="12.5" fill="#f2f1ec">
        <text x="52" y="137">배우 · <tspan font-weight="600">21</tspan>세</text>
        <text x="52" y="173">배우 · <tspan font-weight="600">26</tspan>세</text>
        <text x="52" y="209">배우 · <tspan font-weight="600">34</tspan>세</text>
      </g>
      <g font-size="11" font-family="'IBM Plex Mono', ui-monospace, monospace" fill="#8e8e88" text-anchor="end">
        <text x="186" y="137">#1</text><text x="186" y="173">#2</text><text x="186" y="209">#3</text>
      </g>
      <!-- strict result -->
      <rect x="244" y="118" width="176" height="102" rx="8" fill="none" stroke="#b4b3ac" stroke-width="1" stroke-dasharray="4 3"/>
      <g font-size="13" font-weight="600" text-anchor="middle" fill="#f2f1ec">
        <text x="332" y="152">아직 조건에 맞는</text>
        <text x="332" y="170">배우가 없습니다</text>
      </g>
      <g font-size="11.5" text-anchor="middle" fill="#b4b3ac">
        <text x="332" y="192">무엇을 완화할지는</text>
        <text x="332" y="208">제작진이 정합니다.</text>
      </g>
    </svg>
  </div>
  <figcaption id="fig4-cap"><b>그림 4.</b> 같은 가상의 공고 “50대 남성, 영어 필수”를, 그런 배우가 한 명도 없는 가상의 배우 풀에 대고 검색한 결과입니다. 조건을 완화하면 조용히 다른 질문에 답하게 되고, 엄격한 필터는 실제로 받은 질문에 답합니다.</figcaption>
</figure>

빈 결과에도 정보가 있습니다. 제작진은 플랫폼에 아직 그런 배우가 없다는 사실을 알게 되고, 배역에 정말 중요한 조건이 무엇인지는 제작진이 가장 잘 아니 어떤 조건을 완화할지 직접 정할 수 있습니다. 저희에게는 어떤 조건의 배우가 부족한지 알려 주는 신호이기도 합니다.

우대 조건은 누구도 제외하지 않습니다. 엄격한 필터를 통과한 배우들의 순서에만 영향을 줍니다. 우대 조건을 만족하는 배우는 위로 올라가고, 만족하지 않는 배우도 목록에 그대로 남습니다.

결과 옆에 숫자도 표시하지 않습니다. "일치도 92%"도, 인상 점수도 없습니다. 모델이 낸 유사도 값은 그 사람이 배역에 맞을 확률이 아닌데, 화면에 숫자가 있으면 사람들은 그렇게 읽기 쉽고 87과 84의 차이를 실제 차이로 받아들이게 됩니다. 결과는 점수 없이 순서대로만 보여 줍니다.

## 예시로 따라가 보기

그림 1의 가상 캐스팅 공고 전문입니다.

<div class="casting-call">
  <dl>
    <dt>배역</dt><dd>남자 조연, 40대 초반~50대 초반.</dd>
    <dt>필수</dt><dd>영어 능통자 필수.</dd>
    <dt>키</dt><dd>178cm 이상.</dd>
    <dt>우대</dt><dd>수영 가능자 우대.</dd>
    <dt>외모</dt><dd>따뜻하지만 세월이 느껴지는 얼굴, 짧은 머리.</dd>
  </dl>
</div>

LLM 호출 한 번이 다음을 돌려줍니다(필드 이름과 라벨은 단순화했습니다).

```json
{
  "filters": {
    "gender": "male",
    "age": { "min": 40, "max": 53 },
    "height_cm": { "min": 178, "max": null }
  },
  "skills": {
    "required": ["영어"],
    "preferred": ["수영"]
  },
  "appearance": "따뜻하고 세월이 느껴지는 얼굴의 남성, 짧은 머리"
}
```

한 줄씩 보면 이렇습니다.

- "남자"는 `"gender": "male"`이 됩니다. "조연"은 배우가 아니라 배역의 비중을 말하는 것이므로 조건이 되지 않습니다.
- "40대 초반~50대 초반"은 그림 2의 규칙에 따라 40–53이 됩니다.
- "178cm 이상"에는 숫자가 있으므로 키의 하한이 됩니다. 상한은 없습니다.
- "영어 능통자 필수"는 플랫폼 특기 '영어'로 정규화되고, 공고에 "필수"라고 적혀 있으니 필수 특기가 됩니다.
- "수영 가능자 우대"는 우대로 표시되어 있으니 '수영'은 우대 조건입니다.
- "따뜻하지만 세월이 느껴지는 얼굴, 짧은 머리"는 외모 텍스트가 됩니다. 이 중 어떤 것도 필터가 되지 않습니다.

그러면 검색은 40세부터 53세 사이, 키 178cm 이상, 특기에 영어가 있는 남성 배우만 남깁니다. 이 집합이 비어 있으면 제작진에게는 "아직 조건에 맞는 배우가 없습니다"가 보입니다. 비어 있지 않으면 수영이 가능한 배우가 위로 올라가고, 나머지 순서는 외모 채널이 정합니다.

## 아직 풀지 못한 것들

- **모호한 나이 표현은 기본값에 기댑니다.** "청년"은 프롬프트가 정해 둔 범위가 됩니다. 기본값은 명시적이고 바꾸기도 쉽지만, 제작진이 의도한 바에 대한 저희의 추정일 뿐이고 모든 제작진이 동의하지는 않을 겁니다.
- **어휘는 계속 관리해야 합니다.** 새로운 특기, 그리고 기존 특기를 부르는 새로운 표현("승마 가능", "말 탈 줄 앎", "승마 경험 있음")이 생길 때마다 어휘 목록에 자리를 만들어 줘야 합니다. 어휘에 없는 특기는 아예 필터가 될 수 없습니다.
- **출생 연도는 거친 정보입니다.** 출생 연도로 계산한 나이는 생일 전후로 한 살 차이가 날 수 있어서, 범위 경계에 있는 배우는 범위 안에 들 수도, 밖으로 밀려날 수도 있습니다.
- **엄격함은 양날의 검입니다.** 필수 조건은 사람을 배제하므로, 추출 실수(나이를 잘못 읽거나 우대 특기를 필수로 표시하는 것)도 아무 말 없이 사람을 배제합니다. 엄격한 필터를 쓰면 이 LLM 호출 한 번의 품질이 덜 중요해지는 게 아니라 더 중요해집니다.
- **평가는 진행 중입니다.** 실제 공고의 다양한 표현을 추출이 얼마나 안정적으로 처리하는지 아직 측정하고 있습니다. 믿을 만한 결과가 나오면 따로 쓰겠습니다.

## 다음 글

이 글에서는 캐스팅 공고의 절반인 기호적 조건, 즉 필드에 대응하고 정확하게 확인할 수 있는 조건을 다뤘습니다. 나머지 절반인 "따뜻하지만 세월이 느껴지는 얼굴", "선이 날카로운 이목구비" 같은 외모 표현은 그렇게 확인할 수 없습니다. 이런 표현은 별도의 이미지–텍스트 채널에서 처리하며, 그 내용은 다음 글에서 다룹니다.
