---
title: web audio api 前端音效处理
date: 2017-02-29 20:46:27
categories: "web audio api"
tags:
    - web audio
    - audioContext
    - 前端
    - 音效
    - 音频
    - 音频可视化
    - 淡入淡出
    - 人声
    - 空间环绕声
    - 低音增强
    - 滤波
---
## 简介
本文主要列举几种常见的音效实现在web平台下的实现，项目实现的源代码已上传 [github](https://github.com/Zenaro/web-audio-effect)，喜欢的话就点个star吧，也可点击 [项目地址](http://zenaro.cn/Music-Effect/index.html) 进行测试（chrome体验最佳）。

## Audio API基本概念
Audio API 提供了控制音频的一个依据，它量化了音频在web系统上的流通轨迹，使得开发者可以通过 Audio Context对象来指定多种音频的流通渠道，并且在流通的过程中实现丰富的音频处理。
在实现上，Audio API将音频路径量化为一个个相连的节点，音频节点通过它们的输入输出相连，形成了一道道可控的音频通路，并最终汇集在同一个目的地destination（通常是扬声器或耳机等播放器）。一个典型的audio音效工作流如下所示：

![web audio api](http://7xstax.com1.z0.glb.clouddn.com/web-audio-api.png)

以上音频的实现步骤可分为：
1.创建音频环境上下文（Audio Context）；
2.在音频环境中创建音源（source），该对象可以为振荡器、Audio标签对象等等；
3.创建音效节点，如混响、双二阶滤波器、平移、压缩；
4.为音频选择一个终点（destination），如系统扬声器；
5.连接音源到音效节点，以及连接音效节点到终点，以构成音频播放的通路。
通过图示，我们可以将音频处理分解为三个节点的相连：源节点、分析和处理节点、目的节点。在中间的处理节点上，可以通过底层API实现音效，如使用BiquadFilterNode调整音色（即滤波器）、使用ChannelSplitterNode分割左右声道、使用GainNode调整增益值实现音乐的淡入淡出。

<!-- more -->

## Audio API 实现
首先创建全局audio对象、Audio环境对象、分析器Analyser、gainNode节点，随后创建一条播放的通路
``` bash
let audio = new Audio();
let audioCtx = new(window.AudioContext || window.webkitAudioContext)();
let source = audioCtx.createMediaElementSource(audio); // 全局的音频源
let analyser = audioCtx.createAnalyser(); // 全局的音频分析器
let gainNode = audioCtx.createGain(); // 全局的gainNode，主要为实现淡入淡出效果
audio.src = '***.mp3';
analyser.fftSize = 2048;

// source -> analyser -> gainNode -> destination  创建播放的一条通路
source.connect(analyser);
analyser.connect(gainNode);
gainNode.connect(audioCtx.destination);
```

## 淡入淡出声音
在切换歌曲或切换播放/暂停状态时，使用淡入淡出音量的效果可以增强体验，有了web audio api后，可以更精确地模拟音量增益的平滑衰减和平滑增加过程
![web audio api](http://7xstax.com1.z0.glb.clouddn.com/gainNode.png)
``` bash
// 淡入
function layin() {
    gainNode.gain.value = 0; // 音量从0开始
    audio.play(); 
    let currentTime = audioCtx.currentTime;

    // 0.7 为衰减总时间，单位为秒
    gainNode.gain.linearRampToValueAtTime(1, currentTime + 0.7); 
}

// 淡出
function layout() { // 音量从1开始
    let currentTime = audioCtx.currentTime;
    gainNode.gain.linearRampToValueAtTime(0, currentTime + 0.7);
    setTimeout(() => {
        audio.pause();
    }, 700); // 700ms后再暂停
}
```

## 音频可视化
使用全局的analyser来对音频进行分析，analyser可以计算快速傅里叶变换后的数据，这里取其中的频域数据，结合canvas绘制音频的频域柱状图：
``` bash
let canvas = document.getElementById('myCanvas');
let canvasCtx = canvas.getContext('2d');
let freqByteData = new Uint8Array(analyser.frequencyBinCount); // 频域数据
let [WIDTH, HEIGHT, bufferlength] = [300, 500, 1024];
function drawRect () {
    requestAnimationFrame(drawRect);
    analyser.getByteFrequencyData(freqByteData);
    canvasCtx.clearRect(0, 0, WIDTH, HEIGHT);
    for (let i = 0, cellW = 3, length = bufferLength; i < length; i += cellW) {
        // requestAnimationFrame(() => {
        let y = freqByteData[length - i] / 4;
        y = y > 1 ? y : 1;
        canvasCtx.fillStyle = 'rgb(255, 255, 255)';
        canvasCtx.fillRect(i / cellW * (cellW + 1), HEIGHT - y, cellW, y);
        // });
    }
}

```

## 封装公共的断开音轨函数:disconnect
实现特定音效前均需要断开原先的音频轨道，重新创建一条通路
``` bash
function disconnect() { // 断开轨道
    // 断开analyser和gainNode的连接，重新确定路径
    gainNode.disconnect(0);
    analyser.disconnect(0);
    
}
```

## 空间环绕声
在web audio api下可以通过pannerNode来模拟实现立体声，其本质是调整左右声道的音频衰减和增益以模拟声音的方向性。
``` bash
disconnect(); // 断开轨道
let [panner, gain1, gain2, convolver] = [audioCtx.createPanner(), audioCtx.createGain(), audioCtx.createGain(), audioCtx.createConvolver()];

panner.setOrientation(0, 0, 0, 0, 1, 0);
let [index, radius] = [0, 20];
// 声源将绕着收听者以单位20的半径旋转
let effectTimer = setInterval(() => {
    panner.setPosition(Math.sin(index) * radius, 0, Math.cos(index) * radius);
    index += 1 / 100;
}, 16);
source.connect(panner);
gain1.gain.value = 5;
panner.connect(gain1);
gain1.connect(audioCtx.destination);

convolver.connect(gain2);
gain2.connect(audioCtx.destination);
```

## 低通滤波
在web下的低通滤波，是将音频中高于某Hz的音频过滤掉，以实现音乐的低音增强音效，其他带通滤波同理

``` bash
function lowpassFilter(freq) {
    disconnect();
    let biquadFilter = audioCtx.createBiquadFilter();
    biquadFilter.type = 'lowpass'; // 低阶通过
    biquadFilter.Q.value = 2;
    biquadFilter.frequency.value = freq || 800; // 临界点的 Hz，默认800Hz
    source.connect(biquadFilter);
    biquadFilter.connect(audioCtx.destination);
}
```

## 礼堂回声
在原有的音频上，为其增加一个0.06s、增益为1.2的延时音频，来实现礼堂回声效果，贴出简单的示例如下：
``` bash
disconnect();

let [delay, gain] = [audioCtx.createDelay(), audioCtx.createGain()];

delay.delayTime.value = 0.06; // 延时0.06s
gain.gain.value = 1.2;

// 两条平行通路

// 1. source -> gainNode -> destination
gainNode.connect(audioCtx.destination);

// 2. source -> gainNode -> delay -> gain -> destination
gainNode.connect(delay);
delay.connect(gain);
gain.connect(analyser);
analyser.connect(audioCtx.destination);
```

## 人声消除
音频可分为人声和伴奏，人声消除即为去除人声保留伴奏。目前人声消除主要分为两种，一种是根据人声的分布范围直接对该范围内的音频进行减益。（一般而言，歌唱者发出的声音频率范围为：男低音82～392Hz，基准音区64～523Hz，男中音123～493Hz，男高音164～698Hz；女低音82～392Hz，基准音区160～1200Hz，女中音123～493Hz，女高音220～1.1KHz），使用带通滤波完成该效果：
``` bash
disconnect();
let biquadFilter = audioCtx.createBiquadFilter();
biquadFilter.type = 'lowshelf'; // 低于该频率将获得衰减
biquadFilter.gain.value = -100;

//临界点的Hz，此处取男中音临界点493Hz
biquadFilter.frequency.value = 493; 

gainNode.connect(biquadFilter);
biquadFilter.connect(analyser);
analyser.connect(audioCtx.destination);
```
以上消除人声的方法消除效果一般（容易把伴奏也消除了），另外一种消除人声效果较佳的方式是采取两个声道相减的办法来消除立体声歌曲中的人声，人声的声波波形在歌曲的两个声道是相同或者相似的，将左右声道互减将得以除去音频公共的部分（即人声），且伴奏得以保留。因此可以先将音频分离出左右声道，左声道叠加一个右声道-100%的反向增益，右声道亦叠加左声道的-100%来实现互减，最后再将左右合成双声道。简单点说，就是先分离左右声道，然后互减，最后合成。（注意到在使用这种方式消除人声的时候也会消除音乐的低音部分，因此，后期还需要对低音进行补偿。此处代码仅实现三步走，尚未进行低音补偿）。
``` bash
disconnect();
let gain1 = audioCtx.createGain(),
    gain2 = audioCtx.createGain(),
    gain3 = audioCtx.createGain(),
    channelSplitter = audioCtx.createChannelSplitter(2),
    channelMerger = audioCtx.createChannelMerger(2);

// 反相音频组合
gain1.gain.value = -1;
gain2.gain.value = -1;

gainNode.connect(analyser);
analyser.connect(channelSplitter);

// 交叉音轨，减去相同的音频部分（即人声）
channelSplitter.connect(gain1, 0);
gain1.connect(channelMerger, 0, 1);
channelSplitter.connect(channelMerger, 1, 1);

channelSplitter.connect(gain2, 1);
gain2.connect(channelMerger, 0, 0);
channelSplitter.connect(channelMerger, 0, 0);

// 最后合成左右声道
channelMerger.connect(gain3);
gain3.connect(audioCtx.destination);
```
