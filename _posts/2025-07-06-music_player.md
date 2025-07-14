---
layout: post
title: 自制云音乐播放器
---

无语了，网易云音乐会员7月一过期没注意，结果把我下载的歌都搞没了，于是不打算续了，但是要听歌怎么办，于是顺手抄起 `AI` 写了一个 `html` 的音乐播放器，然后配合我的 `proxy` 提供的 `http` 服务，通过 `proxy` 的 `http` 返回文件列表 `json` 接口功能，就地搭了一个云音乐播放器，嘿嘿，支持显示歌词呢，还挺有意思的

[播放器示例](https://www.jackarain.org/music/index.html)

![image](https://github.com/user-attachments/assets/3910015e-f4d6-4162-912b-8b7594c41d88)

跑手机上也不错呢，我把这个 html 文件也上床到 github 上了 [music_player](https://github.com/Jackarain/proxy/tree/master/example/music_player) 分享给大家，有兴趣的朋友也可以玩一下。
