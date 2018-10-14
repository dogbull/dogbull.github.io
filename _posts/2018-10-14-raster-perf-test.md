---
title: "GeoTIFF, VRT, WarpedVRT 읽기 속도 비교"
date: 2018-10-14 01:51:28 +0900
categories: computer
tag: [python, gis, gdal, rasterio, raster, vrt, WarpedVRT] 
---

본 포스트는 [GeoTIFF][gdal-geotiff], [GDAL Virtual Format][gdal-vrt], [WarpedVRT][warpedvrt] 등
3가지의 래스터 파일 포멧에 대한 읽기 속도 측정 결과를 제시합니다.

GDAL 라이브러리/유틸리티를 이용하여 GIS 래스터를 다룰 때 
[다양한 자료 형식][gdal-formats] 중 필자는 주로 [GeoTIFF][gdal-geotiff] 를 사용합니다. 
다양한 압축 형식, 타일링, 내부 오버뷰(오버뷰를 tif 파일 내에 저장하여 하나의 파일로 관리할 수 있는 편리함.
외부 오버뷰도 지원.), 태그, 파일 최대 크기 제한 없음(BigTIFF 를 이용하여 가능), 멀티 스레드 읽기/쓰기 등
다양한 편의 기능들이 있기 때문입니다.

[GDAL Virtual Format][gdal-vrt](이하 VRT)는 래스터의 실제 격자 값을 담고 있지 않습니다.
킬로바이트 정도의 XML 형식으로 작성됩니다.([예시][vrt-sample]) 이 XML 안에 투영, 지오트랜스,
실제격자값을갖는파일의위치(GeoTIFF 등과 같은 GDAL 지원 형식의 파일) 등과 같은 기본 적인 정보와 타일 크기,
NO DATA, 보간 방법, 데이터 유형(Float32, Int16, ...) 등과 같은 정보가 기술되어 있습니다. VRT 는 GDAL 유틸리티를
이용하여 매우 쉽게 생성할 수 있습니다.(`gdalwarp -of vrt src.tiff dst.vrt`)

[WarpedVRT][warpedvrt]는 Python의 [rasterio][rasterio] 패키지에 포함된 
고수준의 래스터 처리 라이브러리입니다. Python GDAL 을 직접 사용하는 것 보다 rasterio 를 사용하면 보다 적은
코드로도 동일한 작업을 수행할 수 있고, 몇 가지 편리한 함수들을 사용할 수 있습니다.

그렇다면 `VRT`와 `WarpedVRT`와 같은 기능(?)을 사용하는 이유는 무엇이길래 이 포스트까지 작성하는 걸까요?
필자의 한정적인 경우에는 투영변환과 영역설정의 경우에 주로 사용합니다. 
`gdalwarp -of vrt src.tiff dst.vrt` 와 같은 명령을 수행하면 `dst.vrt` 라는 XML 형식의 텍스트 파일이 만들어지게 되는데,
실제 격자 값은 VRT에 담겨 있지 않고 `src.tiff` 라는 파일의 포인터(파일 경로) 만을 가지고 있게 되어 
아주 빠른 속도(1초 이내)로 변환을 수행할 수 있게 되고(단 이때 얻은 이득은 실제 데이터를 읽어들이는 타임에 발생.)
변환된(사실 변환 되었다기 보다는 변환 될 수 있는 정보를 가지고 있는 상태가 더 정확.) 파일의 크기가 매우 작고, 
사람이 읽을 수 있는 텍스트 형식이므로 수정이 용이하다는 장점이 있습니다. `WarpedVRT`의 경우는 VRT와 같이 XML 파일을 
직접 생성하지 않고 프로그램 상에서 VRT의 XML 내용에 상응하는 정보를 기술하여
투영변환과 영역설정을 할 수 있는 장점이 있습니다.

다음과 같은 시나리오를 가정해 봅시다.
1. EPSG:5179를 투영으로 하는 전국 1km 격자 해상도의 GeoTIFF 파일 `A`가 있음.
2. EPSG:5186를 투영으로 하는 전국 2km 격자 해상도의 GeoTIFF 파일 `B`가 있음.
3. `A`와 `B`을 곱하여 `C`를 계산.
4. EPSG:4326를 투영으로 하는 경기도(경기도를 포함하는 최소한의 사각형 영역) 5km 격자 해상도의 GeoTIFF로 `C`를 저장

이 작업을 수행하는 최적의 코드를 작성하는 것이 목표이며, 본 포스트는 이를 위한 몇 가지 확인 작업 중 한 가지입니다.

아래는 GeoTIFF, VRT, WarpedVRT 를 이용한 격자 자료 읽기에 대한 코드입니다.
```python
from time import time

import rasterio
from rasterio.vrt import WarpedVRT

tiff_path = '/mnt/c/Users/pjh/Desktop/dem.30m.1.tiff'
vrt_path = '/mnt/c/Users/pjh/Desktop/dem.30m.1.vrt'

s = time()
r = rasterio.open(tiff_path)
d = r.read(1)
print(f'GeoTIFF 읽기 소요 시간: {time() - s}초')

s = time()
r = rasterio.open(vrt_path)
d = r.read(1)
print(f'VRT 읽기 소요 시간: {time() - s}초')

s = time()
r = WarpedVRT(rasterio.open(tiff_path))
d = r.read(1)
print(f'WarpedVRT 읽기 수행 소요 시간: {time() - s}초')
```

VRT 파일은 아래와 같은 명령으로 생성했습니다.
```bash
gdalwarp -of VRT '/mnt/c/Users/pjh/Desktop/dem.30m.1.tiff' '/mnt/c/Users/pjh/Desktop/dem.30m.1.vrt'
```
이 명령은 원본자료 dem.30m.1.tiff 자료를 이용하여 dem.30m.1.vrt 파일을 생성하는데, 어떤 변환 작업도 수행하지 않습니다.
즉 tiff 파일의 격자 값과 vrt 파일의 격자 값은 동일합니다.(격자의 값 뿐만 아니라 가로/세로 크기, 래스터의 투영,
영역 등 모두 동일.) 이렇게 아무런 변환도 수행하지 않는 쓸모없는 VRT 를 생성하는 이유는 tiff와 vrt의 읽기 속도를 동일
조건에서 측정하기 위한것입니다.

아래와 같이 작성된 WarpedVRT 역시 앞서 설명된 VRT와 같이 아무런 변환도 수행하지 않는 더미입니다.
```python
r = WarpedVRT(rasterio.open(tiff_path))
```

아래는 가장 상세한 해상도인 30m(래스터 해상도는 21396x20324)에서 부터 310m(래스터 해상도는 2139x2033)까지 10m씩 증가시켜
읽기속도를 측정한 결과입니다.

![해상도 변경에 따른 속도 변화]({{ "/assets/img/posts/2018/10/14/plot1.png" | absolute_url }})

VRT와 WarpedVRT의 속도는 동일하다고 해도 무방해 보입니다. GeoTIFF를 직접 읽어들이는 것이 항상 VRT/WarpedVRT 대비 
항상 빠르고 최대치는 4배 정도로 측정되었습니다. 
변환된 래스터를 자주 로드해야 한다면 VRT/WarpedVRT와 같은 가상 데이터를 사용하는 대신 
실제로 변환된 데이터를 파일로 저장해 놓고 재사용하는 것이 훨씬 나은 선택일 것으로 보입니다.
하지만 변환된 데이터를 보관하고 있을 필요가 없는 일시적인 사용이라면
가상 데이터를 사용하는 것도 편의성 면에서 좋은 선택이 될 수 있을 것입니다. 



[gdal-vrt]: https://www.gdal.org/gdal_vrttut.html
[warpedvrt]: https://rasterio.readthedocs.io/en/latest/topics/virtual-warping.html
[gdal-formats]: https://www.gdal.org/formats_list.html
[gdal-geotiff]: https://www.gdal.org/frmt_gtiff.html 
[vrt-sample]: https://gist.github.com/dogbull/422d26cae60be65682f47bb018358be4
[rasterio]: https://github.com/mapbox/rasterio