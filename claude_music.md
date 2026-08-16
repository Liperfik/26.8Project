# Spotify Audio Feature 音频分析任务 — 复用Prompt

请在新对话开头粘贴以下全部内容，然后上传mp3文件。

---

## 任务
我会上传mp3文件。请分析音频，输出仿Spotify Audio Features API格式的一行数据：
`artist, song, duration_ms, danceability, energy, key, loudness, mode, speechiness, acousticness, instrumentalness, valence, tempo, year, genre`

**环境限制（已验证）**：这个环境**没有网络访问给bash_tool**（pip install会失败），librosa/essentia/aubio都装不上。只能用 **numpy + scipy + ffmpeg(系统自带)** 手写信号处理。不要浪费time尝试pip install音频库，直接手写。

## 标准流程（已验证有效，按此执行）

### Step 0: 解码
```bash
ffmpeg -y -i input.mp3 -ac 1 -ar 22050 -acodec pcm_s16le audio_mono.wav
```
用 `ffprobe -show_format` 先看有没有内嵌tags（artist/title有时藏在metadata里）。

### Step 1: STFT基础
n_fft=2048, hop=512, hanning窗，手写rfft循环（scipy没有现成stft就手撸）。

### Step 2: tempo — 自相关法 + **务必做倍频校验**
用onset能量包络（帧间spectral magnitude差分，clip负值）做自相关，在设定BPM范围内取峰值lag转tempo。
**关键教训**：这个方法极易抓到半拍或双倍拍（octave error），三次测试里出现了3次倍频错误。**必须**：
1. 算出来的tempo如果 >160 或 <70，本能地怀疑是倍频/半频
2. 有条件的话上网搜"歌名 BPM"做交叉验证，用查到的真实BPM校正（三次验证下来，网上查的BPM数据库比自己算的更可靠）
3 没法验证时，同时给出raw值和raw÷2、raw×2作为候选，選择更接近人声风格常见范围(85-140大部分歌，说唱/舞曲可到160+)的那个

### Step 3: loudness — 近似ITU-R BS.1770 K加权积分响度
不要用简单RMS转dB（早期版本这么干误差有+4dB这么大）。做法：
1. 高频架式滤波(+4dB shelf, f0≈1650Hz, Q=0.707, RBJ cookbook公式)
2. 二阶巴特沃斯高通(~38Hz)模拟RLB加权
3. 400ms窗口/100ms跳步分块算均方值
4. 两级门限（绝对门限-70LUFS，相对门限=未门限响度-10LU）
5. 积分响度 = -0.691 + 10*log10(门限后均值)
6. **经验校准偏移量：+6.5dB**（因为是mono downmix+近似滤波，不是真正stereo BS.1770，实测三首歌加上这个offset误差能压到±1dB内）

### Step 4: key/mode — chroma + Krumhansl-Schmuckler模板
**关键教训**：不要用能量加权平均chroma（人声/响亮乐器会主导，判断错误率高）。改用：
1. 每帧chroma先归一化（除以该帧总能量）
2. 取所有帧的**中位数**而不是均值，得到更鲁棒的chroma向量
3. 和大调/小调Krumhansl模板做12次滚动相关，取最高分
即便这样，3首歌里对了2首，错了1首（有人声的复杂编曲仍会干扰）——如实告知用户这项准确率有限。

### Step 5: energy — loudness驱动为主
`energy ≈ 0.65 * clip((loudness_est+60)/60, 0,1) + 0.35 * 频谱质心亮度分量`
（实测：energy和loudness相关性远比和RMS高低起伏的相关性强，这是最有效的单项改进）

### Step 6: acousticness — 频谱平坦度驱动
`acousticness ≈ 1 - (0.5*flatness_norm + 0.3*brightness + 0.2*energy)`
flatness高（失真/噪声化）→acousticness低；这个方向不要搞反（第一版搞反过）。

### Step 7: danceability / valence — 节拍规律性+调式+tempo+亮度加权
```
danceability ≈ 0.4*beat_regularity + 0.3*tempo接近120的程度 + 0.3*energy
valence ≈ 0.35*(大调=0.6/小调=0.3) + 0.35*tempo_score + 0.30*brightness
```
**如实告知**：这两项系统性容易被低估（尤其valence），这是DSP heuristic的已知短板，不是每次都能修正。

### Step 8: speechiness / instrumentalness — 人声频段调制谱分析
用300-3400Hz频段能量占比的包络，做FFT看3-8Hz（人声音节速率）vs 8-16Hz（乐器/节奏速率）的能量比例。
**重要教训**：这个"真实算法"在纯演唱的流行/摇滚歌曲上，测试结果反而**比直接给0.05常数更差**（会把鼓点/吉他节奏误判成人声调制）。但对**说唱类歌曲**，这个算法给出的偏高speechiness（~0.13-0.15）大概率是对的，因为说唱本身语速快、调制信号更强。
**建议**：如果不确定歌曲类型，speechiness默认给低值(~0.05)+用这个算法做小幅上调修正，而不是完全相信算法原始输出；如果明确是说唱/hip-hop，可以更多相信算法输出。

### Step 9: genre / year — **不要用kNN瞎猜，直接网上搜**
之前尝试过用CSV数据集(songs_normalize.csv)做kNN检索来猜genre/year，**两次测试两次都判断错误**（音频特征空间信息量不足以区分相近genre，比如"pop" vs "rock, pop" vs "Dance/Electronic"）。
**现在的标准做法**：直接web_search"歌名 歌手 发行年份 专辑"，从权威信源（官方专辑资料/维基百科/豆瓣）拿到真实year和genre，比任何音频算法都准。只有搜不到的冷门/小众曲目才退回到音频推测，并明确告知用户这是低置信度猜测。

### Step 10（可选）: kNN局部先验，仅用于continuous特征的"锚定"
如果有songs_normalize.csv（或类似真值数据集）可用，可以对danceability/energy/valence/acousticness/speechiness/instrumentalness这几项做一次kNN检索（标准化欧氏距离，k=15），把raw DSP值和k近邻真实均值做加权融合（权重经验值：tempo 0.9, loudness 0.75, energy 0.55, danceability/valence/acousticness/speechiness/instrumentalness 0.35-0.45），能小幅降低方差。**但如果分析对象是数据集里没有类似曲风的歌（比如华语说唱 vs 欧美流行摇滚库），这个先验会失真，应该调低权重或干脆不用。**

## 输出要求
1. 先给raw DSP值
2. 再给任何用到的先验/查证结果
3. 最终融合值
4. **诚实标注每项的置信度**：tempo(校验后)/key/mode/loudness/energy/genre/year(查证后) 可信度较高；danceability/valence/acousticness/speechiness/instrumentalness 属于中低置信度的heuristic近似，不要假装很准
5. 如果有真值可对照，务必做误差对比表，不要回避讲清楚哪里错了、为什么错

---
（这段prompt本身就是几轮实测调优后的结果，如果新对话里又发现新的系统性偏差，建议同样方式记录下来，持续迭代这份"方法论备忘"。）