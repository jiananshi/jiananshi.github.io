---
title: ServiceWorker + Cache 的前端缓存方案
date: 2017-02-06 23:53:08
tags: serviceworker cache javascript
---

通过 ServiceWorker 可以拦截前端的请求，而 Cache 可以根据请求将响应存储在前端，那么这两者结合起来我们就可以实现一个前端的缓存方案了，这里我们来讨论一下具体的实现和几种缓存策略。

## 网络请求优先策略
![strategy](http://oiw32lugp.qnssl.com/2017-02-06-%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-02-06%20%E4%B8%8B%E5%8D%8810.54.06.png)
（图片来源：https://huangxuan.me/2016/11/20/sw-101-gdgdf/，侵删）

这个方案优先使用网络请求，网络请求失败后再从缓存中提取数据，记得在网络请求后及时更新缓存。不好的地方显而易见，并没有大幅提高用户体验和速度。

```javascript
self.addEventListener('fetch', e => {
	e.respondWith(
    fetch(e.request.url)
		  .catch(e => {
         return caches.match(e.request)
				.then(res => res || e);
       })
  );
});
```

## 缓存优先策略
![strategy](http://oiw32lugp.qnssl.com/2017-02-06-%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-02-06%20%E4%B8%8B%E5%8D%8810.56.33.png)
（图片来源：https://huangxuan.me/2016/11/20/sw-101-gdgdf/，侵删）

这个策略优先使用缓存，缓存没有命中或失效才发送网络请求，这个方案会造成你的用户很难看到最新的响应，如果场景适合这个方案千万记得做好版本控制。

```javascript
self.addEventListener('fetch', e => {
	e.respondWith(
		caches.match(e.request.url)
		  .then(res => res || fetch(e.request))
	);
});
```

## 请求和缓存竞赛
![strategy](http://oiw32lugp.qnssl.com/2017-02-06-%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-02-06%20%E4%B8%8B%E5%8D%8811.00.59.png)
（图片来源：https://huangxuan.me/2016/11/20/sw-101-gdgdf/，侵删）

这个方案在性能上胜于其他方案，只是要额外多一个操作，并且结果是不可控的。

```javascript
self.addEventListener('fetch', e => {
	e.respondWith(
	  Promise.race([
      fetch(e.request.url),
      caches.match(e.request)
	  ])
	);
});
```

## 超时后重新请求策略
![strategy](http://oiw32lugp.qnssl.com/2017-02-06-%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-02-06%20%E4%B8%8B%E5%8D%8811.10.38.png)
（图片来源：https://huangxuan.me/2016/11/20/sw-101-gdgdf/，侵删）

```javascript
self.addEventListener('fetch', e => {
  let fetchData = fetch(`${ e.request.url }?${ Date.now() }`);
  // Request 是一个 Stream
  let _fetchData = fetchData.then(raw => raw.clone());

	e.respondWith(
	  caches.match(e.request)
	    .then(res => res || fetchData)
	);

  e.waitUntil(
    Promise.all([
      _fetchData,
      caches.open('test-1.0')
    ]).then(([res, cache]) => {
      if (res.ok) cache.put(e.reqeust, res);
    });
  );
});
```

## 最快的超时后重新请求策略
其实就是结合竞速啦：

```javascript
self.addEventListener('fetch', e => {
  let fetchData = fetch(`${ e.request.url }?${ Date.now() }`);
  // Request 是一个 Stream
  let _fetchData = fetchData.then(raw => raw.clone());

	e.respondWith(
    Promise.race([fetchData, caches.match(e.request)])
	    .then(res => res || fetchData)
	);

  e.waitUntil(
    Promise.all([
      _fetchData,
      caches.open('test-1.0')
    ]).then(([res, cache]) => {
      if (res.ok) cache.put(e.reqeust, res);
    });
  );
});
```
