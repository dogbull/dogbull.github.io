---
title: "asyncio를 이용한 래스터 읽기 속도 비교"
date: 2018-10-14 22:51:28 +0900
categories: computer
tag: [python, gis, gdal, rasterio, raster, vrt, WarpedVRT, asyncio] 
---

본 포스트는 한 개 이상의 [GeoTIFF][gdal-geotiff] 파일을 
동기 방식과 비동기 방식([asyncio][asyncio]를 이용)으로 읽을 때
소요되는 시간을 비교해 봅니다.

우선 읽어들일 [GeoTIFF][gdal-geotiff] 파일은 256x256으로 타일링된 [deflate][deflate] 알고리즘이 사용되었습니다.
전국 30m 해상도(래스터 격자수는 21396x20324)의 자료를 이용하여 테스트하였으며 각 격자는 float32(4바이트)입니다.
1개에서 부터 5개까지 래스터를 읽어 보면서 소요 시간을 측정해 보았습니다.(약 5개가 테스트 PC에서의 최대치로 예상
됩니다. 300MB 정도로 압축된 GeoTIFF가 압축 해제되어 메모리에 올라갈 경우 1개당 최소 1.7GB 이므로 5개를 읽어들일 시 
약 8.5의 메모리가 요구됩니다.) 

동기 방식(순차 방식과 동의)으로 파일을 읽을 경우 (캐시 효과를 고려하지 않았을 때)
래스터 개수에 대해 선형적으로 소요 시간이 증가할 것으로 예상되는 것에 비해,
비동기 방식으로 읽을 경우 어느 정도의 시간적 이득을 얻을 수 있을 것으로 예상됩니다. 
물론 동기 방식으로 1개의 래스터를 읽어들이는 것이나 비동기 방식으로 1개의 래스터를 읽어들이는 것이나 소요 시간은
동일할 것입니다. 여기서 말하는 비동기 방식이란 여러 개의 래스터를 비동기 방식으로 읽는 것을 의미하기 때문입니다.

아래는 테스트를 수행한 코드입니다.

```python
import asyncio
from time import time

import rasterio

tiffs = [
    f'/mnt/c/Users/pjh/Desktop/dem.30m.{i}.tiff'
    for i in range(0, 5)
]


def tiff_use_sync():
    for t in tiffs:
        r = rasterio.open(t)
        d = r.read(1)


def use_async_tiff():
    async def read(path):
        r = await loop.run_in_executor(None, rasterio.open, path)
        d = await loop.run_in_executor(None, r.read, 1)
        return d

    async def run_async():
        futures = [
            asyncio.ensure_future(read(t))
            for t in tiffs
        ]
        result = await asyncio.gather(*futures)

    loop = asyncio.get_event_loop()
    loop.run_until_complete(run_async())
    loop.close()


s = time()
tiff_use_sync()
e = time()
print(f'동기 방식 소요 시간: {e-s}초')

s = time()
use_async_tiff()
e = time()
print(f'비동기 방식 소요 시간: {e-s}초')
```

아래는 한 개에서 다섯 개까지의 래스터를 동기 방식과 비동기 방식으로 읽어들이면서 속도를 측정한 결과입니다.

![래스터 개수 변경에 따른 속도 변화 비교]({{ "/assets/img/posts/2018/10/14/asyncio-raster-read-1.png" | absolute_url }})

비동기 방식이 대량 2.5배 정도 빠릅니다. CPU 사용률을 측정하지는 않았지만 눈 대중으로 살펴본 결과
싱글코어를 이용하는 동기 방식과 달리 비동기 방식은 멀티코어를 이용하는 것으로 보였습니다. (이것과 관련된 측정은
차후 제시해보도록 하겠습니다.) 비동기 방식의 읽기는 속도 향상 효과가 크지만 동기 방식 대비 프로그램 코드가 보다 
복잡합니다. 효율적인 커스터마이징(타일로 나누어 읽기, 분산 처리 등)이라도 하려면 async에 대한 이해가 필요합니다. 
우려되는 점은 프로그램을 대충 작성해도 잘 실행된다는 점입니다. 
비동기 효과가 없는 채로 말입니다. 그 결과 보게 되면 비동기 방식이 동기 방식 대비 
속도 이득이 없다는 오해를 하게 될 수 있습니다. 
어떻게 하면 이것이 잘 작성된 비동기 코드인지 알 수 있을까요? 
아직 답을 찾지 못했습니다.

[gdal-geotiff]: https://www.gdal.org/frmt_gtiff.html 
[asyncio]: https://docs.python.org/3/library/asyncio.html
[deflate]: https://en.wikipedia.org/wiki/DEFLATE
