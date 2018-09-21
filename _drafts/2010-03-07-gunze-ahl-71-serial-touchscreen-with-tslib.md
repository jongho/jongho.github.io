---
title: "GUNZE AHL-71 Serial TouchPanel with tslib"
date: 2010-03-07 20:40:00 +0900
categories: gunze ahl71 serial touchpanel tslib
---
GUNZE AHL-71 Serial TouchPanel with tslib
==========================================

### 1. 개요

#### tslib
tslib는 흔히 임베디드 리눅스에서 qt를 사용할 때, 터치스크린을 이용하기 위해 사용되는 중간 드라이버 레이어(?)이다.
- http://www.tslib.org/
- https://github.com/kergoth/tslib

#### GUNZE AHL-71 Serial TouchPanel
터치스크린과 보드 사이의 거리가 멀면, 전송이 어려워진다. 이러한 상황에서 AHL-71과 같은 컨트롤러를 사용하여 시리얼 방식으로 전송한다.
- http://www.gunze.co.jp/denzai/en/download/index.html
- https://www.gunzeusa.com/products/controllers-and-drivers/controllers-and-drivers-analog-resistive/
- https://www.gunzeusa.com/wp-content/uploads/AHL-71N.pdf
 

### 2. 소스

#### gunze71-raw.c (tslib/plugins)
```c
#include <stdlib.h>
#include <stdio.h>
#include <unistd.h>

#include "config.h"
#include "tslib-private.h"

/* 
 FORMAT: T 0123 , 4567 <LF>
 INDEX:  0 1234 5 6789 0
*/

#define TS_PACKET_SIZE 11

static int gunze71_read(struct tslib_module_info *inf, struct ts_sample *samp, int nr)
{
	struct tsdev *ts = inf->dev;
	char *data;
	int ret;
	int total = 0;
	int i;

	data = alloca(TS_PACKET_SIZE * (nr + 1));
	ret = read(ts->fd, data, TS_PACKET_SIZE * (nr + 1));

	for(i = 0; i < TS_PACKET_SIZE; i++) {
		if(data[0] == 'T' || data[0] == 'R')
			break;
		data++;
		ret--;
	}

	if(ret > 0) {
		i = 0;
		while(ret >= TS_PACKET_SIZE && i < nr) {
			data[5] = '\0';
			data[10] = '\0';

			samp->x = (short)atoi(&data[1]);
			samp->y = (short)atoi(&data[6]);

			if(data[0]=='T')
				samp->pressure = 1;
			else if(data[0] == 'R')
				samp->pressure = 0;
			else
				return -1;

#ifdef DEBUG
        fprintf(stderr,"RAW---------------------------> %d %d %d\n",samp->x,samp->y,samp->pressure);
#endif /*DEBUG*/
			gettimeofday(&samp->tv,NULL);
			samp++;
			data += TS_PACKET_SIZE;
			ret -= TS_PACKET_SIZE;
			i++;
		}
	} else {
		return -1;
	}

	ret = nr;
	return ret;
}

static const struct tslib_ops gunze71_ops =
{
	.read	= gunze71_read,
};

TSAPI struct tslib_module_info *mod_init(struct tsdev *dev, const char *params)
{
	struct tslib_module_info *m;

	m = malloc(sizeof(struct tslib_module_info));
	if (m == NULL)
		return NULL;

	m->ops = &gunze71_ops;
	return m;
}
```

#### 다운로드
- [tslib-rev77-gunze71.tar.gz](http://zwolf.org/attachment/cfile27.uf@155FDB104B8E15C34AE1FE.gz)
