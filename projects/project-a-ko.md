## 프로젝트 개요

이 프로젝트는 언리얼 엔진 5 기반으로 **수십만 개의 건물과 도로가 존재하는 대규모 가상 도시**를 실시간으로 렌더링하는 최적화 기법을 연구하고 적용한 사례입니다.

> "프레임 드랍 없이 도시를 달릴 수 있을 때, 플레이어는 비로소 그 세계가 실재한다고 느낀다."

초기 프로토타입에서는 시내 중심부를 달릴 때 **평균 23fps**에 불과했습니다. 최종 빌드에서는 동일 구간에서 **안정적인 60fps**를 달성했습니다.

![최종 렌더링 결과 — 도심 구간 야간 씬](images/project-a/result-01.png)
*UE5 Lumen + 커스텀 대기 산란 쉐이더 적용 후 최종 렌더링 결과물*

---

## 핵심 기술 문제

도시 규모의 씬을 렌더링할 때 발생하는 주요 병목은 크게 세 가지였습니다.

- **Draw Call 폭증**: 건물 하나하나가 독립적인 메시일 때, 카메라 시야에만 수천 개의 Draw Call이 발생
- **메모리 스트리밍 지연**: 고해상도 텍스처를 뒤늦게 로드하여 팝핑(Popping) 현상 빈발
- **Lumen 글로벌 일루미네이션 오버헤드**: 개방형 씬에서 Lumen의 비용이 예상보다 3배 이상 높음

---

## 해결 방법

### 1. HLOD (Hierarchical Level of Detail) 재설계

기본 UE5 HLOD 시스템은 우리 씬 구조와 맞지 않아 직접 파이프라인을 수정했습니다.

```cpp
// 커스텀 HLOD 생성 로직 (간략화)
void UUrbanHLODBuilder::GenerateClusters(
    const TArray<AActor*>& InActors, 
    float ClusterRadius)
{
    for (AActor* Actor : InActors)
    {
        FVector Origin = Actor->GetActorLocation();
        AssignToNearestCluster(Origin, ClusterRadius);
    }
}
```

클러스터 반경을 거리별로 동적 조정한 결과, Draw Call을 **기존 대비 78% 절감**했습니다.

### 2. 커스텀 Post-Process Shader

도심 특유의 대기 산란(Atmospheric Scattering)을 표현하기 위해 HLSL로 커스텀 패스를 작성했습니다.

```hlsl
// 대기 산란 근사 (Mie Scattering 단순화)
float3 ApplyHaze(float3 color, float depth)
{
    float3 hazeColor = float3(0.7, 0.75, 0.85);
    float factor = 1.0 - exp(-depth * 0.0008);
    return lerp(color, hazeColor, saturate(factor));
}
```

### 3. 비동기 텍스처 스트리밍 개선

UE5의 기본 스트리밍 우선순위 알고리즘을 오버라이드하여, **플레이어 이동 벡터**를 예측 인자로 추가했습니다. 이를 통해 팝핑 현상이 육안으로 거의 식별 불가능한 수준으로 감소했습니다.

---

## 성과 요약

| 항목 | Before | After | 개선율 |
|---|---|---|---|
| 평균 FPS (도심) | 23 fps | 61 fps | **+165%** |
| Draw Call (시야 내) | 4,200 | 920 | **-78%** |
| 텍스처 스트리밍 지연 | 1.8s | 0.3s | **-83%** |
| GPU 메모리 사용량 | 9.2 GB | 6.1 GB | **-34%** |

---

## 회고

이 프로젝트를 통해 **엔진 내부 구조를 이해하는 것이 최적화의 핵심**임을 깊이 체감했습니다. 문서화된 방법만으로는 한계가 있었고, 직접 소스 코드를 분석하고 실험하는 과정에서 가장 큰 성과를 얻었습니다.

다음 단계로는 GPU Instancing과 Nanite의 커스텀 LOD 파이프라인 연동을 탐구할 계획입니다.
