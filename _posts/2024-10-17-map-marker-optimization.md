---
title: React Native 지도 및 마커 최적화하기
description: 클러스터링을 이용하여 마커 렌더링 시간을 개선해보자
date: 2024-10-17 22:45:00 +/-TTTT
categories: [Frontend, Map]
tags: [react native, naver maps, google map, markers, clustering, optimization]
image:
  path: assets/img/writing/7/extent_clustering_markers.png
sitemap: 
    changefreq : daily
    priority : 0.8
---

## 1. 개요

[react-native-naver-map](https://github.com/mym0404/react-native-naver-map) v1.5.9를 활용하여 네이버 지도를 사용하고 있었는데 몇 가지 문제점이 발생했다.

1. Zoom 레벨이 낮을수록(지도의 넓은 범위를 렌더링할수록) 수많은 마커를 렌더링하게 되는데, 이로 인해 성능 저하가 발생한다.
2. 터치가 종료되는 시점에서 위치를 받아오지 못해, 카메라의 위치가 변경될 때마다 불필요한 `onCameraChanged` 이벤트가 발생한다.
  - 이러한 잦은 이벤트 발생으로 인해 지도에서 **1초 이상의 지연**이 발생하며, 현재 위치에서 주변 마커를 조회하는 기능이 느려진다.

오늘이 이 문제들을 해결해 나가는 과정들을 기록으로 남기고자 한다! 우선 첫 번째 문제부터 바로 해결해 보자.

## 2. 마커(Marker) 최적화하기

![few_markers](assets/img/writing/7/few_markers.png)

현재 이미지처럼 지도에서 적은 수의 마커를 렌더링하는 경우, 73.3ms밖에 걸리지 않는다. 이 정도의 시간은 사용자가 느끼기에 큰 문제가 되지 않는다.

> 하지만 문제는 Zoom 레벨이 낮아질 때, 즉 지도의 범위가 넓어질 때 발생한다. 만약 마커 수가 1,000개가 된다면 어떻게 될까?
{: .prompt-warning }

이를 테스트하기 위해 모킹 데이터를 사용해 1,000개의 마커를 렌더링해 보았다.

![over_markers](assets/img/writing/7/over_markers.png)

그 결과 **1,000개의 마커를 렌더링하기 위해 총 4.4s**가 걸렸다. 렌더링이 되기까지 걸리는 시간도 문제점이지만, 현재 지도상에 너무 많은 마커가 표시되고 있어 성능을 저하할 뿐만 아니라 시인성을 떨어뜨리고 있다. 오케이 문제점에 대해 인지했으니 이제 해결해 보자.

### 2-1. 겹치는 마커 숨기기

우선 첫 번째로 시도해 본 방법은 마커가 중복되는 위치에 접해있는 경우, 즉 교집합이 생기는 마커에 대해서는 후순위로 렌더링 되는 마커를 렌더링하지 않도록 하는 것이었다. react-native-naver-map의 공식 문서를 살펴보면 [isHideCollidedMarkers props](https://mym0404.github.io/react-native-naver-map/interfaces/NaverMapMarkerOverlayProps.html#isHideCollidedMarkers)를 확인할 수 있다. 사용법은 간단하다, 렌더링하고 있는 `NaverMapMarkerOverlay` 컴포넌트의 `props`로 `isHideCollidedMarkers` 옵션을 추가해 주기만 하면 된다.

```tsx
import MapView, { Marker, PROVIDER_GOOGLE } from 'react-native-maps';

export function MapHomeScreen() {
  // ...

  return (
    <View style={styles.container}>
      <NaverMapView
        ref={mapRef}
        provider={PROVIDER_GOOGLE}
        style={styles.mapView}
        {...getMapDefaultOption(coords)}
      >
        {markers.map((marker) => (
            <NaverMapMarkerOverlay
              key={marker.id}
              latitude={marker.latitude}
              longitude={marker.longitude}
              isHideCollidedMarkers // ✅ 다른 마커와 겹치지 않는 마커만이 노출
            />
        ))}
      </NaverMapView>
    </View>
  );
}
```

![isHideCollided_markers](assets/img/writing/7/isHideCollided_markers.png)

위와 같이 적용한 결과 지도에서 노출되는 마커의 수가 줄어든 것을 확인할 수 있었다. 하지만 노출되는 마커의 수와는 달리 렌더링 속도는 전혀 개선되지 않은 모습을 보여주고 있는데, 이는 `isHideCollidedMarkers`이 겹치는 마커를 렌더링하지 않는 것이 아닌 `opacity`를 0으로 하여 **숨김 처리**를 하기 때문이다. 실제로 줌 레벨을 확대하면 현재 보이지 않는 마커들의 `opacity`가 1로 변하면서 보이게 되는 것을 확인할 수 있다.

첫 번째로 시도한 방법은 현재 2가지의 문제가 있다.

1. 성능 개선이 전혀 이루어지지 않은 점 (4.4s -> 4.2s)
  - 실제로 Profiler를 확인해 보면 Render 속도는 줄어들지 않고 Layout effects의 시간만 줄어든 것을 확인할 수 있다.
2. 겹치지 않는 마커만 노출했음에도 여전히 많은 마커가 보이고 있으며 아직도 시인성이 좋지 못하다.

첫 번째로 시도한 방법에서는 내가 첫대목에 풀고자 했던 **성능 저하** 문제를 해결하지 못했다. 그렇다면 다른 방법인 클러스터링을 적용해 보자.

### 2-2. 클러스터링

> **클러스터링(Clustering)**
>
> 클러스터링이란 '무리를 이룬다'는 뜻으로, 서로 유사한 속성을 갖는 데이터를 같은 군집으로 묶어주는 작업을 의미합니다.
{: .prompt-tip }

클러스터링의 개념은 머신러닝에서도 사용되는 개념이지만 지도에서 마커들을 그룹화하는 데도 사용되는 개념이다. 그럼, 현재 위 지도에 클러스터링을 적용하여 마커들을 그룹화해 보자. react-native-naver-map 라이브러리에서 제공해 주는 [clusters props](https://mym0404.github.io/react-native-naver-map/interfaces/NaverMapViewProps.html#clusters)를 적용하여 마커들을 그룹화해 보자.

```tsx
import MapView, { Marker, PROVIDER_GOOGLE } from 'react-native-maps';

export function MapHomeScreen() {
  // ...

  return (
    <View style={styles.container}>
      <NaverMapView
        ref={mapRef}
        provider={PROVIDER_GOOGLE}
        style={styles.mapView}
        clusters={[ // ✅ Clustering props 적용
          {
            animate: true, // 확대/축소 시 클러스터링 애니메이션 적용
            minZoom: 4, // 클러스터링할 최소 줌 레벨
            maxZoom: 16, // 클러스터링할 최대 줌 레벨
            screenDistance: 30, // 클러스터링할 마커들의 거리 기준 (화면 상 거리)
            markers: markers.map((marker) => ({
              identifier: String(marker.id), // 문자열로 된 id 값
              latitude: marker.latitude,
              longitude: marker.longitude,
            })),
          },
        ]}
        {...getMapDefaultOption(coords)}
      />
    </View>
  );
}
```

![clusters_markers](assets/img/writing/7/clusters_markers.png)

위 방법으로 적용한 결과 모든 마커를 렌더링하는 것이 아닌 마커들을 그룹화하고, 그룹화한 마커를 렌더링하는 방식으로 렌더링 작업이 완료되기까지의 시간이 **4.2s -> 1.9s**로 크게 개선된 것을 확인할 수 있었다. 우선 내가 생각했던 문제인 1번을 해결할 수 있게 되었다.

- [x] Zoom 레벨이 낮을수록(지도의 넓은 범위를 렌더링할수록) 수많은 마커를 렌더링하게 되는데, 이로 인해 성능 저하가 발생한다.

하지만 렌더링 속도에 대한 문제를 해결하고 보니 다른 문제가 보이기 시작했다.

1. 클러스터링 영역의 반경을 설정할 수 없다.
2. 클러스터링 된 마커들에 대해 커스텀 스타일을 적용할 수 없다.

2번 문제는 최신 버전(v2.2.0)으로 업데이트하면 `width`, `height` 등의 추가적인 props를 사용하여 어느 정도 크기는 조정이 가능하지만, 현재 1.5.9 버전을 사용하고 있는 나에게 major 버전의 업데이트는 부담이었다. 왜냐하면 커스텀 스타일이라고 단순한 스타일링만 제공할 뿐 커스텀 마커, 커스텀 클러스터링이 불가능했기 때문이다. 또한 v2.2.0의 최신 버전을 이용하기 위해서는 RN v0.74의 요구사항을 만족해야 하는데, RN 버전을 업데이트하게 되면 기존에 설치된 라이브러리들과의 호환성 문제가 발생하게 된다.

마커를 최적화하는 데 불편함이 없었더라면 터치가 종료되는 시점에서 위치를 받아올 수 없었던 문제를 해결하기 위해 노력하였겠지만, 마커를 최적화하고 또 우리 서비스에 맞게 커스텀하는데 어려움이 있어 현재 사용하고 있는 react-native-naver-maps를 제거하고 [react-native-maps](https://github.com/react-native-maps/react-native-maps)로 지도를 변경하게 되었다.

## 3. Google Maps

구글 지도를 사용하기 위해 [react-native-maps](https://github.com/react-native-maps/react-native-maps) 라이브러리를 사용하였는데 해당 라이브러리는 원활하게 운영되고 있는 오픈 소스로 15.1k의 스타를 가지고 있다.

네이버 지도는 국내에서만 사용되는 반면 구글 지도는 아무래도 global 하게 사용되는 지도인 만큼 어느 정도 규모가 있는 것을 확인할 수 있다. 본론으로 돌아가 구글 지도에 앞서 사용한 마커 1,000개를 렌더링하고 성능을 측정해 보자. 

{% raw %}
```tsx
import MapView, { Marker, PROVIDER_GOOGLE } from 'react-native-maps';

export function MapHomeScreen() {
  // ...

  return (
    <View style={styles.container}>
      <MapView
        ref={mapRef}
        provider={PROVIDER_GOOGLE}
        style={styles.mapView}
      >
        {markers.map((marker) => (
          <Marker
            key={marker.id}
            coordinate={{
              latitude: marker.latitude,
              longitude: marker.longitude,
            }}
          />
        ))}
      </MapView>
    </View>
  );
}
```
{% endraw %}

![google_markers](assets/img/writing/7/google_markers.png)

Profiler로 성능을 측정해 본 결과 놀랍게도 구글 지도에서 마커 1,000개를 렌더링하는 데 걸리는 시간은 네이버 지도와 비교하여 무려 3초(4.4s -> 1.4s)나 빨랐다. 심지어 네이버 지도에서 클러스터링을 적용한 것과 거의 동등한 렌더링 속도를 보여주고 있었다. 그렇다면 구글 지도에서 클러스터링을 적용하면 속도가 얼마나 빨라질까?

### 3-1. react-native-map-clustering

[react-native-maps](https://github.com/react-native-maps/react-native-maps)를 이용하여 지도와 마커를 구현하였다면 클러스터링을 적용하는 것은 react-native-maps를 설정한 것만큼이나 쉽다. [react-native-map-clustering](https://github.com/venits/react-native-map-clustering) 라이브러리를 이용하여 클러스터링을 적용해 보기 전 기능들에 대해 알아보자.

react-native-map-clustering 라이브러리는 정말 쉬운 설정으로 강력한 기능들을 제공해 준다.

1. 클러스터링 기능
2. 클러스터링 마커 커스텀 뷰 설정
3. react-native-maps MapView 기반

그럼 클러스터링을 적용한 후에 테스트를 해보자.

{% raw %}
```tsx
import { Marker, PROVIDER_GOOGLE } from 'react-native-maps';
import MapView from 'react-native-map-clustering'; // ✅ 클러스터링 맵뷰 적용

export function MapHomeScreen() {
  // ...

  return (
    <View style={styles.container}>
      <MapView
        ref={mapRef}
        provider={PROVIDER_GOOGLE}
        initialRegion={{ // ✅ 클러스터링 맵뷰 사용을 위해 initialRegion을 설정
          latitude: 37.5665,
          longitude: 126.978,
          latitudeDelta: 0.0922,
          longitudeDelta: 0.0421,
        }}
        style={styles.mapView}
      >
        {markers.map((marker) => (
          <Marker
            key={marker.id}
            coordinate={{
              latitude: marker.latitude,
              longitude: marker.longitude,
            }}
          />
        ))}
      </MapView>
    </View>
  );
}
```
{% endraw %}

![google_clusters_markers](assets/img/writing/7/google_clusters_markers.png)

클러스터링을 적용한 결과 렌더링이 완료되기까지 걸리는 시간은 1.7s가 걸리는 것을 확인할 수 있다. 하지만 여기서 중요한 것은 Render 시간이 이전과 비교하여 크게 **개선(273.6ms -> 98.8ms)**이 이루어진 것을 확인할 수 있다. 하지만 지도를 확인해 보면 클러스터링 마커가 깔끔하게 적용되지 않고 있는 것을 확인할 수 있는데 이는 현재 MapView의 타일 범위가 넓게 잡혀있기 때문이다.

> extent
> 타일 범위를 나타내는 속성으로 클러스터링의 반경(radius)이 이 값을 기준으로 계산됩니다. react-native-map-clustering에서 `extent`의 기본값은 256입니다.
{: .prompt-tip }

그렇다면 extent 값을 조정하여 타일 범위를 좁게 하여 더 넓은 반경의 마커들을 포함할 수 있도록 처리해 보자.

{% raw %}
```tsx
import { Marker, PROVIDER_GOOGLE } from 'react-native-maps';
import MapView from 'react-native-map-clustering';

export function MapHomeScreen() {
  // ...

  return (
    <View style={styles.container}>
      <MapView
        ref={mapRef}
        provider={PROVIDER_GOOGLE}
        initialRegion={{
          latitude: 37.5665,
          longitude: 126.978,
          latitudeDelta: 0.0922,
          longitudeDelta: 0.0421,
        }}
        extent={128} // ✅ 타일의 범위를 128로 설정
        style={styles.mapView}
      >
        {markers.map((marker) => (
          <Marker
            key={marker.id}
            coordinate={{
              latitude: marker.latitude,
              longitude: marker.longitude,
            }}
          />
        ))}
      </MapView>
    </View>
  );
}
```
{% endraw %}

![extent_clustering_markers](assets/img/writing/7/extent_clustering_markers.png)

위와 같이 타일의 범위를 설정해 준 후 성능을 측정해 보면, Render 시간이 무려 26.6ms로 크게 줄어든 것을 확인할 수 있다. 이렇듯 `extent`는 타일의 범위를 설정하는 값으로 낮으면 낮을수록 반경이 크게 적용된다. `extent`의 값을 줄일수록 클러스터링 된 마커의 수가 줄어들기 때문에 성능이 개선되겠지만 지도에 텅 빈 마커를 사용자에게 보여줄 수 있으니 이는 잘 고려하여 값을 설정해야 한다.

### 3-2. Deep Dive

네이버 지도와 구글 지도의 오픈 소스를 파헤쳐 마커 렌더링 속도 차이가 발생하는 원인을 알아보자. 비교한 버전은 네이버 지도 v1.5.9, 구글 지도 v1.13.2이고 간단하게 네이버 지도에서 사용되는 마커는 네이버 마커, 구글 지도에서 사용되는 마커는 구글 마커라고 하겠다.

#### 3-2-1. 컴포넌트 내부 연산 작업

[네이버 마커](https://github.com/mym0404/react-native-naver-map/blob/v1.5.9/src/component/NaverMapMarkerOverlay.tsx#L319-L429)를 살펴보면 마커를 렌더링하기 이전에 총 4번의 `useMemo` 연산이 사용되며, [구글 마커](https://github.com/react-native-maps/react-native-maps/blob/v1.13.2/src/MapMarker.tsx#L329-L430)는 값을 캐싱하기 위한 `useMemo` 연산이 없는 것을 확인할 수 있다.

medium에 작성된 [Should You Really Use useMemo in React? Let’s Find Out.](https://medium.com/swlh/should-you-use-usememo-in-react-a-benchmarked-analysis-159faf6609b7) 게시물을 확인해 보면 `useMemo`를 이용하였을 경우와 이용하지 않았을 경우의 렌더링 속도를 비교한 벤치마킹 표를 확인할 수 있다. 초기 렌더링에서는 `useMemo`를 사용하지 않는 경우가 빠르지만 대체로 데이터의 수가 많아짐에 따라 리렌더링이 발생할 때 `useMemo`의 렌더링 속도가 빠른 것을 확인할 수 있다.

이러한 `useMemo` 연산으로 인해 네이버 마커가 초기 렌더링 시 발생하는 속도가 구글 마커보다 느린 것으로 생각된다.

#### 3-2-2. ref를 이용한 마커 렌더링 방식

```tsx
// https://github.com/react-native-maps/react-native-maps/blob/v1.13.2/src/MapMarker.tsx#L329-L430

export class MapMarker extends React.Component<MapMarkerProps> {
  // ...

  private marker: NativeProps['ref'];

  constructor(props: MapMarkerProps) {
    super(props);

    this.marker = React.createRef<MapMarkerNativeComponentType>();
    this.showCallout = this.showCallout.bind(this);
    this.hideCallout = this.hideCallout.bind(this);
    this.setCoordinates = this.setCoordinates.bind(this);
    this.redrawCallout = this.redrawCallout.bind(this);
    this.animateMarkerToCoordinate = this.animateMarkerToCoordinate.bind(this);
  }

  setNativeProps(props: Partial<NativeProps>) {
    // @ts-ignore
    this.marker.current?.setNativeProps(props);
  }

  showCallout() {
    if (this.marker.current) {
      Commands.showCallout(this.marker.current);
    }
  }

  hideCallout() {
    if (this.marker.current) {
      Commands.hideCallout(this.marker.current);
    }
  }

  setCoordinates(coordinate: LatLng) {
    if (this.marker.current) {
      Commands.setCoordinates(this.marker.current, coordinate);
    }
  }

  redrawCallout() {
    if (this.marker.current) {
      Commands.redrawCallout(this.marker.current);
    }
  }

  animateMarkerToCoordinate(coordinate: LatLng, duration: number = 500) {
    if (this.marker.current) {
      Commands.animateMarkerToCoordinate(
        this.marker.current,
        coordinate,
        duration,
      );
    }
  }

  redraw() {
    if (this.marker.current) {
      Commands.redraw(this.marker.current);
    }
  }

  // ...
}
```

구글 마커에서는 마커를 렌더링하기 위해 혹은 마커의 정보를 업데이트하기 위해 상태를 변경하는 방식이 아닌 ref를 이용한 참조를 통해 해당 문제를 해결하고 있었다. 이러한 방식으로 구글 마커는 불필요한 리 렌더링을 최소화하며 즉시 실행하여 성능상의 이점을 챙긴 것으로 같다.

## 4. 마치며

마커 최적화 작업을 진행하며 클러스터링이라는 기법에 대해 학습할 수 있었으며 이를 이용해 성능을 크게 개선할 수 있었다. 성능상의 이점을 가져온 것도 매우 값지지만 각 지도의 차이를 분석하면서 성능 차이가 발생하는 원인에 대해서도 깊이 이해할 수 있어 정말 값진 경험이었다.

- [x] Zoom 레벨이 낮을수록(지도의 넓은 범위를 렌더링할수록) 수많은 마커를 렌더링하게 되는데, 이로 인해 성능 저하가 발생한다.
  - [react-native-map-clustering](https://github.com/venits/react-native-map-clustering) 적용 및 `extent` 값 조정
  - **Committed at: 4.4s -> 1s**
  - **Render: 3222.1ms -> 26.6ms**
- [x] 터치가 종료되는 시점에서 위치를 받아오지 못해, 카메라의 위치가 변경될 때마다 불필요한 `onCameraChanged` 이벤트가 발생한다.
  - 기존에 react-native-naver-maps를 사용할 때 발생하는 지속적으로 카메라의 위치를 확인해야만 했던 문제를 [react-native-maps](https://github.com/react-native-maps/react-native-maps) MapView 컴포넌트의 `onRegionChangeComplete`를 이용하여 해결

## 참고

- [[노코드머신러닝] 데이터를 유사한 속성으로 묶어준다고요?](https://ablearn.kr/newsletter/?bmode=view&idx=13465571)
- [React Native Naver Map \| MJ Studio](https://mym0404.github.io/react-native-naver-map)
- [Cluster react-native-maps Markers with react-native-map-clustering](https://medium.com/@abegehr/cluster-react-native-maps-markers-with-react-native-map-clustering-72f50db26891)
- [supercluster](https://github.com/mapbox/supercluster)
