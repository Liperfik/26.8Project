# 26.8Project

本次探学营我制作的项目是根据Spotify上的歌曲音乐数据，使用机器学习算法来猜测其是否能成为热门歌曲

数据库来源：[Kaggle数据库](https://www.kaggle.com/datasets/paradisejoy/top-hits-spotify-from-20002019)


项目制作：[Colab项目](https://colab.research.google.com/drive/1NGO_sYKdzCtFV5PrSF3LuNjwt1M3sesw?usp=sharing)，可能需要自己导入[songs_normalize.csv](https://github.com/Liperfik/26.8Project/blob/main/songs_normalize.csv)文件

如果想要预测本地音乐文件，可以在[spotify_popularity.ipynb](https://github.com/Liperfik/26.8Project/blob/main/spotify_popularity.ipynb)中运行第二个代码框（前提是先运行第一个框保存文件）并输入音频特征

在以前，音频特征是可以直接申请Spotify的API Key来获得的，可以点击[此链接](https://developer.spotify.com/dashboard)转至Spotify开发者平台

但是这项服务在24年停运了，所以可以选择使用大模型来给出评分（本来打算用机器学习训练模型给分的，但是时间确实过于紧张），我在仓库中上传了[claude_music.md](https://github.com/Liperfik/26.8Project/blob/main/claude_music.md)文件，其中是我写好的可在Claude中使用的关键词

Release中共有5个视频文件，它们是我写代码时录制的过程，如有需要可以观看
