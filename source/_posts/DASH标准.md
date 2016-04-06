title: DASH标准
comments: true
date: 2015-06-06 11:28:31
tags:
keywords:
description:
thumbnailImage:
coverImage:
photos:
---
#DASH标准

##功能

***Multiple Audio Channels***: 支持有多个语言的音频切换

***Closed Captions/Subtitles***: 使用[webvtt](http://dev.w3.org/html5/webvtt/)来描述字幕。

***Efficient AD Insertion***

***Fast Channel Switching***: DASH的chunk为2-4秒，小的chunk可以方便做到不同频道间的快速切换，减少响应时间，而且有利于码率自适应的调整。但这个是以牺牲Video编码的效率(大的chunk可以有更高的压缩比)和网络协议的开销为代价的。

***Support Multiple CDNs In Parallel***

DASH可以通过BaseURLs来告诉客户端可用的CDN，客户端可以自行选择。

***HTML5支持***

支持HTML5的[Media Source Extensions](http://w3c.github.io/media-source/)的浏览器可以直接播放。

***HbbTV支持***

***Agnostic to Audio/Video Codecs***

支持不同的Audio/Video编码

***ISO Base Media File Format Segments***

***MPEG-2 TS Segments***

***WebM Segments***

***HEVC and 4K***

***Multiplexed Audio/Video Content***

***Quality of Metrics***

***Client Logging & Reporting***

***Client Failover***

***Remove and add Quality Levels during Streaming***

***Multiple Video Views***

***Efficient Trick Modes***

##参考

[MPEG-DASH vs. Apple HLS vs. Microsoft Smooth Streaming vs. Adobe HDS]<http://www.bitcodin.com/blog/2015/03/mpeg-dash-vs-apple-hls-vs-microsoft-smooth-streaming-vs-adobe-hds/>

[What is MPEG DASH?]<http://www.streamingmedia.com/Articles/Editorial/What-Is-.../What-is-MPEG-DASH-79041.aspx>

[DASH Industry Forum]<http://dashif.org/>